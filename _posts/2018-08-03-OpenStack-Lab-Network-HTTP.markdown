---
layout:     post
title:      "OpenStack Lab Network - ZTP Server (Part 2)"
subtitle:   "Configuring a HTTP server with Ansible"
date:       2018-08-03
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

This post is the fourth in a series that plans to document my progress through installing and configuring an OpenStack Lab, using Cumulus switches for the network.

For other posts in this series, see the overview section of the [introduction post](https://www.wadman.co.nz/2018/02/08/OpenStack-Lab-Network-Introduction/#overview).

In the last post, we configured the DHCP part of the ZTP server.
This post will cover the installation of the HTTP server part on the ZTP server using Ansible.

# Configuration

Configuring the ZTP server to serve a script via HTTP needs to be done in two parts.

We'll start with getting the server serving files (the ZTP script itself) over HTTP.
Once we verify that is working, we'll configure the contents of the ZTP script itself so that we can tell the switches what to do after booting.

## HTTP Server

### Ansible Playbook

As we already started the playbook for this host last time, I'm just going to add another role.
This role I'll call "nginx-ztp":

```yaml
---
- name: Installing and Configuring ISC's DHCP Server
  hosts: openstack_ztp
  become: true
  gather_facts: true
  roles:
    - { role: isc-dhcp, tags: [ 'isc-dhcp' ] }
    - { role: nginx-ztp, tags: [ 'nginx-ztp' ] }
```

### Ansible Role

Like I noted for DHCP roles in my [last post](https://wadman.co.nz/2018/06/08/OpenStack-Lab-Network-DHCP/#ansible-role), there are a lot of good nginx roles (including [one written by the same author](https://github.com/debops/ansible-nginx)).

However, unlike with DHCP, most of the nginx roles out there are meant as roles used to configure a web server serving a web page to users.  
I feel that in our case, where all we're doing is serving a single file, that it is best if we write our own simple role.  
Plus it makes for good practice and possibly a more interesting blog post as well.

&nbsp; <!--- Used to add a double line break --->

We'll start out by creating directories for the role and its' configuration to live:

```bash
$ mkdir -p /etc/ansible/roles/nginx-ztp/tasks/
```

Within this directory, we'll create the `main.yml` file to begin our role.

&nbsp; <!--- Used to add a double line break --->

To start off, I'll be writing this role with only Ubuntu in mind. Because of this, it's a good habit to start the tasks file with a check of the operating system to ensure that we're not running the role on a host that isn't supported (and so that we don't break stuff).  
We'll accomplish this using the `fail` module in Ansible.

```yaml
- name: Check whether running on a supported operating system
  fail:
    msg: "This role is currently only supported on Ubuntu systems. Sorry!"
  when: ansible_distribution != "Ubuntu"
```

Next up is the installation of the nginx package, which we'll use the `apt` module for

```yaml
- name: Install nginx package
  apt:
    name: nginx
    state: present
    cache_valid_time: 3600
```

&nbsp; <!--- Used to add a double line break --->

After installation, nginx installs a default site configuration at `/etc/nginx/sites-enabled/default`, which we need to disable.  
We're also going to delete the default 'html' directory that nginx creates on installation, as I want to use `/var/www/` as the directory that we serve the ZTP script from and it's best if that is empty.

<!-- {% raw %} -->
```yaml
- name: Remove default nginx site
  file:
    path: "{{ item }}"
    state: absent
  with_items:
    - "/etc/nginx/sites-enabled/default"
    - "/var/www/html"
  notify: Restart nginx
```
<!-- {% endraw %} -->

I've included the notify option at the bottom of this task, as we'll need to reload nginx in order for this to take effect.  
This 'notify' option calls a handler, which we'll define in a new file - `/etc/ansible/roles/nginx-ztp/handlers/main.yml`:

```yaml
---
- name: Restart nginx
  service:
    name: nginx
    state: restarted
```

After doing so, we need to write our own configuration file and tell nginx to use it.  
For the configuration file itself, I'm going to use the `template` module as it's perfect for this type of thing - especially if we want to make changes to the configuration later:

```yaml
- name: Configure nginx site
  template:
    src: nginx-site.j2
    dest: /etc/nginx/sites-available/nginx-site
  notify: Restart nginx
```

This won't work without the template file itself, which we'll create at `/etc/ansible/roles/nginx-ztp/templates/nginx-site.j2`:

<!-- {% raw %} -->
```jinja
server {
  listen {{ ansible_default_ipv4.address }}:80;

  root {{ nginx_root }};
  autoindex on;
}
```
<!-- {% endraw %} -->

Super simple configuration, we're telling nginx to listen on the primary IPv4 IP address and serve files from a directory defined by the variable <!-- {% raw %} -->`{{ nginx_root }}`<!-- {% endraw %} -->.

For those not familiar with nginx, site configuration files are stored in `/etc/nginx/sites-available` and then symlinked into `/etc/nginx/sites-enabled`.  
So all we need to do is create that symlink, using the `file` module again:

```yaml
- name: Symlink nginx site to sites-enabled
  file:
    src: /etc/nginx/sites-available/nginx-site
    dest: /etc/nginx/sites-enabled/nginx-site
    state: link
  notify: Restart nginx
```

We also need to define the variable `nginx_root`, as we've used it in the Jinja template.  
The file we're creating to do so will be `/etc/ansible/roles/nginx-ztp/defaults/main.yml`
I'm using the role defaults to define this because the value we'll set this to (`/var/www`) is the default of nginx.  

```yaml
---
nginx_root: "/var/www/"
```

&nbsp; <!--- Used to add a double line break --->

Before adding our ZTP script, let's run this playbook quickly to ensure everything is working.

```bash
$ ansible-playbook /etc/ansible/playbooks/openstack_ztp.yml
...
PLAY RECAP ****************************************************************
cumulus-ztp                : ok=9   changed=6    unreachable=0    failed=0
```

We could now test the site is running by visiting http&#58;//192.168.11.221/ - this will result in the following page:
<!--- Using the longcode for the colon symbol so that the URL isn't highlighted above --->

![Nginx Blank File Server](/img/nginx-blank-file-server.png)

## ZTP Script

Before we write the ZTP script itself, we'll add a task into the role that will move this into the web server root:

<!-- {% raw %} -->
```yaml
- name: Place ZTP script into web server root
  template:
    src: cumulus-ztp.j2
    dest: "{{ nginx_root }}/cumulus_ztp.sh"
```
<!-- {% endraw %} -->

### ZTP Configuration

The script itself we'll create under the templates directory for the role, just like the nginx configuration.  
Cumulus has some [good documentation](https://docs.cumulusnetworks.com/display/DOCS/Zero+Touch+Provisioning+-+ZTP#ZeroTouchProvisioning-ZTP-WritingZTPScripts) on their website about the writing of ZTP scripts.

It all boils down to the following requirements:
- Include the line `# CUMULUS-AUTOPROVISIONING` anywhere in your script  
- Be written in a language that Cumulus Linux supports (Bash, Perl, Python or Ruby)  
- The script must return an exit code of 0 if successful, so that the switch knows when everything has worked.

Other than the above, the script can house anything that you want.

Common tasks include:
- Installing the licence onto the switch  
- Adding the Debian repositories so that packages outside of the Cumulus ones are installable  
- Installing new packages and upgrading current ones  
- Configuring authentication mechanisms, so that administrators can log into the switch with their LDAP/Radius/TACACs credentials straight away  
- Calling back to configuration automation (i.e. Ansible) so that the full configuration can be loaded

&nbsp; <!--- Used to add a double line break --->

I don't have many requirements in my lab, so I'm simply going to add the Debian repo's and then upgrade the packages on the device to ensure that they're the latest version.

This is what my script (`cumulus-ztp.j2`) looks like:

```bash
#!/bin/bash

# Log all output from this script
exec >> /var/log/autoprovision 2>&1
date "+%FT%T ztp starting script $0"

# Add Debian Repositories
cat <<EOT >> /etc/apt/sources.list.d/debian.list
deb http://deb.debian.org/debian/ jessie main contrib non-free
deb http://deb.debian.org/debian/ jessie-updates main contrib non-free
deb http://deb.debian.org/debian-security jessie/updates main
EOT

# Update apt cache then upgrade packages to their latest version
apt-get update -y
apt-get dist-upgrade -y

# CUMULUS-AUTOPROVISIONING
exit 0
```

### Testing ZTP

After saving the above files, we'll run the playbook again so that our ztp script is now available on the web server.

Next is booting one of the Cumulus switches (`vagrant up cumulus_spine01`) and checking whether everything completes successfully.
The best way to do this is by checking with the ZTP daemon on the switch, using the command `ztp -s`:

```bash
$ ztp -s

ZTP INFO:

State                                 disabled
Version                               1.0
Result                                success
Method                                ZTP DHCP
URL                                   http://192.168.11.221/cumulus_ztp.sh
```

This clearly shows us that ZTP both picked up the correct URL for the script, but also ran this with a successful result.

## Running Ansible Playbooks with Vagrant

The above is all well and good, but if we ran the command `vagrant up` to provision our entire environment from scratch it wouldn't work.  
This is because we aren't running the above playbook on the ZTP guest during this provisioning (between the creation of the ZTP and switch guest VM's).

To solve this we need to enter a `.vm.provision` block into our Vagrantfile, and because we're only doing this on the ZTP guest we'll only place this in the ZTP VM block:

```ruby
##########
# ZTP VM #
##########

# ZTP VM
config.vm.define "openstack_ztp" do |device|
  device.vm.box = "geerlingguy/ubuntu1804"
  device.ssh.host = "192.168.11.221"

  # VirtualBox Config
  device.vm.provider "virtualbox" do |vb|
    vb.name = "ztp-server"
    vb.memory = 1024
    vb.cpus = 1
    vb.customize ["modifyvm", :id, "--macaddress1", "080027000021"]
  end

  # Run the ZTP playbook on the ZTP host
  device.vm.provision :ansible do |ansible|
    ansible.playbook = "/etc/ansible/playbooks/openstack_ztp.yml"
    ansible.inventory_path = "/etc/ansible/hosts"
  end
end
```

The syntax is quite simple;
- `ansible.playbook` is where you tell Vagrant the path to the playbook you want to run.
- `ansible.inventory_path` is needed if you're specifying Ansible variables outside of the Vagrantfile.

## Vagrant Up!

With the above defined, we can finally provision our entire environment.  
The below shows a successful run:

```bash
$ vagrant up
...
==> openstack_ztp: Running provisioner: ansible
...
PLAY RECAP ****************************************************************
cumulus-ztp                : ok=14   changed=9    unreachable=0    failed=0
...
==> openstack_ztp: Machine booted and ready!
...
==> cumulus_spine01: Machine booted and ready!
...
==> cumulus_spine02: Machine booted and ready!
...
==> cumulus_leaf01: Machine booted and ready!
...
==> cumulus_leaf02: Machine booted and ready!
...
==> cumulus_leaf03: Machine booted and ready!
...
==> cumulus_leaf04: Machine booted and ready!
...
==> openstack_control: Machine booted and ready!
...
==> openstack_compute: Machine booted and ready!
...
```

We can test that everything is up and reachable through ansible as well:

```bash
$ ansible -m ping openstack_lab
cumulus-leaf03 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
cumulus-ztp | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
cumulus-leaf04 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
cumulus-leaf01 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
cumulus-leaf02 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
cumulus-spine01 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
cumulus-spine02 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
openstack-compute | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
openstack-control | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

# Conclusion

In this post we've covered how to write a ztp script for Cumulus Linux devices, and the configuration of a (very) simple web server to serve this to the switches as they boot.  
We would now be able to run `vagrant up` and all hosts (including the Ubuntu hosts for OpenStack) would get an address via DHCP and the Cumulus Switches would also be updated on top of that.

In the next post, I'll move onto the base configuration of the Cumulus switches using Ansible.

## References:

[Cumulus Technical Documentation - Zero Touch Provisioning](https://docs.cumulusnetworks.com/display/DOCS/Zero+Touch+Provisioning+-+ZTP)  
[Cumulus Networks Example ZTP Scripts on Github](https://github.com/CumulusNetworks/example-ztp-scripts)  
[Vagrant - Ansible Provisioner](https://www.vagrantup.com/docs/provisioning/ansible.html)

## Versions used:

Desktop Machine: *kubuntu-18.04*
VirtualBox: *virtualbox-5.2.10*
Vagrant: *2.1.2*
Cumulus VX Vagrant Box: *CumulusCommunity/cumulus-vx (virtualbox, 3.6.2)*
Ubuntu Server Vagrant Box: *geerlingguy/ubuntu1804 (virtualbox, 1.0.6)*
Ansible: *2.6.2*
