---
layout: post
title: Software Engineer meets Ansible
comments: true
---

## Introduction

"Ansible is the simplest way to automate apps and IT infrastructure." Or at least that's what they say. I've been thinking of it for quite a while but only recently decided to give it a go (like, two days ago!). Let's see how Ansible works in slightly unusual software development setup.

<!--more-->

## Me and My Setup

I've mentioned that my setup may not be the most usual for developers, why? To develop oVirt, I maintain a small cluster for me and my colleagues to use. It consists of a VM where ovirt-engine runs and multiple physical machines. The machines are a combination of standard 1U pizza boxes, a [beefy workstation]({% post_url 2016-09-19-real-time-host-in-ovirt %}#poc-setup) and even beefier POWER8 2U machine.

For the sake of simplicity, the post will assume 6 machines:

```
pizza1 \
pizza2  +- 1U servers
pizza3 /
ws - the workstation
power8 - the POWER8 server
engine - the VM running ovirt-engine
```

Luckily, I don't have to maintain the storage -- I'm using a given NFS mount with enough storage space for everything. As for the software side, it's pretty simple: development version of RHEL 7 almost everywhere (latest CentOS would suffice, but I'm used to RHEL). The engine runs on CentOS.

## Automation Objectives

There are 3 tasks to be automated in the development environment:

1. keeping the systems updated and configured
2. updating the ovirt-engine instance (keeping up with master snapshots is enough)
3. building and deploying VDSM, the host management agent of oVirt

