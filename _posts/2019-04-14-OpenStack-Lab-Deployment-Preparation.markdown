---
layout:     post
title:      "OpenStack Lab Network - Deployment Preparation"
subtitle:   "Preparing hosts for OpenStack-Ansible to be run"
date:       2019-04-14
author:     "Michael Wadman"
catalog:    true
tags:
    - OpenStack
    - Ansible
    - Vagrant
    - VirtualBox
    - Ubuntu
---

# Overview

This post is the tenth in a series that plans to document my progress through installing and configuring a small OpenStack Lab.

For other posts in this series, see the overview section of the [introduction post](https://www.wadman.co.nz/2018/02/08/OpenStack-Lab-Network-Introduction/#overview).

In the [last post](https://wadman.co.nz/2019/02/03/OpenStack-Lab-Host-Routing/), we installed and configured the FRR suite on our OpenStack hosts.
In this post, we'll start making the final touches before we run OpenStack-Ansible.

# Configuration

This post will be a quick one, with the goal of preparating our OpenStack hosts for the OpenStack services to be deployed to them.

The steps that I'm going to cover will be a direct translation of what is listed in OpenStack-Ansible's deployment guide. Specifically, the following pages:
- [OpenStack-Ansible: Prepare the deployment host](https://docs.openstack.org/project-deploy-guide/openstack-ansible/latest/deploymenthost.html)
- [OpenStack-Ansible: Prepare the target hosts](https://docs.openstack.org/project-deploy-guide/openstack-ansible/latest/targethosts.html)

Let's start with the deployment host preparation.

## Deployment Host

The deployment guide recommends that (in a testing environment) one of the control nodes act as the deployment host as well.  
In production this role would instead be handed to a host that won't be participating in the OpenStack services, to avoid potential conflicts occurring during initial deployment or upgrades (as the deployment host will act as the source of any future updates to OpenStack services).

### Ansible Playbook and Role

With that in mind, let's create a new role and add this to our existing playbook.

```yaml
---
- name: Configuring OpenStack hosts
  hosts: openstack_hosts
  become: true
  gather_facts: true
  roles:
    - { role: openstack-interfaces, tags: [ 'openstack-interfaces' ] }
    - { role: openstack-routing, tags: [ 'openstack-routing' ] }

- name: Prepare deployment host
  hosts: openstack-control
  become: true
  gather_facts: true
  roles:
    - { role: openstack-deployment, tags: [ 'openstack-deployment' ] }
```

Note that instead of adding this role onto the existing "Configuring OpenStack hosts" entry, I've defined another entry.  
By doing this, I can specify that the role is only to be run on `hosts: openstack-control`.

We'll create the skeleton of our new role too.

```bash
$ mkdir -p /etc/ansible/roles/openstack-deployment/tasks/
```

### Packages

As always, our first step is to ensure the required packages are installed on our host.  
The deployment guide gives us a list of [required packages](https://docs.openstack.org/project-deploy-guide/openstack-ansible/latest/deploymenthost.html#configure-ubuntu).

Here’s what the first task in our file “/etc/ansible/roles/openstack-interfaces/tasks/main.yml” looks like:

```yaml
- name: Install prerequisite packages
  apt:
    name:
      - aptitude
      - build-essential
      - git
      - ntp
      - ntpdate
      - openssh-server
      - python-dev
      - sudo
    state: latest
    cache_valid_time: 3600
```

### Clone Repository

Next, the guide says to clone their repository into the local directory "/opt/openstack-ansible".  
For this, we'll use Ansible's ["git" module](https://docs.ansible.com/ansible/latest/modules/git_module.html).

<!-- {% raw %} -->
```yaml
- name: Clone openstack-ansible repository
  git:
    repo: "https://github.com/openstack/openstack-ansible"
    dest: "{{ openstack_ansible_exec_dir }}"
    version: "stable/stein"
```
<!-- {% endraw %} -->

This module has a ton of parameters that are able to be passed in, but we're going to stay simple and use just the following:
- `repo` - The address to clone the repository from. I'll use the projects Github mirror as I've had issues with the official repository link in the past.
- `dest` - The directory to clone the repository into.
- `version` - The version of the repository to clone. In our case, we're going to reference the branch named "stable/stein", as we'll be installing the OpenStack version named "Stein" using OpenStack-Ansible.

I've also referenced the variable "openstack_ansible_exec_dir", which isn't yet defined.  
I'm going to define this variable (as "/opt/openstack-ansible"), along with another variable we'll use later, in the defaults file ("defaults/main.yml") for this role:

```yaml
---
openstack_ansible_exec_dir: "/opt/openstack-ansible"
openstack_ansible_config_dir: "/etc/openstack_deploy"
```

### SSH Authentication

During the running of the OpenStack-Ansible playbooks, Ansible on the deployment host will attempt to connect as the root user to other remote nodes.  
In order for this to work, we'll need to create and then set the root users' public key on the deployment (control) host as an authorised key on our compute host.

```yaml
- name: Create a SSH key for the root user
  command: ssh-keygen -q -t rsa -b 4096 -f /root/.ssh/id_rsa -C "" -N ""
  args:
    creates: /root/.ssh/id_rsa.pub
```

The above command will create the ssh-key if it doesn't already exist.  
In the future, we should be able to use the new ["openssh_keypair"](https://docs.ansible.com/ansible/devel/modules/openssh_keypair_module.html) module, but this is currently only available in the development release.

To store the contents of the public key file as a variable, we'll use Ansible's ["slurp"](https://docs.ansible.com/ansible/latest/modules/slurp_module.html) and ["set_fact"](https://docs.ansible.com/ansible/latest/modules/set_fact_module.html) modules.

<!-- {% raw %} -->
```yaml
- name: Get root user public key
  slurp:
    path: /root/.ssh/id_rsa.pub
  register: root_pub_key

- set_fact:
    openstack_deploy_pub_key: "{{ root_pub_key['content'] | b64decode }}"
```
<!-- {% endraw %} -->

"slurp" will gather the contents of a file on a remote host. Together with the ["register"](https://docs.ansible.com/ansible/latest/user_guide/playbooks_conditionals.html#register-variables) keyword, we can save this as a (base-64 encoded) variable.

We then use "set_fact" to decode and save the result as another variable.  
We need to save this off again, as the "register" action only saves the variable on the host the playbook is running on at the time, in our case the OpenStack control host, and we want this available across all of our hosts.

### Bootstrap OpenStack-Ansible

The last step for the deployment host is to run the bootstrap script that was cloned in the step before.

<!-- {% raw %} -->
```yaml
- name: Run bootstrap-ansible.sh script
  command: ./scripts/bootstrap-ansible.sh
  args:
    chdir: "{{ openstack_ansible_exec_dir }}"
    creates: "/usr/local/bin/openstack-ansible"
```
<!-- {% endraw %} -->

The `creates` argument is a way of telling the task whether it needs to be run or not, by saying what file is created if it has been run successfully.  
I've used the file "/usr/local/bin/openstack-ansible" as this is the last (we want the creation of the file to be as near to the end of the script as possible) file that is always created when the script is run.

## Target Hosts

Onto the target hosts. The [guide for which](https://docs.openstack.org/project-deploy-guide/openstack-ansible/latest/targethosts.html) is the next page on from the deployment host guide.

### Ansible Playbook and Role

We'll add our new role to the playbook first:

```yaml
---
- name: Configuring OpenStack hosts
  hosts: openstack_hosts
  become: true
  gather_facts: true
  roles:
    - { role: openstack-interfaces, tags: [ 'openstack-interfaces' ] }
    - { role: openstack-routing, tags: [ 'openstack-routing' ] }

- name: Prepare deployment host
  hosts: openstack-control
  become: true
  gather_facts: true
  roles:
    - { role: openstack-deployment, tags: [ 'openstack-deployment' ] }

- name: Configuring OpenStack hosts
  hosts: openstack_hosts
  become: true
  gather_facts: true
  roles:
    - { role: openstack-preparation, tags: [ 'openstack-preparation' ] }
```

And our role creation.

```bash
$ mkdir -p /etc/ansible/roles/openstack-preparation/tasks/
```

### Packages

You know the drill by now. [Pre-requisitie packages](https://docs.openstack.org/project-deploy-guide/openstack-ansible/stein/targethosts.html#configure-ubuntu) first.

```yaml
- name: Install prerequisite packages
  apt:
    name:
      - bridge-utils
      - debootstrap
      - ifenslave
      - ifenslave-2.6
      - lsof
      - lvm2
      - openssh-server
      - python
      - sudo
      - tcpdump
      - vlan
    state: latest
    cache_valid_time: 3600
```

### Kernel Modules

After installing packages, the guide then says to complete the following:
- Install the "linux-image-extra" package. For our case, this would be "linux-image-extra-virtual-hwe-18.04", which doesn't install anything other than a copyright text file in "/usr/share/doc", at least at the time of writing this.
- Configure the chrony service. The astute of you will notice that I have omitted the chrony package from the list in the previous step. This is because we really don't need to worry about time drift in an ephemeral lab deployment like ours. Even in a production environment, I would consider just configuring ntpd correctly instead.
- Load the kernel modules "8021q" and "bonding". Since we aren't bonding interfaces in our topology, we'll just enable the 8021q module below.

```yaml
- name: Enable required kernel modules
  modprobe:
    name: 8021q
```

Yes, that simple.

### SSH Authentication

Carrying on from where we left off in the SSH Authentication section for the deployment host, we now have a variable which stores the public key of the root user.  
All we need to do now is set this as an authorised key on all other hosts. Luckily there is an [Ansible module](https://docs.ansible.com/ansible/latest/modules/authorized_key_module.html) for doing just that.

<!-- {% raw %} -->
```yaml
- name: Set root user authorised ssh key
  authorized_key:
    user: root
    key: "{{ hostvars['openstack-control']['openstack_deploy_pub_key'] }}"
```
<!-- {% endraw %} -->

### Storage

Because we've already set up the networking, the last step is to [set up LVM storage](https://docs.openstack.org/project-deploy-guide/openstack-ansible/stein/targethosts.html#configuring-the-storage) for the Cinder (block storage) service and lxc.  
We'll be utilising the second hard disk that we set up on both of our OpenStack hosts in the [Vagrant configuration](https://wadman.co.nz/2018/04/08/OpenStack-Lab-Network-Vagrant/#storage).

<!-- {% raw %} -->
```yaml
- name: Create required partitions
  parted:
    name: "{{ item.name }}"
    state: present
    device: /dev/sdb
    number: "{{ item.number }}"
    part_start: "{{ item.part_start }}"
    part_end: "{{ item.part_end }}"
    label: gpt
    flags: [ lvm ]
  loop:
    - { name: cinder-volumes, number: 1, part_start: '0%', part_end: '50%' }
    - { name: lxc, number: 2, part_start: '50%', part_end: '100%' }

- name: Create required LVM volume groups
  lvg:
    vg: "{{ item.vg }}"
    pvs: "{{ item.pvs }}"
    vg_options: --metadatasize 2048
  loop:
    - { vg: cinder-volumes, pvs: /dev/sdb1 }
    - { vg: lxc, pvs: /dev/sdb2 }
```
<!-- {% endraw %} -->

First up we use the ["parted"](https://docs.ansible.com/ansible/latest/modules/parted.html) module to split our disk into two equal sized partitions. The naming of these is important, as the OpenStack-Ansible playbooks look for these specific names when running.  
Next, we create LVM volume groups using the ["lvg"](https://docs.ansible.com/ansible/latest/modules/lvg.html) module. Note that we don't need to create the physical volumes separately, as these are created automatically by the module if they don't exist.

# Conclusion

In this post, we've completed the final preparation steps (a bit of an oxymoron, I know) required on our OpenStack hosts.

## References:

[Ansible - "apt" module](https://docs.ansible.com/ansible/latest/modules/apt_module.html)  
[Ansible - "git" module](https://docs.ansible.com/ansible/latest/modules/git_module.html)  
[Ansible - "command" module](https://docs.ansible.com/ansible/latest/modules/command_module.html)  
[Ansible - "slurp" module](https://docs.ansible.com/ansible/latest/modules/slurp_module.html)  
[Ansible - "set_fact" module](https://docs.ansible.com/ansible/latest/modules/set_fact_module.html)  
[Ansible - "authorized_key" module](https://docs.ansible.com/ansible/devel/modules/authorized_key_module.html)  
[Ansible - "parted" module](https://docs.ansible.com/ansible/latest/modules/parted.html)  
[Ansible - "lvg" module](https://docs.ansible.com/ansible/latest/modules/lvg.html)  
[OpenStack-Ansible - Prepare the deployment host](https://docs.openstack.org/project-deploy-guide/openstack-ansible/latest/deploymenthost.html)  
[OpenStack-Ansible - Prepare the target hosts](https://docs.openstack.org/project-deploy-guide/openstack-ansible/latest/targethosts.html)  

## Versions used:

Desktop Machine: *kubuntu-18.04*  
VirtualBox: *virtualbox-5.2.18*  
Vagrant: *2.2.4*  
Cumulus VX Vagrant Box: *CumulusCommunity/cumulus-vx (virtualbox, 3.7.5)*  
Ubuntu Server Vagrant Box: *geerlingguy/ubuntu1804 (virtualbox, 1.0.7)*  
Ansible: *2.7.10*
