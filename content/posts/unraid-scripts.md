+++
title = "Unraid Scripts"
publishDate = "2023-07-05T08:00:00Z"
tags = ["System Administration", "Bash", "Scripting"]
showFullContent = false
readingTime = false
hideComments = false
+++
One of the services my unRaid box runs is a borgbackup server (which is secretly just an SSH server with a forced command). As a check to make sure the important repos are indeed getting backed up to I've got a script running daily via the _User Scripts_ Plugin. It's a pretty simple script, all it does is check when the files in certain target repos have last been written to. If it hasn't been in the last 7 days we send a Discord message for awareness.

Recently I was performing some maintenance and my script ran warning me that they all had not been written to within that cutoff. While I was investigating the logs it took me an embarrassingly long period of time to realize the unRaid array was simply not running. So, where the script was checking was missing. Time to add some safety to the script. Now the first section runs and the script will exit if the Array isn’t running.


Borg Check Script:
```bash
#!/bin/bash

# Check if array is started
ls /mnt/disk[1-9]* 1>/dev/null 2>/dev/null
if ! ls /mnt/disk[1-9]*
then
   echo "ERROR:  Array must be started before using this script"
   exit
fi

readonly WEBHOOK_URL="https://discord.com/api/webhooks/0000/XXXX"

# List of directories to check
readonly WATCH_DIRS=(
  "/mnt/dir1"
  "/mnt/dir2"
  "/mnt/dir3"
)

# Function to send a Discord webhook message
send_discord_message() {
  local message="$1"
  local payload="{\"content\": \"$message\"}"

  curl -H "Content-Type: application/json" -d "$payload" "$WEBHOOK_URL"
}

# Iterate over the directories
for dir in "${WATCH_DIRS[@]}"; do
  # Check if any files were modified in the last 7 days
  if find "$dir" -type f -newermt "-7 days" 2>/dev/null | grep -q .; then
    continue
  fi

  # Send Discord webhook message for directories without recent changes
  
  MESSAGE="Directory $dir has not been modified in the last 7 days"
  send_discord_message "$MESSAGE"
  echo "$MESSAGE"
done
```