Easy enough. Reading the awesome [Ansible documentation](http://docs.ansible.com/ansible/intro_getting_started.html), all I have to do is

1. create an inventory of the systems
2. create 3 [playbooks](http://docs.ansible.com/ansible/playbooks.html), each handling one of the objectives above
3. ansible -i inventory playbook.yml

## Ansible Inventory

Ansible uses a concept of [inventory](http://docs.ansible.com/ansible/intro_inventory.html), a simple .yml file that lists all the machines that will be managed. The documentation also mentions that the machines may be grouped and that there are mechanisms to define some variables for the groups. I came up with a pretty simple hierarchy:

```yaml
[engine]
engine

[nodes-workstation]
ws

[nodes-x86_64]
pizza1
pizza2
pizza3

[nodes-ppc64le]
power8

[nodes:children]
nodes-workstation
nodes-x86_64
nodes-ppc64le

[everything:children]
engine
nodes
```

Why so many groups? The power8 and ws machines will most likely need different treatment, and so will the engine. Therefore the hierarchy allows us to target everything including the engine (to cover yum -y update), just the nodes (VDSM deployment), the engine (ovirt-engine updates) or nodes by their architecture/use (GPU specific stuff goes to nodes-ws, power8 to nodes-ppc64le). Handy feature!

Although the variables may be defined in multiple ways, the [best practices](http://docs.ansible.com/ansible/playbooks_best_practices.html) document outlines a neat project structure. The following is the one I use:

```
hosts
group_vars
- everything
files
*.yml
```

where

```
$ cat groups_vars/everything

ansible_user: root
```

Using root is quite an evil thing -- and yet I use the machines as root most of the time. Maybe with proper configuration management (wink wink Ansible), this habit will perish.

## Step 2: playbooks

Linux is awesome (not only) because it contains a proper shell. If it contains a shell, you can automate it. Ansible has a support for running shell commands as a [module](http://docs.ansible.com/ansible/modules.html). Knowing this, it's easy to march forward -- in the worst case, the playbook will be a bunch of awkward shell scripts written in yaml. It does, of course, contain more modules than just the shell - `file` for file handling stuff, `yum` for interfacing with yum, `make`... see where I'm going? It looks like most of the everyday administration tasks are already neatly wrapped in modules that abstract the commands required to accomplish anything in the Linux world. 

The first step of the evil automation master plan was to ensure that the machines have correct repositories set up. And here comes first (and so far unsolved) hiccup. Ansible allows a user to add or delete a repository, but my objective was to enable *only* the repositories I explicitly request. There doesn't seem to be anything that would do this in a nice, declarative way. It's probably possible to iterate through the [facts](http://docs.ansible.com/ansible/setup_module.html) and disable/enable specific repositories. Or just keep the repository file locally, `rm /etc/yum.repos.d/\*` and then copy the stuff over. I don't like either, so for now the playbook just makes sure ovirt-release-master is installed and extra repo is present.

Everything else was a breeze: adding a /etc/motd (totally required, preferably generated with cowsay), making sure my SSH key is authorized, git installed... you name it. The file looks like this:

```yaml
- hosts: nodes
  tasks:
      - name: make sure that correct repositories are enabled
        yum_repository:
            description: repo
            name: repo
            baseurl: repo
            enabled: yes
            file: repo
            state: present

- hosts: all
  tasks:
      - name: add motd warning
        copy:
            src: files/motd
            dest: /etc/motd

      - name: make sure user has .ssh directory
        file:
            path: /root/.ssh
            state: directory
            owner: root
            group: root
            mode: 0700

      - name: add authorized keys
        copy:
            src: files/authorized_keys
            dest: /root/.ssh/authorized_keys
            owner: root
            group: root
            mode: 0600

      - name: install ovirt-release-master
        yum:
            name: 'http://resources.ovirt.org/pub/yum-repo/ovirt-release-master.rpm'
            state: present

      - name: install git
        yum:
            name: 'git'
            state: present

      - name: update everything
        yum:
            name: '*'
            state: latest
```

{% include image.html name="moo.png" width="100%" %}

## Step 2.1: host deployment

The deployment is something that was previously automated with a shell script, so moving to Ansible isn't a top priority. That being said, it's a useful practice and helped me find few more annoyances. The deployment goes roughly like this:

1. rsync the git repo to target machines
2. run ./autogen.sh --system && ./configure
3. make rpm
4. remove previous RPMs
5. install the new RPMs
6. do some misc stuff and restart the service

So the first thing that bothers me is that rsync is provided by Ansible in a module called `synchronize`. Why?! Except for that, nothing too fancy -- everything you expect from rsync is provided in the module. It's also not a bad idea to extend the machine update playbook above to also install rsync onto all nodes.

Autogen and configure? Just use `command` module and execute those directly. For make, there is `make` - nothing extra useful but gets the job done in a nice way. Getting rid of previous RPMs was a bit more painful: the previous shell script was

`for pack in $(rpm -qa | grep vdsm); do rpm -e $pack --nodeps; done`

After fiddling with `yum` module, I've found out that the simplest solution works: just call `shell` module with the script and be done with it. What really annoys me is the installation of new RPMs: initially, the plan was to use `yum` module and install everything in rpmbuild/RPMS/\*/vdsm-\*.rpm. Surprise, `yum` does not expand wildcards for local paths. It's probably possible to get all the files in rpmbuild/RPMS and feed them into `yum` module, but that gets annoying. `shell: yum -y install /root/rpmbuild/RPMS/*/vdsm-*.rpm` to the rescue. 

In the end, the deployment file isn't pretty, but it works:

```yaml
- hosts: nodes
  vars:
      local_vdsm: ~/vdsm/
  tasks:
      - name: remove previous build artifacts
        file:
            path: /root/rpmbuild
            state: absent

      - name: rsync over vdsm development directory
        synchronize:
            src: '{{ '{{' }} local_vdsm{{ }} }}'
            dest: /root/vdsm/
            delete: yes
            recursive: yes

      - name: make sure that the repo has correct permissions
        file:
            path: /root/vdsm/
            owner: root
            group: root
            recurse: yes

      - name: run autogen
        command: ./autogen.sh --system
        args:
          chdir: /root/vdsm

      - name: run configure
        command: ./configure
        args:
          chdir: /root/vdsm

      - name: make vdsm RPMs
        make:
            chdir: /root/vdsm
            target: rpm

      - name: uninstall previous vdsm version
        shell: for pack in `rpm -qa | grep vdsm`; do rpm -e $pack --nodeps; done

      - name: install the RPMs
        shell: yum -y install /root/rpmbuild/RPMS/*/vdsm-*.rpm

      - name: configure vdsm
        command: vdsm-tool configure --force 

      - name: restart the vdsm service
        service:
            name: vdsmd
            state: restarted
            enabled: yes
```

## Step 2.2: engine deployment

Actually, I didn't get that far. Something to hope for in a future post? :)

## Summary

In the end, I'm surprised. Ansible definitely delivers automation that is easy to use and even adapts to quite a weird use case. I've actually managed to discover much more -- variables, includes, loops... no more root building! I'm still trying to figure out secrets and hosts isolation so that I can keep the playbooks in public git repo and just the essential files private. You can expect part 2 somewhat soon - this post is long enough already!
