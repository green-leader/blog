+++
title = "Promtail Plex"
publishDate = "2023-12-11T08:00:00Z"
tags = ["System Administration", "Logging", "Promtail", "Loki"]
showFullContent = false
readingTime = false
hideComments = false
+++

The other day while watching Plex it would reliably stop playback 2 minutes before the end. When it did this an error would pop up "Conversion failed. The transcoder exited due to an error." Helpful.

Not a problem. We'll log onto the log server and see what it spat out. Well much to my dismay, Plex doesn't seem to log to the typical I/O streams. While I'm not overly surprised it definitely was something I didn't think about.

Made sure the actual log files were being exported as intended, and then started the trial and error of getting Promtail to appropriately label the logs and ship them to the Loki server. Here's what I ended up with putting in my scrape config:

```yaml
# Scrape plex logs
# *{r,e,s} is a little hard coded but it allows for ignoring the rotated logs
- job_name: plexlogs
  pipeline_stages:
    - regex:
        expression: '(?P<timestamp>\w{3}\s{1,2}\d{1,2}, \d{4} \d{2}:\d{2}:\d{2}.\d{3}) \[(?P<pid>.*)\] (?P<level>[A-Z]{4,5}) - (?P<message>.*)'
    - labels:
        level:
  static_configs:
    - targets:
        - localhost
      labels:
        host: storage01
        job: plexlogs
        __path__: /var/log_plex/*{r,e,s}.log
```
