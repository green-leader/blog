+++
title = "Docker Logs"
publishDate = "2023-12-04T08:00:00Z"
tags = ["Docker", "System Administration", "Today I Learned"]
showFullContent = false
readingTime = false
hideComments = false
+++

I was investigating what it would take to create a docker container and have the logs go to their appropriate log files in addition to being available via the `docker logs` command (don't remember why).
During the course of doing that I didn't accomplish my goal but I did learn that the typical I/O streams of STDOUT and STDERR are the defaults. So it's essentially just piping those output streams directly to the command.

Quick little post which was mostly just reading the documentation of the tool I'm using. Ha!

source data: [https://docs.docker.com/config/containers/logging/](docs.docker.com/config/containers/logging/)
