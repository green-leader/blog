+++
title = "Image Change"
publishDate = "2023-07-15T08:00:00Z"
tags = ["unraid", "docker", "system administration"]
showFullContent = false
readingTime = false
hideComments = false
+++
While changing what source image a docker container in unRaid was using, the Docker managment service encountered an error. The error itself I forgot to write down. But what is normally a typical process of "download new image, stop old container. create new container, cleanup old image." was interrupted and it left the container in a down state. This wasn't good primarily because I am lazy and didn't want to spend the mental power and try to come up with what the previous config options were so there can be no change except what image is being used.

After fixing the underlying cause for what caused the error. (The ghcr package for this particular repo was marked as private only). I started trying to recover what was a downed service. A brief spat of web searches and I discover that when creating a custom docker image 2 things are true. The first that it's stored as an xml template in `/boot/config/plugins/dockerMan/templates-user`. The second that I didn't have to open the XML file and try to transcode as I had thought. You can simply go to `Docker` -> `Add Container` -> `Templates` and all of your custom docker invocations are contained there as templates. Easy Peasy.