---
layout: post
title: Advanced Ansible for Development Infrastructure
comments: true
---

## Introduction

There is more to Ansible than just a plain playbook that describes the desired state of a bunch of machines. Let's look at how it's possible to reuse Ansible tasks and playbooks by taking advantage of advanced tools at our disposal.

<!--more-->

The [previous post]({% post_url 2017-07-13-software-engineer-meets-ansible %}) was more of a personal first encounter with Ansible. Today, we will look into transforming the simple playbooks into something that can be reused and, preferably, shared publicly.

As a remainder, there are 3 main objectives (= 3 actual playbooks).

1. keep all machines updated
2. update ovirt-engine using ovirt-release-master repository on the engine node
3. build & deploy VDSM on all nodes

There is also a new constraint for VDSM building and deployment: everything has to use least privileges possible. The RPMs must not be built under root user, but under a new `developer` account. The account must be present on all host -- one of the playbooks needs to take care of that.

## Variables, Includes
To enable code reuse, variables are our first step. Although they are not super useful for everything in the Ansible land, some tasks simply beg to be parametric. Perfect example is a creation of a user. Ansible ensures that the specified user exists on the target machines by a [task](http://docs.ansible.com/ansible/glossary.html#term-tasks) like this:

```yaml
- name: 'ensure a user developer is present'
  user:
      name: developer
      groups: wheel
      append: yes
      state: present
```

Such task definition by itself isn't generic enough and has a simple caveat -- we don't set the user password. The password part of the issue is actually pretty simple: we already know of group [variables](http://docs.ansible.com/ansible/playbooks_variables.html), so it's only a matter of adding `password: '{{ '{{' }} password{{ }} }}'` to the user attributes and having the password defined under group\_vars (make sure it's in correct format though, as explained in `ansible_doc user`).

Now, what if we need another user? Could we just copy the definition and change the name? That's like a recipe for a maintenance hell. Instead, Ansible has a concept of [includes](http://docs.ansible.com/ansible/playbooks_roles.html). The user creation can be playbook or task on it's own, and the main playbook just includes it. On top of that, includes support passing variables. Instead of creating a specific user in a [playbook](http://docs.ansible.com/ansible/glossary.html#term-playbooks)

```yaml
- hosts: nodes
  tasks:
      - name: 'ensure a user developer is present'
        user:
            name: developer
            groups: wheel
            append: yes
            state: present
        become: yes
```

we may define the task separately using a variable(s) (in this case, the tasks is defined in `tasks/` subdirectory)

```yaml
- name: 'ensure a user '{{ '{{' }} user{{ }} }}' is present'
  user:
      name: '{{ '{{' }} user{{ }} }}'
      groups: wheel
      append: yes
      state: present
```

and include the task:

```yaml
- hosts: nodes
  tasks:
      - include: tasks/add_user.yml user=developer
```

We are now able to create any user by just including the task and setting the `user` variable to the desired value. We could also pass multiple variables which is extremely helpful when setting up standard users as well as root user. Root's home usually resides at `/root`, regular users have their home set up at `/home/$username`. That means if we have a critical task such as making sure both root and developer users have oh-my-zsh set-up, everything can be handled by a single task:

```yaml
- name: 'Clone oh-my-zsh repo ({{ '{{' }} user{{ }} }})'
  git:
      repo: https://github.com/robbyrussell/oh-my-zsh.git
      dest: '{{ '{{' }} password{{ }} }}/{{ '{{' }} user }}/.oh-my-zsh'
```

where path is either `/` or `/home`, depending on whether the user to be created is root or a user whose home directory resides elsewhere.

Includes can be partially automated by using roles, but that is out of scope for this deployment. It seems that roles are mostly suited to application deployment -- you can have a look at [openshift-ansible](https://github.com/openshift/openshift-ansible) to get an idea of how roles are used.

## ansible\_user, become, become\_user
Privilege escalation is an important subject. Not every task is made equal, and while executing everything as root works, it's not exactly the best practice. A great example where it's preferred to use regular user is [RPM building](https://fedoraproject.org/wiki/How_to_create_an_RPM_package#Preparing_your_system). Since we are dealing with development environment, this is probably the only case where dropping privileges is essential.

The usage of `become` depends on how `ansible_user` is set. If `ansible_user` is root, we are implicitly privileged -- in that case, `become` and `become_user` are a means of dropping privileges. On the other hand, if `ansible_user` is unprivileged, `become` is a critical tool to manipulate things that normally require sudo or other means of privilege escalation.

### unprivileged ansible\_user
If Ansible is instructed to run as a non-privileged user, `become` usage is trivial. A task to update the system via yum

```yaml
- name: update everything
  yum:
      name: '*'
      state: latest
```

becomes


```yaml
- name: update everything
  yum:
      name: '*'
      state: latest
  become: yes
```

### privileged ansible\_user

Privileged (let's assume root) case is similar to the unprivileged case with a small caveat: `become` must be called simultaneously with `become_user`. It's extremely easy to get burned by not declaring both -- `become_user` does not imply `become` (the documentation states that though). So if we have a [play](http://docs.ansible.com/ansible/glossary.html#term-plays) to build an RPM, we may define the privilege (de)escalation at the top of the play:


```yaml
- hosts: nodes
  become: yes
  become_user: developer
  vars:
      user: developer
      local_vdsm: ~/Projects/vdsm/
  tasks:
      - name: make vdsm RPMs
        make:
            chdir: '/home/{{ user }}/vdsm'
            target: rpm
```

## Handlers

Handlers are the callback mechanism in Ansible. Each task may notify a handler - either a single handler by name, or a group of handlers that listen to specific notification. After experimenting with handlers in the infrastructure deployment, two important points surfaced:

1. it's better to use listen/notify than notifying handler by name
2. passing variables to handlers is annoying, causing parametrized tasks to be a better choice in most of the situations (see this [gist](https://gist.github.com/schlueter/85bd22cd6c9e2bd054a3))

The basis for 1. is that calling handler by name won't work if handler's name changes. That is unnecessary coupling of unrelated information. As for 2., handlers use globally defined variables or facts. There doesn't seem to be a way to pass a variable to it directly except for the mentioned global setting that every tasks would be required to do.

In the end, I've found the usage of handlers in the management of a small infrastructure to be more of a complication than improvement. That would probably change if playbooks required multiple service restarts -- that is the perfect case for handlers.

```yaml
handlers:
    - name: restart vdsm
      service: name=vdsm state=restarted
      listen: "restart vdsm"

tasks:
    - name: configure vdsm
      command: vdsm-tool configure --force 
      notify: "restart vdsm"
```

## Vault and Keeping Playbooks Public
[Vault](http://docs.ansible.com/ansible/playbooks_vault.html) is a way to encrypt various files, mostly variable definitions, in Ansible. Although the idea sounds perfect, it's questionable what kind problem it solves. Sharing encrypted files in a plain sight is a terrible idea as vulnerability in the encryption algorithm could be found, encryption key leaked and other doomsday scenarios could happen. The most sensible usage is internal security, but even then the vault should not be treated as an all-secure solution.

## Summary

The whole work resulted in a following file hierarchy:

```
$ tree .
.
├── deploy.yml
├── engine.yml
├── files
│   ├── authorized_keys
│   ├── motd
│   ├── rpmmacros
│   └── zshrc
├── group_vars
│   └── ovirt-all
├── hosts
├── tasks
│   ├── add_user.yml
│   ├── setup_ssh.yml
│   ├── setup_user_rpmbuild.yml
│   └── setup_zsh.yml
└── update.yml
```

Unfortunately due to a number of private repositories used on the hosts, the idea of keeping all playbooks private doesn't work without destroying readability of the playbooks. The share-able playbooks are VDSM to host deployment [gist](https://gist.github.com/mpolednik/fac2fb927b0d7d09073754c6442ce35e) and engine deployment [gist](https://gist.github.com/mpolednik/3be277e34ac8e27e17b5ca291796de7a). That is an unfortunate result -- keeping the playbooks in public would make the infrastructure easily reproducible without internal communication. If you have any idea how to solve that, feel free to leave a comment - I'd be happy to try it.

Except for the sharing hiccup, ansible seems like a perfect candidate for this kind of task. The machines are easily reinstalled using [kickstart](http://pykickstart.readthedocs.io/en/latest/) and then brought up to date & configured with few playbooks. The MTTD (mean time to deployment) is now shorter, requires no user intervention and allows for a peaceful lunch breaks.
