+++
title = "Publishing Obsidian Documentation"
date = "2023-02-27T05:33:12Z"
tags = ["homelab", "documentation", "docker"]
showFullContent = false
readingTime = false
hideComments = false
+++


I keep my notes in markdown files in a git repo my primary editor is a tool called [obsidian.md](https://obsidian.md/). I've got minor gripes and for the most part I've got it syncing well and working well. However, a nice to have feature would be to share individual documents with the public. Couple examples, keeping a digital recipe box for the household to read from or sharing TTRPG notes after a session.

For right now I want to set it up such that I can share things with the household. Log into the designated subdomain, and so long as you're on my network using my DNS server it should load happily.

Here's the process I've taken and settled on for now.

Workflow:
1. sync git repo
2. process markdown into HTML
3. serve HTML with a web server


Tools:
1. Githubs cli tool [github.com/cli/cli](https://github.com/cli/cli)
2. [obsidian-to-hugo](https://github.com/devidw/obsidian-to-hugo)
3. [hugp](https://gohugo.io/)
4. nginx running on docker

### Workflow Step 1:

Something I've been trying to deal with and come up with a solution I like is giving my server read-only access to multiple private repos. As of the time of this writing I keep most of my git repositories located at github. Github doesn't allow you to specify an ssh key as a deploy key to multiple projects. And the deploy keys work great if you only need to grant access to 1 repo. But this runs into problems when you're trying to grant access to several, purely from a management standpoint. how you do you cleanly associate an ssh key with individual repos then refer to them. I saw some hacks online but I hadn't liked any of them.

Doing a bit of experimenting on this when I was trying to deploy this I remembered the Github CLI tool. After some trial and error I found that you can not use a personal access token however neither fine-grained tokens or classic tokens seem to have access to just perform read-only actions on the content of repos. I ended up using http authentication, logging in as myself. While writing this I'm trying to think if it's got read-only access or full read-write. I'll have to investigate it later.

### Workflow Step 2:

Processing markdown files into HTML is a joint problem. There's a number of static site generators that will take in markdown and output HTML. The tool i'm most familiar with is hugo, which is what this blog is using. Doing a bit of research I found a python module that converts obsidian markdown files into hugo markdown files.

looking at the readme it also provides functionality on scanning the contents of the file for specific conditions as to whether it should graduate into a published document. I've got a small python script written below which I run before I have hugo process the documents into HTML.

process.py:
```python
from obsidian_to_hugo import ObsidianToHugo

def filter_file(file_contents: str, file_path: str) -> bool:
    if "publish: true\n---" in file_contents:
        return True
    else:
        return False
    if "template_" in file_path:
        return False

obsidian_to_hugo = ObsidianToHugo(
    obsidian_vault_dir="/local/kb-obsidian",
    hugo_content_dir="/local/hugo-site/content",
    filters=[filter_file],
)

obsidian_to_hugo.run()
```

This is a pretty strightforward snippet on the python projects readme I've adjusted a bit for my needs.

After generating the content with that processing script we now need to trigger hugo to run and convert the content markdown files into HTML. Creating and configuring the initial hugo site only needs to be done the first time, but after that you can just change to the hugo site directory, then execute a simple `hugo`. Depending on which theme you're using it could require the extended version of hugo. Below I've included my config file.

When taking notes I will add page links to pages that don't exist. This can cause hugo to throw a fit and refuse to process then the files.
`REF_NOT_FOUND: Ref "Azure": <snip>: page not found`

Adding `refLinksErrorLevel = "WARNING"` to the config file results in warnings still being emitted but will tell hugo that it's alright.

hugo config.toml
```toml
baseURL = '/' 
languageCode = 'en-us'
title = 'My New Hugo Site'
theme = 'ananke'
refLinksErrorLevel = "WARNING"
```

### Workflow Step 3:

This is for the most part the simplest step, couple bullet points:

- Create a custom nginx config
- Point nginx at the hugo sites public directory
- Launch nginx container

I like to set the volume mounting and launch the container to create the directory structure. Replacing `nginx/nginx/site-confs/default.conf` with our custom config. 

nginx config file
```
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    server_name _;

    set $root /app/www/public;
    if (!-d /app/www/public) {
        set $root /config/www;
    }
    root $root;
    index index.HTML;

    location / {
        try_files $uri $uri/ /index.HTML /index.php$is_args$args =404;
        add_header 'Access-Control-Allow-Origin' '*'; # Allow access to resources (for www and non www)
    }
}
```

docker-compose.yml
```yaml
---
version: "2.1"
services:
  nginx-docs:
    image: lscr.io/linuxserver/nginx:latest
    env_file: .env
    volumes:
      - ./nginx:/config
      - ./hugo-site/public:/config/www
    ports:
      - 3003:80
    restart: unless-stopped
    networks:
      - mgmt_network
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.docs-http.entrypoints=web"
      - "traefik.http.routers.docs-http.rule=Host(`docs.${DOMAIN}`)"
      - "traefik.http.routers.docs-http.middlewares=docs-https"
      - "traefik.http.middlewares.docs-https.redirectscheme.scheme=https"
      - "traefik.http.routers.docs.entrypoints=websecure"
      - "traefik.http.routers.docs.rule=Host(`docs.${DOMAIN}`)"
      - "traefik.http.routers.docs.tls=true"
      - "traefik.http.routers.docs.tls.certresolver=myresolver"
      - "traefik.http.services.docs.loadbalancer.server.port=80"

networks:
  mgmt_network:
    external: true
```


Now when someone on the wifi goes the the subdomain docs on my internal domain name they get served the default hugo index page. There's no sitemap, but if things link to each other correctly it's plenty user friendly. While it's not pretty it's a minimum viable product and I plan to further iterate on it.
