+++
title = "Full Hd"
publishDate = "2023-03-20T08:00:00Z"
tags = ["", ""]
showFullContent = false
readingTime = false
hideComments = false
+++
I woke up this morning to my RSS feed misbehaving. FreshRSS was popping up an error along the lines that it was unable to make an internet connection. This was a little odd as I was connected over the local network to the app. It was loading at all which meant it could connect. I won't bore you with how I found the problem, but the root partition of the server was full.

A full root partition can cause a wide array of problems. As there shouldn't be anything downloading to this partition it was well over what it should be. The root partition is located on a 112 GB SSD. so there's not a whole lot but after cleaning up space I found that I was using less than a quarter of the drive.

When the drive is full you need to prioritze geting any space. just so the system can start to breathe again. 

As this is a docker host `docker system prune` is a great candidate. It only cleared off a few hundred megs but it did what it was supposed to in this instance. Now we can search down and track down what the issue is.

`sudo du -hsx /* | sort -rh | head -n 5`
Couple things going on so lets break down the arguments
`du -hsx`:
- `h`: Human readable output. As a lowly human just reading bytes is hard.
- `s`: summarize, just put the totals on everything that du is ran against.
- `x`: one-file-system. Don't leave the filesystem, as we are only looking at the root partiition we don't need to know about everything that's mounted within the `/` filesystem.

`sort -rh`: 
- `r`: Reverse. We want the big numbers first
- `h`: human-numeric-sort. We're using human numbers because we're human.

`head -n 10`:
- `n`: Pretty simple, this is how many lines we want. By specifying a number we're not just using the default.

And the output:
```bash
$ sudo du -hsx /* | sort -rh | head -n 5
86G     /var                               
3.2G    /home                   
2.0G    /usr
81M     /boot                               
3.9M    /etc
```

Well it's `/var` as a culprit. We can just run the same command but specifying the path so we can dig in and keep digging down to find where the issue is. I won't post the outputs but I'll give the commands:
1. `$ sudo du -hsx /var/* | sort -rh | head -n 5`
2. `$ sudo du -hsx /var/lib/* | sort -rh | head -n 5`
3. `$ sudo du -hsx /var/lib/docker/* | sort -rh | head -n 5`
4. `$ sudo du -hsx /var/lib/docker/containers/* | sort -rh | head -n 5`

And the output for that last command points where the culprit is, mainly this nifty line:`64G     /var/lib/docker/containers/containers/c585fa6366ddbd90e8565e07797431deab5c86bd157f757317d3b0655c099562`

Looking a bit further (I forgot to grab logs of my commands) we can see that the container log(s) is a single 63 gig file. Whelp. There's the problem. A single docker container log is eating up 64 gigs of data, probaly bad configuration on my part. Using `docker inspect` we can find out which container was badly configured. Ended up being my borgmatic container, which has been running daily for over a month. Great! A log file is an easy fix! We just need to implement log rotation.

Couple different ways we can do it.
- logrotate daemon
- docker daemon log-driver settings

The docker daemon is probably a little easier to configure, but I'm also just more comfortable with it. Setting defaults on the docker stuff is primarily done via the config file which on a linux system is at `/etc/docker/daemon.json`. This file may need to be created, and we'll do so putting the below into the file. Updating this file and restarting the service via `sudo systemctl restart docker.service` will cause any new container to use these new logging driver settings. As this isn't a new container we'll just use `docker-compose down` followed by `docker-compose up -d` on the compose file with the culprit container.

```json
{
  "log-driver": "local",
  "log-opts": {
    "max-size": "15m",
    "max-file": "5"
  }
}
```

After updating these settings we can see that was the issue `df -h` is now reporting that the root partition is at 27G which is much much better. Should figure out how to send logs to splunk. Though I'm not sure I like using splunk. We should also set up monitoring to get a notification when we get close to filling the drive again.