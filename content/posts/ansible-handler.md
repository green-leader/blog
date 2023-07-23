+++
title = "Ansible Handler"
publishDate = "2023-07-23T08:00:00Z"
tags = ["Ansible", "System Administration", "Today I Learned", "Promtail", "Loki"]
showFullContent = false
readingTime = false
hideComments = false
+++

Spent a good part of the morning working on writing an Ansible playbook to install and setup promtail on a new server, adding it to the central logging setup.

At first I started manually downloading the binary and manually installing it when I remembered I wanted to work on using ansible more. So I stopped where I was and started over again except this time electing to use the power of automation.
Pretty simple what we needed to do

1. Download and install the binary
1. Install the config file
1. Create a service user
1. Create a systemd unit and start it

### Create an empty playbook

```yml
---
- name: Installing and configuring Promtail
  hosts: all
  tasks:
    - name: Hello World
      ansible.builtin.debug:
        msg: "Hello World!"
```

As with any good software project we start with a basic template which we'll expand upon. This simply outputs "Hello World!" and finishes the playbook.

and to check our work:
`ansible-playbook -i 192.0.2.2, playbook.yml`

### Download and install the binary

Replacing our hello world task we create two more, the first:

```yml
- name: Download binary
    ansible.builtin.unarchive:
    src: "https://github.com/grafana/loki/releases/download/v{{ promtail_version }}/promtail-linux-amd64.zip"
    dest: /tmp
    remote_src: true
```

There are a few things of note going on here. The module will automatically download and extract the archive from the URI given (but in this case "unzip" needs to be installed on the target host). Notice how the URI is in quotes and some curly braces are in the middle, this is because further up the document we've defined the version of promtail we want as a variable. Like so

```yml
vars:
  promtail_version: "2.8.2"
```

Lastly we've set `remote_src` to true, this tells ansible that the file in question is located on the host that it's currently working on.

The task responsible for installing the freshly downloaded binary.

```yml
- name: Install binary
  become: true
  ansible.builtin.copy:
    src: /tmp/promtail-linux-amd64
    dest: /usr/local/bin/promtail
    remote_src: true
    mode: "0754"
    owner: root
    group: root
```

This one is fairly self explanatory. We're using copy to move the binary. Then changing the owner and setting permissions. Again `remote_src` is used to tell ansible the file is on the host it's working on. I like specifying become on the per task level as opposed to per playbook, this helps with only needing to be root when needed.

### Install the config file

```yml
- name: Create Promtail directory
  become: true
  ansible.builtin.file:
    path: "/etc/promtail"
    state: directory
    mode: "0755"
    owner: root
    group: root
- name: Ensure Promtail config in place
  become: true
  ansible.builtin.template:
    src: promtail-config.yml.j2
    dest: /etc/promtail/config.yml
    mode: "0644"
    owner: root
    group: root
  notify:
    - Reload promtail service
```

These two tasks are pretty simple, create a directory for promtail config files. Then copy the config file, while replacing the jinja variables in the template. While I was iterating on the file, I ran into a minor issue where I would run the playbook with an updated config file. It would successfully run, but promtail was using the old config file. I ended up learning about Ansible Handlers. Before I had a task hacked together that would check if the previous task finished or not. Now I have a handler at the end of the document. Handlers are described a bit [here](https://ansible.readthedocs.io/projects/lint/rules/no-handler/).

```yml
- name: Reload promtail service
  become: true
  ansible.builtin.systemd:
    name: promtail
    state: restarted
```

The `notify` keyword at the end of the task triggers a handler when tasks are changed. Then the handler (which is identical to a normal task) is executed, which in this case reloads the service (which we haven't defined yet).

### Create a service user

This is a pretty simple task, we just create a system user, and give it access to the `adm` group, this will give it permissions to access most of the logs found in `/var/logs`. By creating a system user we're making a user that isn't for users. This is an automated account that processes can run over, this is best practice for security as we don't want processes running under regular users or root.

```yml
- name: Create Promtail service user
  become: true
  ansible.builtin.user:
    name: promtail
    system: true
    state: present
    groups: 'adm'
```

### Create a systemd unit and start it

Again, just copying a file into place on the server, except we're also giving the text that should be in the source file instead of creating a source file. I haven't decided on a standard way to do this, and with a simple enough service file like the below it's probably okay to define in this case.

After installing the service, we start and enable it, then define a handler for in the event we replace service file with something else.

```yml
- name: Create Promtail service file
    become: true
    ansible.builtin.copy:
    content: |
        [Unit]
        Description=Promtail service for Grafana Loki
        After=network.target

        [Service]
        Type=simple
        User=promtail
        ExecStart=/usr/local/bin/promtail -config.file=/etc/promtail/config.yml
        Restart=always

        [Install]
        WantedBy=multi-user.target
    dest: /etc/systemd/system/promtail.service
    mode: "0644"
    owner: root
    group: root
    notify:
    - Reload systemd daemon

- name: Start Promtail service
    become: true
    ansible.builtin.systemd:
    name: promtail
    enabled: true
    state: started

handlers:
- name: Reload systemd daemon
    become: true
    ansible.builtin.systemd:
    daemon_reload: true
```

### The end result

This is what I ended up getting for the end result. In all it is pretty straightforward, and I think fairly easy to understand.

```yml
---
- name: Installing and configuring Promtail
  hosts: all
  vars:
    # Get a version that matches Loki
    promtail_version: "2.8.2"
  tasks:
    - name: Download binary
      ansible.builtin.unarchive:
        src: "https://github.com/grafana/loki/releases/download/v{{ promtail_version }}/promtail-linux-amd64.zip"
        dest: /tmp
        remote_src: true

    - name: Install binary
      become: true
      ansible.builtin.copy:
        src: /tmp/promtail-linux-amd64
        dest: /usr/local/bin/promtail
        remote_src: true
        mode: "0754"
        owner: root
        group: root

    - name: Create Promtail directory
      become: true
      ansible.builtin.file:
        path: "/etc/promtail"
        state: directory
        mode: "0755"
        owner: root
        group: root

    - name: Ensure Promtail config in place
      become: true
      ansible.builtin.template:
        src: promtail-config.yml.j2
        dest: /etc/promtail/config.yml
        mode: "0644"
        owner: root
        group: root
      notify:
        - Reload promtail service

    - name: Create Promtail service user
      become: true
      ansible.builtin.user:
        name: promtail
        system: true
        state: present
        groups: 'adm'

    - name: Create Promtail service file
      become: true
      ansible.builtin.copy:
        content: |
          [Unit]
          Description=Promtail service for Grafana Loki
          After=network.target

          [Service]
          Type=simple
          User=promtail
          ExecStart=/usr/local/bin/promtail -config.file=/etc/promtail/config.yml
          Restart=always

          [Install]
          WantedBy=multi-user.target
        dest: /etc/systemd/system/promtail.service
        mode: "0644"
        owner: root
        group: root
      notify:
        - Reload systemd daemon

    - name: Start Promtail service
      become: true
      ansible.builtin.systemd:
        name: promtail
        enabled: true
        state: started

  handlers:
    - name: Reload systemd daemon
      become: true
      ansible.builtin.systemd:
        daemon_reload: true

    - name: Reload promtail service
      become: true
      ansible.builtin.systemd:
        name: promtail
        state: restarted
```
