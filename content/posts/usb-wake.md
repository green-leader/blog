+++
title = "Usb Wake"
publishDate = "2023-07-16T08:00:00Z"
tags = ["System Administration", "Ansible", "Linux"]
showFullContent = false
readingTime = false
hideComments = false
+++
When done working on my desktop I suspend it, either through the Cinnamon panel or via `systemctl suspend`. But it had an issue. I would frequently come downstairs to find it on. The monitors with their familiar glow lighting up the room. My desktop running Linux Mint had a problem. Some initial diagnosis lead me to finding out that the issue was that the keyboard and mouse, paired with the lovely kitties were waking it from it's slumber.

After a bit of web searching I found a forum post https://forums.linuxmint.com/viewtopic.php?t=239133 that seems to have a similiar issue and a solution. I've rewritten a little bit here with examples, ending with an Ansible playbook to resolve the issue (at least on my machine). Updating it for your own equipment is an exercise left fot the user.

We run `cat/proc/acpi/wakeup | grep enabled` to get a list of devices that can wake the computer.

Then cross-reference it with `lspci | grep USB` to get the PCI bus for the USB devices.

```bash
sion@sion-dlnx:~$ lspci | grep USB
00:14.0 USB controller: Intel Corporation Cannon Lake PCH USB 3.1 xHCI Host Controller (rev 10)
01:00.2 USB controller: NVIDIA Corporation TU116 USB 3.1 Host Controller (rev a1)
01:00.3 Serial bus controller: NVIDIA Corporation TU116 USB Type-C UCSI Controller (rev a1)
sion@sion-dlnx:~$ cat /proc/acpi/wakeup | grep enabled
PEG0	  S4	*enabled   pci:0000:00:01.0
RP11	  S4	*enabled   pci:0000:00:1d.0
GLAN	  S4	*enabled   pci:0000:00:1f.6
XHC	  S4	*enabled   pci:0000:00:14.0
AWAC	  S4	*enabled   platform:ACPI000E:00
```
Looking at this, we just need to disable `XHC` to get our intended result of preventing USB from waking the computer.

The linked forum post calls out adding `echo "EHC3" > /proc/acpi/wakeup` to `/etc/rc.local` for executing and adding it on system startup (replacing EHC3 with the actual device name).

rc.local is a file that's used with sysVinit systems, it would be executed last when changing runlevels. As it's more common to encounter systemd we need to use an alternative, creating a script and a matching systemd service.

```bash
sion@sion-dlnx:~/Toybox$ sudo chmod +x /usr/local/bin/customstartup.sh 
sion@sion-dlnx:~/Toybox$ sudo cp customstartup.service /etc/systemd/system
```
After briefling installing them to make sure everything works I deleted them. Since I'm also trying to use Ansible more I'll go ahead and create a playbook which handles this. Yes It should technically be a role, but at this point I only have a couple things that need to happen so they'll be playbooks and get converted to roles soon enough.

disable_usb_wake.sh
```bash
#!/bin/bash
set -o errexit
set -o nounset
set -o pipefail

################################################################################
# Disable USB devices from waking computer

/usr/bin/systemd-cat -t "$(basename "$0")" disabling 'XHC' device from waking computer
echo "XHC" > /proc/acpi/wakeup
```

disable_usb_wake.service
```bash
################################################################################
# disable_usb_wake.service
#
# This service unit is to disable USB devices from waking computer
#
################################################################################
# This should be placed in /etc/systemd/system.
################################################################################

[Unit]

Description=Runs /usr/local/bin/disable_usb_wake.sh

  
[Service]

ExecStart=/usr/local/bin/disable_usb_wake.sh


[Install]

WantedBy=multi-user.target
```

disable_usb_wake.yml
```yaml
---
- name: Install systemd service disabling USB waking computer
  hosts: all
  tasks:
    - name: Install disable_usb_wake.sh script
      become: true
      ansible.builtin.copy:
        src: files/disable_usb_wake.sh
        dest: /usr/local/bin/disable_usb_wake.sh
        mode: "0544"
        owner: root
        group: root
    - name: Install disable_usb_wake service
      become: true
      ansible.builtin.copy:
        src: files/disable_usb_wake.service
        dest: /etc/systemd/system/disable_usb_wake.service
        mode: "0644"
        owner: root
        group: root
    - name: Start disable_usb_wake service
      become: true
      ansible.builtin.systemd:
        enabled: true
        name: disable_usb_wake
```
