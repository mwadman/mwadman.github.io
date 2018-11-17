---
layout:     post
title:      "Ansible - Firewalld Module on Ubuntu"
subtitle:   "Getting Ansible firewalld tasks working on Ubuntu based systems"
date:       2018-11-17
author:     "Michael Wadman"
catalog:    true
tags:
    - Ansible
    - Ubuntu
---

# Overview

When trying to run an Ansible "firewalld" task on an Ubuntu machine recently, I received the following error:

```bash
TASK [Firewalld Task] **************************
fatal: [vagrant-test]: FAILED! => {"changed": false, "msg": "Python Module not found: firewalld and its python module are required for this module, version 0.2.11 or newer required (0.3.9 or newer for offline operations)"}
```

Here is the task that I'm running:

```yaml
name: Firewalld Test Task
firewalld:
  service: https
  state: enabled
```


# Investigation

After a quick google search, I found some [other](https://github.com/ansible/ansible/issues/24855) [people](https://github.com/ansible/ansible-modules-extras/issues/1282) have run into this as well.  
Although there were some fixes noted in the above issues, none of them sat well with me as they required installing extra packages.

There has to be a better way right?

Hunting further took me back to the Ansible [firewalld module documentation](https://docs.ansible.com/ansible/2.7/modules/firewalld_module.html), which includes the following notes:

> Not tested on any Debian based system.

> Requires the python2 bindings of firewalld, which may not be installed by default.

> For distributions where the python2 firewalld bindings are unavailable (e.g Fedora 28 and later) you will have to set the ansible_python_interpreter for these hosts to the python3 interpreter path and install the python3 bindings.

Those last points ring true with us. But how do we go about fixing this?

# Rectification

Well, turns out that the fix is considerably easier than installing extra packages.

Following the advice from that last note in the documentation, we first need to check what version of the bindings we're using:

```bash
$ apt-cache depends firewalld | grep Depends | grep python
  Depends: python3-dbus
  Depends: python3-gi
  Depends: python3-slip-dbus
  Depends: <python3:any>
```

Now that we've confirmed we're using the python3 bindings, we'll simply tell Ansible to use python3 when running the task.  
This is accomplished by setting the variable `ansible_python_interpreter` to "/usr/bin/python3".

In true Ansible fashion, the variable can be set in multiple ways/places. But because I don't want it to impact the running of my other tasks, I'm going to use the task level `vars` option, as below:

```yaml
name: Firewalld Test Task
firewalld:
  service: https
  state: enabled
vars:
  ansible_python_interpreter: '/usr/bin/python3'
```

This results in a happy firewalld task and a change of the firewalld state on the host:

```bash
TASK [Firewalld Task] **************************
changed: [vagrant-test]
```

# Conclusion

This was another quick post, which may help other people who are running into a similar issue.  

I could have gone into more detail around how I might refactor this task to include logic so that the python interpreter is only changed when running against Ubuntu machines, but I wanted to keep this quick and easy to read.

## References:

[Ansible Github Issue #24855](https://github.com/ansible/ansible/issues/24855)  
[Ansible Modules Extras Github Issue #1282](https://github.com/ansible/ansible-modules-extras/issues/1282)

## Versions used:

Local Machine:
- Operating System: *Kubuntu-18.04.1*
- Ansible: *2.7.2*  

Remote Machine:
- Operating System: *Ubuntu-16.04.5*  
- Firewalld: *0.4.0*
