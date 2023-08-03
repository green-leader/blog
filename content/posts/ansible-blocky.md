+++
title = "Ansible Blocky"
publishDate = "2023-08-03T08:00:00Z"
tags = ["Ansible", "System Administration", "DNS", "Blocky"]
showFullContent = false
readingTime = false
hideComments = false
+++

I've given myself a personal goal of creating 100 ansible playbooks, I suppose roles would also count. The destination is essentially accomplishing 100 goals with ansible.

These are what I've already accomplished.

- blocky.yml
- install_disable_usb_wake.yml
- install_sshd_config.yml
- install_sudo.yml
- lynis.yml
- podman.yml
- promtail.yml
- run_updates.yml

Overall not terribly complex roles, most of them have hardcoded defaults but that'll probably get resolved when the existing playbooks get converted to roles.


Most recently I made a playbook that setup and configured blocky-dns which would be my 8th playbook.

What's accomplished:
1. Remove systemd-resolved
1. Install the DNS server
1. Setup a systemd unit
1. Start the service

### Remove systemd-resolved

```yml
- name: Remove and purge systemd-resolved
    become: true
    ansible.builtin.apt:
    name: systemd-resolved
    state: absent
    purge: true
```

We're doing the fairly standard process of removing a package, using the apt module, and in this case we're working on `systemd-resolved`. Mainly this is being removed because it would interfere with blocky dns doing its job for a couple reasons. First and foremost that it's currently occupying port 53 and thus blocky wouldn't be able to secure the rights to use it. Another issue being that it can act as a DNS stub resolver, which isn't quite what we want in this instance. Overall I think it's great and definitely makes things a little easier in linux than they used to be but as we're just setting up a full dns server we won't need it.

### Install the DNS server

```yml
- name: Download binary
  ansible.builtin.unarchive:
    src: "https://github.com/0xERR0R/blocky/releases/download/v0.21/blocky_v{{ blocky_version }}_Linux_x86_64.tar.gz"
    dest: /tmp
    remote_src: true

- name: Install binary
  become: true
  ansible.builtin.copy:
    src: /tmp/blocky
    dest: /usr/local/bin/
    remote_src: true
    mode: "0755"
    owner: root
    group: root

- name: Create config directory - {{ service_name }}
  become: true
  ansible.builtin.file:
    path: "/etc/{{ service_name }}"
    state: directory
    mode: "0755"
    owner: root
    group: root

- name: Ensure config in place for - {{ service_name }}
  become: true
  ansible.builtin.copy:
    src: blocky-config.yml
    dest: "/etc/{{ service_name }}/config.yml"
    mode: "0644"
    owner: root
    group: root
  notify:
    - Reload service
```

This is a fairly common pattern for using things released on Github, or otherwise not desirable via the respective distributions package manager. Not much different than what a user would be doing. Download, Place binary in proper place, and create config files. I'm not convinced the parmisions are the most secure but just short of creating seperate groups for binaries based on the service user that should be able to use them, we'll call it good enough.

### Setup a systemd unit

```yml
- name: Create service user - {{ service_name }}
  become: true
  ansible.builtin.user:
    name: "{{ service_name }}"
    system: true
    state: present

- name: Create service file - {{ service_name }}
  become: true
  ansible.builtin.copy:
    content: |
      [Unit]
      Description=Service file for blocky-dns
      After=network.target

      [Service]
      Type=simple
      User=blocky
      ExecStart=/usr/local/bin/blocky --config=/etc/blocky/config.yml
      AmbientCapabilities=CAP_NET_BIND_SERVICE
      Restart=always

      [Install]
      WantedBy=multi-user.target
    dest: /etc/systemd/system/blocky.service
    mode: "0644"
    owner: root
    group: root
  notify:
    - Reload systemd daemon

- name: Start service - {{ service_name }}
  become: true
  ansible.builtin.systemd:
    name: "{{ service_name }}"
    enabled: true
    state: started
```

There's a lot of substitutions going on here. `{{ service_name }}` is a Jinja template. If the variable `service_name` is defined then the entire string will be replaced with whatever is set to that variable. the other interesting and different thing is `AmbientCapabilities=CAP_NET_BIND_SERVICE` being defined in the service file. This gives the service the capability to use port numbers <1024. Some documentation is [here](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#Capabilities) essentially this is a list of capabilities that are being included for the executed process as a non-privleged user.

Then we start the service.


### The end result

```yaml
---
- name: Installing and configuring blocky-dns
  hosts: all
  vars:
    blocky_version: "0.21"
    service_name: "blocky"
  tasks:
    - name: Remove and purge systemd-resolved
      become: true
      ansible.builtin.apt:
        name: systemd-resolved
        state: absent
        purge: true

    - name: Download binary
      ansible.builtin.unarchive:
        src: "https://github.com/0xERR0R/blocky/releases/download/v0.21/blocky_v{{ blocky_version }}_Linux_x86_64.tar.gz"
        dest: /tmp
        remote_src: true

    - name: Install binary
      become: true
      ansible.builtin.copy:
        src: /tmp/blocky
        dest: /usr/local/bin/
        remote_src: true
        mode: "0755"
        owner: root
        group: root

    - name: Create config directory - {{ service_name }}
      become: true
      ansible.builtin.file:
        path: "/etc/{{ service_name }}"
        state: directory
        mode: "0755"
        owner: root
        group: root

    - name: Ensure config in place for - {{ service_name }}
      become: true
      ansible.builtin.copy:
        src: blocky-config.yml
        dest: "/etc/{{ service_name }}/config.yml"
        mode: "0644"
        owner: root
        group: root
      notify:
        - Reload service

    - name: Create service user - {{ service_name }}
      become: true
      ansible.builtin.user:
        name: "{{ service_name }}"
        system: true
        state: present

    - name: Create service file - {{ service_name }}
      become: true
      ansible.builtin.copy:
        content: |
          [Unit]
          Description=Service file for blocky-dns
          After=network.target

          [Service]
          Type=simple
          User=blocky
          ExecStart=/usr/local/bin/blocky --config=/etc/blocky/config.yml
          AmbientCapabilities=CAP_NET_BIND_SERVICE
          Restart=always

          [Install]
          WantedBy=multi-user.target
        dest: /etc/systemd/system/blocky.service
        mode: "0644"
        owner: root
        group: root
      notify:
        - Reload systemd daemon

    - name: Start service - {{ service_name }}
      become: true
      ansible.builtin.systemd:
        name: "{{ service_name }}"
        enabled: true
        state: started

  handlers:
    - name: Reload systemd daemon
      become: true
      ansible.builtin.systemd:
        daemon_reload: true

    - name: Reload service
      become: true
      ansible.builtin.systemd:
        name: "{{ service_name }}"
        state: restarted
```