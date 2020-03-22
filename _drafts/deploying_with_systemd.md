---
title: Deploying a service with ansible and systemd
layout: post
tags:
    - ansible
    - systemd
    - devops
    - golang
---

## Introduction

You may be a sole developer or member of a small development team with no dedicated ops people. You may also have a handful of small-ish services, a few cronjobs and a couple of VPSs to run them on. What are your options? At one end of the spectrum, there's kubernetes as the current best practice recommended™ by the internetz. However, you can't pay the price (i.e. time) or do not have the desire to ride the steep learning curve that this path entails. On the other end of the spectrum, there's always `rsync`/`scp` and bash scripts but you'd like something better than that (including process management, logs, infrastructure as code etc.). So, is there anything worthwile in between these two extremes?

This article is about how to deploy a service to a remote server using ansible and systemd. All the "configuration" that is neccessary to do that will be checked into a repo and will be easily reproducible on any set of servers without the need to log into the servers and do any manual work (apart from one thing at the beginning). Now, a few words about our components.

[Ansible](https://www.ansible.com/) is a tool for automating task execution in remote servers. It runs locally on your development machine and can connect to an arbitrary set of servers through ssh in order to execute the specified tasks without the need of an "agent" process on the server(s). There's a [wide variety of modules](https://docs.ansible.com/ansible/latest/modules/modules_by_category.html) that can accomplish common tasks such as creating users and groups, installing dependencies, copying files and many more.

[systemd](https://www.freedesktop.org/wiki/Software/systemd/) is the basic foundation of most linux systems nowadays as the replacement of sysvinit and can accomplish a wide variety of things including managing processes and services (the feature that we'll be using for this article).

For our demonstration, we will be using a service written in Go, which very nicely and conveniently consists of a single statically-linked binary, but the concepts are the same for anything that can be executed on the remote server (this includes programs writen in ruby/python/java/dotnet etc.). So, let's start!

## Prerequisites

We will be needing the following on our local (development) machine:

* a [working Go installation](https://golang.org/doc/install) in order to build our small service
* the [ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) tool
* the `make` program (check your system using `which make`)

I have assumed a remote server running Debian Buster (v10), but any linux system with sshd and systemd should do.

All the work that follows is checked into [this repo](https://github.com/kkentzo/deployment-ansible-systemd-demo) which can be cloned using `git clone https://github.com/kkentzo/deployment-ansible-systemd-demo.git`. The repo contains the following components:

* `cmd/demo/main.go`: our service
* `Makefile`: targets for building and deploying the service
* `demo.yml`: the description of our deployment (ansible)
* `roles/demo`: the deployment specifics of the demo dervice (ansible)
* `hosts`: the list of hosts to which the demo service will be deployed

## The Guide

### Writing the service

Our service is a very simple one: it accepts http requests and responds with a greeting to the client based on the contents of the url path. The code is really simple:

```go
package main

import (
	"fmt"
	"log"
	"net/http"
)

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		var name string
		if name = r.URL.Path[1:]; name == "" {
			name = "stranger"
		}
		fmt.Fprintf(w, "hello %s!", name)
	})
	log.Fatal(http.ListenAndServe(":9999", nil))
}
```

The code above starts an http server that listens in port 9999. If the url path is the root path ("/") then the service greets the "stranger", otherwise it greets whoever is mentioned in the url path (e.g. `GET /world` will return "hello world!").

This file is placed under `cmd/demo` as `main.go` in our working directory and can be built in executable form (under `bin/`) as follows:

```bash
$ go build -o ./bin/demo ./cmd/demo/...
```

OK, so now we have our service - how about we deploy it?

### Deploying the service

We will use ansible to deploy our service to our remote server as a systemd service unit. The remote server can be any linux system with ssh and systemd. If you don't have access to such a system, you can use a tool such as [virtual box](https://www.virtualbox.org/) in order to setup a debian buster system.

We will specify our remote server in file `hosts` for use by ansible:

```
[local]
127.0.0.1

[myserver]
harpo
```

As you can see, this file can specify multiple named server groups (names in `[]` brackets). Apart from the `local` group which we do not intend to use,  we have specified the section `myserver` which contains the name of our server `harpo`. In this case, `harpo` is an alias defined in our `.ssh/config` file as follows:

```
Host harpo
     HostName 12.34.56.789
     User USERNAME
     IdentityFile ~/.ssh/harpo
```

This configuration facilitates our ssh access to the remote server and assumes that we have [correctly set up ssh access](https://www.digitalocean.com/community/tutorials/how-to-configure-ssh-key-based-authentication-on-a-linux-server) for user `USERNAME` in the server located in the address `12.34.56.789` (pls replace with your own server IP).

Now that we have specified our remote server, we need to define a role (workbook in ansible terminology) for our server as follows:

    $ cd roles
    $ ansible-galaxy init demo

The above command will generate a file/directory structure under `roles/demo` of which the following are relevant to our guide:

* `roles/demo/tasks/main.yml`: the sequence of tasks to execute on the server
* `roles/demo/handlers/main.yml`: actions to execute when a task is completed
* `roles/demo/files/`: contains the files that we will need to copy to the remote server

Let's start with the latter and define our systemd unit in file `roles/demo/files/demo.service`:

```
[Unit]
Description=Demo service

[Service]
User=demo
Group=demo
ExecStart=/usr/local/bin/demo

[Install]
WantedBy=multi-user.target
```

Ss you can see, systemd units are simply defined using a declarative language. In our case, we declare our service executable (`ExecStart`) that will run under user `demo`. The `[Install]` section specifies that our service requires a system state in [which network is up and the system accepts logins](https://unix.stackexchange.com/a/506374).

Now, that we have our systemd unit, let's define the tasks that we will need to run on the server (in file `roles/demo/tasks/main.yml`):

```yaml
---
- name: create demo group
  group:
    name: demo
    state: present

- name: create demo user
  user:
    name: demo
    groups: demo
    shell: /sbin/nologin
    append: yes
    state: present
    createhome: no

- name: Copy systemd service file to server
  copy:
    src: demo.service
    dest: /etc/systemd/system
    owner: root
    group: root
  notify:
    - Start demo

- name: Copy binary to server
  copy:
    src: demo
    dest: /usr/local/bin
    mode: 0755
    owner: root
    group: root
  notify:
    - Start demo
```

The task file is mostly self-explanatory but a few items need clarifications:

* each task has a name and references an ansible module that accepts parameters
* ansible's `group` module creates the specified group if it does not exist
* ansible's `user` module creates users
* ansible's `copy` module copies files that exist locally under `roles/demo/files` (such as `demo.service` that we created previously) to the remote server

Ansible's `notify` directive enqueues a particular handler (`Start demo`) to be executed after the completion of all tasks. All handlers are defined in file `roles/demo/handlers/main.yml`:

```yaml
---
- name: Start demo
  systemd:
    name: demo
    state: started
    enabled: yes
```

This notification uses ansible's `systemd` module and requires the service to be started and enabled (i.e. started every time the remote server boots).

Finally, we complete our ansible configuration by combining our hosts and roles in file `demo.yml`:

```yaml
---
- hosts: myserver
  become: yes
  become_user: root
  roles:
    - demo
```

Here, we declare that we would like to apply the role `demo` that we just defined to the specified hosts (just one: `myserver` as specified in our `hosts` file).

### Wrap up

We're almost there! Let's wrap up the whole thing in a project `Makefile` that contains the two targets of interest, build and deploy our service, as follows:

```
.PHONY: build
build:
	env GOOS=linux go build -o ./bin/demo ./cmd/demo/...

.PHONY: deploy
deploy: build
	cp ./bin/demo ./roles/demo/files/demo
	ansible-playbook -i hosts demo.yml
```

The `build` action compiles our service (for linux) and outputs the executable under `bin/`. The `deploy` target first builds the service, then copies the executable under the demo role's files and executes the entire ansible playbook by using the `demo.yml` spec.

Now, we can deploy our service by issuing:

```bash
$ make deploy
```

The output from my machine was as follows:

```
make deploy
env GOOS=linux go build -o ./bin/demo ./cmd/demo/...
cp ./bin/demo ./roles/demo/files/demo
ansible-playbook -i hosts demo.yml

PLAY [home] ********************************************************************

TASK [Gathering Facts] *********************************************************
ok: [harpo]

TASK [demo : create demo group] ************************************************
changed: [harpo]

TASK [demo : create demo user] *************************************************
changed: [harpo]

TASK [demo : Copy systemd service file to server] ******************************
changed: [harpo]

TASK [demo : Copy binary to server] ********************************************
changed: [harpo]

RUNNING HANDLER [demo : Start demo] ********************************************
changed: [harpo]

PLAY RECAP *********************************************************************
harpo                      : ok=6    changed=5    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```

You can test the service using curl:

```bash
$ curl 12.34.56.789:9999/world
```

where `12.34.56.789` needs to be replaced by the remote server's actual IP. You should see the output "hello world!"

### Status & Monitoring

We can also have a look on how the `demo` process is doing on our remote server by logging in (via ssh) and using the systemd commands `systemctl` and `journalctl` as follows:

```bash
# check the status of our service
$ sudo systemctl status demo
# tail our service's logs
$ sudo journalctl -f -u demo
```

## Epilogue

`ansible` and `systemd` are two fantastic tools that allow one to build automatic and reproducible operational pipelines quickly and efficiently.

The contents of this guide are in this repo.

<!-- Units are the objects that systemd knows how to manage. These are basically a standardized representation of system resources that can be managed by the suite of daemons and manipulated by the provided utilities. -->

<!-- Units in some ways can be said to similar to services or jobs in other init systems. However, a unit has a much broader definition, as these can be used to abstract services, network resources, devices, filesystem mounts, and isolated resource pools. -->

Prerequisites:
* `brew install ansible` on the controller machine
* run `sudo apt-get install python-apt` on all controlled machines!

Setup a new playbook:

    $ cd roles
    $ ansible-galaxy init ROLE_NAME

Execute a playbook on a server:

    $ ansible-playbook -i ./hosts server.yml

Use `-C` on the command above to "just check".



```
make deploy
env GOOS=linux go build -o ./bin/demo ./cmd/demo/...
cp ./bin/demo ./roles/demo/files/demo
ansible-playbook -i hosts demo.yml

PLAY [home] ********************************************************************

TASK [Gathering Facts] *********************************************************
ok: [harpo]

TASK [demo : create demo group] ************************************************
changed: [harpo]

TASK [demo : create demo user] *************************************************
