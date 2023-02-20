+++
title = "Splunk Setup"
date = "2023-02-20T03:24:33Z"
tags = ["splunk", "system administration", "docker", "linux"]
keywords = ["splunk", "system administration", "docker", "linux"]
showFullContent = false
readingTime = false
+++
I set up a splunk docker container recently and there were a couple what feel like oddities catching me up.

1. Default debian doesn't have world readable log files.
2. This is not for production. But it's okay for my homelab.

Starting with this basic docker-compose file we made sure it worked.

```yaml
version: "3.6"

services:
  so1:
    image: ${SPLUNK_IMAGE:-splunk/splunk:latest}
    container_name: so1
    environment:
      - SPLUNK_START_ARGS=--accept-license
      - SPLUNK_PASSWORD
    ports:
      - 8000:8000
```

It's simple, gets everything running without doing anything fancy. Let's iterate it fancier and match the style of the rest of my compose files.

```yaml
version: "3.6"

services:
  splunk:
    image: splunk/splunk:latest
    container_name: splunk
    env_file: .env
    environment:
      - SPLUNK_START_ARGS=--accept-license
    volumes:
      - splunk_config:/opt/splunk/etc
      - splunk_data:/opt/splunk/var
      - /var/log:/host/var/log:ro
    ports:
      - 8000:8000
    restart: unless-stopped
    networks:
      - mgmt_network
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.splunk-http.entrypoints=web"
      - "traefik.http.routers.splunk-http.rule=Host(`splunk.${DOMAIN}`)"
      - "traefik.http.routers.splunk-http.middlewares=splunk-https"
      - "traefik.http.middlewares.splunk-https.redirectscheme.scheme=https"
      - "traefik.http.routers.splunk.entrypoints=websecure"
      - "traefik.http.routers.splunk.rule=Host(`splunk.${DOMAIN}`)"
      - "traefik.http.routers.splunk.tls=true"
      - "traefik.http.routers.secwhoami.tls.certresolver=myresolver"
      - "traefik.http.services.splunk.loadbalancer.server.port=8000"

volumes:
  splunk_config: {}
  splunk_data: {}

networks:
  mgmt_network:
    external: true
```

This is what we end up with after a couple iterations. But largely it matches the style and layout of the rest of my compose files. Three large changes:

1. I've moved the splunk password out of the compose file to a .env file (which gets encrypted for storing in git)
2. Added data persistence
3. Added Traefik labels, and network traefik watches

There was a slight problem once this was done, all of the files in `/var/log` aren't world readable. After a bit of searching we come across the splunk community forums detailing a similar issue [community.splunk.com](https://community.splunk.com/t5/Security/How-to-monitor-root-owned-logs-while-running-Splunk-as-a-non/td-p/16594) while this isn't a docker solution it's a solution to the problem. If not the most docker centric approach it beats setting up a universal forwarder to get files to the same machine.

Boiling it down we need to install acl controls, create a splunk user on the host, set up a logrotate rule file.

## Accomplishing 1 and 2:

```bash
sudo apt install acl
sudo groupadd -g 41812 splunk
sudo adduser --uid 41812 splunk --shell /sbin/nologin --system --no-create-home --gid 41812
```

Notice we created a group then created the user. The version of debian I was running with the version of adduser I had wasn't permitting to explicity specify that I needed a primary group matching the user. Making the user on its own was causing the user to get put in a group "nogroup". Probably user error.

## Accomplishing number 3:
```bash
cat /etc/logrotate.d/Splunk_ACLs
/var/log/splunklog
{
    postrotate
        /usr/bin/setfacl -m g:splunk:r /var/log/auth.log
        /usr/bin/setfacl -m g:splunk:r /var/log/messages
        /usr/bin/setfacl -m g:splunk:r /var/log/syslog
    endscript
}
```

Understanding this fully needs a bit of background.

### logrotate

As a broad stroke logrotate has a simple task. Keep files manageable.

logrotate performs this function through keeping track of the size of the file, or at regular intervals renames the file adding a `.#` onto the end of the filename. The tool is capable of much more but we don't want to delve too far into that.

The directory `/etc/logrotate.d/` is charged with holding different configurations for various packages. In this case we've both created a dummy file `/var/log/splunklog` and listed various rules on how logrotate should interact with that file. By default this file will be run daily as the root user.

### File ACLs

The keywords `postrotate` and `endscript` desginate a block of actions that should be perfomed when the file `Splunk_ACLs` is executed daily. This helps ensure that the specified actions are maintained on the specific files listed.

Lets break down the line(s) `/usr/bin/setfacl -m g:splunk:r /var/log/auth.log`

- `/usr/bin/setfacl` is a command with the sole purpose of setting linux file ACL rules.
- `-m` modify the rules on the designated file
- `g:splunk:r` pretty simple. This is where we specify new rules that should be active on the file. `:` is a delimiter splitting this into a three part spec.

that spec reads as follows:
- `g` says that the following piece will be either a gid or a group name.
- `splunk` the name of the group, we're opting for readability by using a group name.
- `r` the octal or relative form, if using relative any of `rwx` can be used, we'll be opting for read-only for logs in this case.