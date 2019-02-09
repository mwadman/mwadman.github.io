---
layout:     post
title:      "OpenStack Lab Network - ZTP Server (Part 1)"
subtitle:   "Configuring a DHCP server with Ansible"
date:       2018-06-08
author:     "Michael Wadman"
catalog:    true
tags:
    - OpenStack
    - Ansible
    - Cumulus
    - Vagrant
    - VirtualBox
    - ZTP
---

# Overview

This post is the third in a series that plans to document my progress through installing and configuring an OpenStack Lab, using Cumulus switches for the network.

For other posts in this series, see the overview section of the [introduction post](https://www.wadman.co.nz/2018/02/08/OpenStack-Lab-Network-Introduction/#overview).

In the last post, we covered how to provision the switches and servers using Vagrant.
This post will cover the installation of the DHCP part on the ZTP server using Ansible.

# Setup

The first thing we need to do before we can configure the ZTP server is give it an IP address.
This would usually be handled by Vagrant, but like [I covered in the last post](https://wadman.co.nz/2018/04/08/OpenStack-Lab-Network-Vagrant/#networking), this isn't possible when we set the first interface to be bridged.

To do this we need to boot the machine, using the following command:

```bash
$ vagrant up openstack_ztp
```

## Setting a Static IP address

Give it a minute or two and then connect to this through the VirtualBox.
To login, use the username/password of "vagrant/vagrant" (which is also the default for all Vagrant boxes).

There is just one place that we need to change the configuration; under "/etc/network/interfaces", change the section below:

```conf
# The primary network interface

auto enp0s3
iface enp0s3 inet static
  address 192.168.11.221
  netmask 255.255.255.0
  gateway 192.168.11.1
  dns-nameservers 1.1.1.1 1.0.0.1
```

Once completed, we can reset the interface so that it picks up the new address:

```bash
$ sudo ifdown enp0s3 && sudo ifup enp0s3
```

Let's verify that the new address has been picked up before we proceed.

```bash
$ ip address show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:80:ff:83 brd ff:ff:ff:ff:ff:ff
    inet 192.168.11.221/24 brd 192.168.11.255 scope global enp0s3
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe80:ff83/64 scope link
       valid_lft forever preferred_lft forever
```

## Connecting Vagrant

Meanwhile, back in the terminal session that you launched the "vagrant up" command in, you should see a message like the below:

```text
Timed out while waiting for the machine to boot. This means that
Vagrant was unable to communicate with the guest machine within
the configured ("config.vm.boot_timeout" value) time period.
```

This simply means that we took too long configuring the IP address of the machine, and Vagrant timed out trying to connect to it.
To remedy this, simply shut down the machine and then run the same `vagrant up` command.

This should result in a happier looking output and a successful exit from the command.

```text
...
==> openstack_ztp: Machine booted and ready!
...
```

For one last test, we'll ssh into this from the host machine:

```bash
$ ssh 192.168.11.221 -l vagrant
vagrant@192.168.11.221s password:
Welcome to Ubuntu 18.04.1 LTS (GNU/Linux 4.15.0-29-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage
vagrant@vagrant:~$ exit
```

## Installing Ansible

We're going to use Ansible to configure the ZTP server (and all other guests).

In order to install this on Ubuntu, we need to add the official repository to our apt sources and then add the package itself:

> There is an ansible package in the default repositories, but this isn't kept up to date

```bash
$ sudo apt-get install software-properties-common
$ sudo apt-add-repository ppa:ansible/ansible
$ sudo apt-get update
$ sudo apt-get install ansible
```

Once installed, we can run the below command to ensure this installed correctly and to confirm the version is up to date

```bash
$ ansible --version | head -n 1
ansible 2.6.2
```

A fresh install of Ansible will create the configuration directory "/etc/ansible", which includes the files "ansible.cfg" and "hosts".
I won't go into detail as to what these files do in this post.

> If you are curious as to learning more about Ansible, you can always check out the slides that I created for a presentation [here](https://www.wadman.co.nz/2018/03/14/Ansible-Slides/)

We need to own these files ourselves if we want to do anything with them:

```bash
$ sudo chown -R $USER:$USER /etc/ansible/
```

## Ansible Setup

### ansible.cfg

We're going to make just a few changes to the defaults in "ansible.cfg".
I've simply uncommented/changed the following lines to look like the below:

```config
...  
forks = 20 # Increases the simultaneous connections Ansible can make

...  
nocows = 1 # Turns off cowsay during ansible runs

...  
retry_files_enabled = False # Disables the creation of retry files

...  
pipelining = True # Uses the same SSH session for multiple tasks on the same host

...
```

### hosts

In the hosts file, we're going to wipe everything already present and create the entries for all of our guests in the OpenStack lab now, so that we don't need to come back and touch this file again:

```config
#################
# OpenStack Lab #
#################

[openstack_lab:children]
openstack_cumulus
openstack_ztp
openstack_hosts

[openstack_lab:vars]
ansible_user="vagrant"
ansible_ssh_pass="vagrant"
ansible_ssh_common_args="-o StrictHostKeyChecking=no"
ansible_ssh_extra_args="-o StrictHostKeyChecking=no"

# Cumulus Switches
[openstack_cumulus:children]
openstack_cumulus_spines
openstack_cumulus_leafs

[openstack_cumulus_spines]
openstack-cumulus-spine01 ansible_host=192.168.11.201
openstack-cumulus-spine02 ansible_host=192.168.11.202

[openstack_cumulus_leafs:children]
openstack_cumulus_leafgroup1
openstack_cumulus_leafgroup2

[openstack_cumulus_leafgroup1]
openstack-cumulus-leaf01 ansible_host=192.168.11.211
openstack-cumulus-leaf02 ansible_host=192.168.11.212

[openstack_cumulus_leafgroup2]
openstack-cumulus-leaf03 ansible_host=192.168.11.213
openstack-cumulus-leaf04 ansible_host=192.168.11.214

# ZTP Server
[openstack_ztp]
cumulus-ztp ansible_host=192.168.11.221

# OpenStack Hosts
[openstack_hosts]
openstack-control ansible_host=192.168.11.231
openstack-compute ansible_host=192.168.11.232
```

In the above we define a parent group, `[openstack_lab]`, and assign some variables to it so that ansible knows how to log in.

Next, we define children groups so that we can run playbooks against certain subsets of hosts.  
For example, we don't want the configuration we apply to the switches to be applied to the ZTP server or vice-versa.

With the above defined, we should be able to test connectivity using the ansible ping module:

```bash
$ ansible -m ping openstack_ztp
cumulus-ztp | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

# Configuration

## DHCP

### Ansible Playbook

We'll start off by writing the playbook that we'll call to apply the configuration to the host.

Firstly, because I like to organise all of my playbooks into a directory structure, I'm going to create the "/etc/ansible/playbooks" directory.
Under this directory I'm creating a new file named "openstack_ztp.yml", with the following content:

```yaml
---
- name: Installing and Configuring ISC's DHCP Server
  hosts: openstack_ztp
  become: true
  gather_facts: true
  roles:
    - { role: isc-dhcp, tags: [ 'isc-dhcp' ] }
```

I've named the `role: isc-dhcp`, as we're going to be implementing ZTP (and DHCP) using the isc-dhcp-server package on Ubuntu.

### Ansible Role

There are already [a ton of good roles out there](https://galaxy.ansible.com/search?keywords=dhcp%20isc&amp;order_by=-relevance&amp;page_size=10)  for installing isc-dhcp-server. The [DebOps dhcpd role](https://github.com/debops/ansible-dhcpd) is a great example, so I'll clone this as a [submodule](https://chrisjean.com/git-submodules-adding-using-removing-and-updating/) into my local repository:

```bash
/etc/ansible$ git submodule add https://github.com/debops/ansible-dhcpd.git roles/isc-dhcp
```

Most likely your "/etc/ansible" directory isn't a git repository, in which case just copy the files from the repository instead:

```bash
$ git clone --depth=1 https://github.com/debops/ansible-dhcpd.git /etc/ansible/roles/isc-dhcp && rm -rf !$/.git
```

Before we proceed, we need to remove the dependency of this role on the "debops.secret" role, as we're not going to be using functionality that requires it.
In the file "/etc/ansible/roles/isc-dhcp/meta/main.yml", change the "dependencies" section to look like the below:

```yaml
dependencies: []
#  - role: debops.secret
```

Now we can run the playbook and have Ansible configure our DHCP server:

```bash
$ ansible-playbook /etc/ansible/playbooks/openstack_ztp.yml
...
PLAY RECAP
***************************************************************************
cumulus-ztp                : ok=6    changed=4    unreachable=0    failed=0
```

This results in the following configuration on our ZTP server:

```config
$ cat /etc/dhcp/dhcpd.conf
# Ansible managed

not authoritative;

default-lease-time 64800;
max-lease-time 86400;

log-facility local7;

option domain-name "vm";

option domain-search "vm";
option dhcp6.domain-search "vm";

option domain-name-servers 192.168.20.11, 192.168.20.10;

# Generated automatically by Ansible

subnet 192.168.11.0 netmask 255.255.255.0 {
        option routers 192.168.11.1;
}
```

This is great because a subnet has already been created, but our hosts still won't get an address (or the switches able to ZTP themselves).
For that, we need to set some variables.

### Ansible Variables

As per the [documentation](https://github.com/debops/ansible-dhcpd/blob/master/docs/defaults-configuration.rst#id7) on the DebOps dhcpd role, we can either configure the subnet with a pool or set host entries.  
Since I prefer the hosts route in this scenario, I'm going to create an Ansible variable named "dhcpd_hosts" and define each of our hosts.  
I'll put this variable into the openstack_lab group variables file "/etc/ansible/group_vars/openstack_lab/vars.yml".

An example host looks like the following:

```yaml
---
dhcpd_hosts:
  - hostname: cumulus-spine01
    address: '192.168.11.201'
    ethernet: '08:00:27:00:00:01'
```

Note that I'm setting the MAC address as per the static entries that were set in the Vagrantfile in my previous post.

&nbsp; <!--- Used to add a double line break --->

With this set for all of our hosts, each of them will now get an IP address on booting.
That isn't good enough for our switches if we want to ZTP them.

Cumulus has [good documentation](https://docs.cumulusnetworks.com/display/DOCS/Zero+Touch+Provisioning+-+ZTP#ZeroTouchProvisioning-ZTP-ConfiguringtheDHCPServer) on how to configure ZTP.  
According to the page linked, we simply need to define DHCP option 239 (what the switches request when they ZTP boot), give this option a value and then associate it with the appropriate hosts.

In terms of variables in Ansible, this is pretty simple.  
Again, we'll use the variable names that the dhcpd role is expecting, which in this case are "dhcpd_options" and the "options" key under each host entry in the "dhcpd_hosts" dictionary.

With these defined (and a little bit of variable-ization), our file ends up looking like the following:

<!-- {% raw %} -->
```yaml
---
dhcpd_options: "{{ ztp_option_name }} code 239 = text;"
dhcpd_hosts:
  - hostname: cumulus-spine01
    address: '192.168.11.201'
    ethernet: '08:00:27:00:00:01'
    options: '{{ ztp_option_name }} "{{ ztp_url }}";'
  - hostname: cumulus-spine02
    address: '192.168.11.202'
    ethernet: '08:00:27:00:00:02'
    options: '{{ ztp_option_name }} "{{ ztp_url }}";'
  - hostname: cumulus-leaf01
    address: '192.168.11.211'
    ethernet: '08:00:27:00:00:11'
    options: '{{ ztp_option_name }} "{{ ztp_url }}";'
  - hostname: cumulus-leaf02
    address: '192.168.11.212'
    ethernet: '08:00:27:00:00:12'
    options: '{{ ztp_option_name }} "{{ ztp_url }}";'
  - hostname: cumulus-leaf03
    address: '192.168.11.213'
    ethernet: '08:00:27:00:00:13'
    options: '{{ ztp_option_name }} "{{ ztp_url }}";'
  - hostname: cumulus-leaf04
    address: '192.168.11.214'
    ethernet: '08:00:27:00:00:14'
    options: '{{ ztp_option_name }} "{{ ztp_url }}";'
  - hostname: openstack-control
    address: '192.168.11.231'
    ethernet: '08:00:27:00:00:31'
  - hostname: openstack-compute
    address: '192.168.11.232'
    ethernet: '08:00:27:00:00:32'

ztp_option_name: "option cumulus-provision-url"
ztp_url: "http://{{ ansible_default_ipv4.address }}/{{ ztp_filename }}"
ztp_filename: "cumulus_ztp.sh"
```
<!-- {% endraw %}) -->

Which results in the following configuration on our ZTP host:

```conf
$ cat /etc/dhcp/dhcpd.conf
# Ansible managed

not authoritative;

default-lease-time 64800;
max-lease-time 86400;

log-facility local7;

option domain-name "vm";

option domain-search "vm";
option dhcp6.domain-search "vm";

option domain-name-servers 192.168.20.10;

# Configuration options
option cumulus-provision-url code 239 = text;

# Generated automatically by Ansible
subnet 192.168.11.0 netmask 255.255.255.0 {
        option routers 192.168.11.1;
}

host cumulus-spine01 {
        option cumulus-provision-url "http://192.168.11.221/cumulus_ztp.sh";
        hardware ethernet 08:00:27:00:00:01;
        fixed-address 192.168.11.201;
}
host cumulus-spine02 {
        option cumulus-provision-url "http://192.168.11.221/cumulus_ztp.sh";
        hardware ethernet 08:00:27:00:00:02;
        fixed-address 192.168.11.202;
}
host cumulus-leaf01 {
        option cumulus-provision-url "http://192.168.11.221/cumulus_ztp.sh";
        hardware ethernet 08:00:27:00:00:11;
        fixed-address 192.168.11.211;
}
host cumulus-leaf02 {
        option cumulus-provision-url "http://192.168.11.221/cumulus_ztp.sh";
        hardware ethernet 08:00:27:00:00:12;
        fixed-address 192.168.11.212;
}
host cumulus-leaf03 {
        option cumulus-provision-url "http://192.168.11.221/cumulus_ztp.sh";
        hardware ethernet 08:00:27:00:00:13;
        fixed-address 192.168.11.213;
}
host cumulus-leaf04 {
        option cumulus-provision-url "http://192.168.11.221/cumulus_ztp.sh";
        hardware ethernet 08:00:27:00:00:14;
        fixed-address 192.168.11.214;
}
host openstack-control {
        hardware ethernet 08:00:27:00:00:31;
        fixed-address 192.168.11.231;
}
host openstack-compute {
        hardware ethernet 08:00:27:00:00:32;
        fixed-address 192.168.11.232;
}
```

# Conclusion

We've now booted, set an IP, and configured DHCP on the ZTP server.

We could boot our OpenStack servers now and they would be ready to configure themselves using Ansible.
If we booted our Cumulus switches, they would get an IP address and know where to look for their ZTP script - but wouldn't find any script file at the location http://192.168.11.221/cumulus_ztp.sh.

In the next post, I'll cover the configuration of NGINX to serve this file to the switches so that we can finally boot them fully.

## References:

[DebOps dhcpd Ansible Role](https://github.com/debops/ansible-dhcpd)  
[Chris Jean's post about Git Submodules](https://chrisjean.com/git-submodules-adding-using-removing-and-updating/)  
[Cumulus Technical Documentation - Zero Touch Provisioning](https://docs.cumulusnetworks.com/display/DOCS/Zero+Touch+Provisioning+-+ZTP)  

## Versions used:
Desktop Machine: *kubuntu-18.04*  
VirtualBox: *virtualbox-5.2.10*  
Vagrant: *2.1.2*  
Cumulus VX Vagrant Box: *CumulusCommunity/cumulus-vx (virtualbox, 3.6.2)*  
Ubuntu Server Vagrant Box: *geerlingguy/ubuntu1804 (virtualbox, 1.0.6)*  
Ansible: *2.6.2*  
