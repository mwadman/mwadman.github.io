---
layout:     post
title:      "OpenStack Lab Network - Switch Configuration (Part 1)"
subtitle:   "Starting the configuration of Cumulus switches with Ansible"
date:       2018-11-18
author:     "Michael Wadman"
catalog:    true
tags:
    - OpenStack
    - Networking
    - Ansible
    - Cumulus
    - Vagrant
    - VirtualBox
---

# Overview

This post is the fifth in a series that plans to document my progress through installing and configuring a small OpenStack Lab.

For other posts in this series, see the overview section of the [introduction post](https://www.wadman.co.nz/2018/02/08/OpenStack-Lab-Network-Introduction/#overview).

In the last post, we configured the HTTP part of the ZTP server and then showed that we were able to bring everything up with Vagrant.
This post will cover the base configuration of the Cumulus switches using Ansible.

# Configuration

So what constitutes "base" configuration?  
When I think of 'base', I think of everything that is done *before* any network configuration is made:
* Installing any prerequisite/additional packages
* ~~Install the licence~~ (Not relevant for Cumulus VX)
* Configuring DNS (Including hostname)
* Configuring NTP
* Configuring Authentication

The first one is definitely a new one for network devices as most network operating systems wouldn't allow for any packages to be installed, but as the switch is a Linux device I'm going to install some packages that I like.

## Ansible Playbook

We'll start by writing the playbook that we'll call.  
In the directory "/etc/ansible/playbooks/" I'm creating another file, this time with the name "openstack-cumulus.yml":

```yaml
---
- name: Configuring Cumulus switches
  hosts: openstack_cumulus
  become: true
  gather_facts: true
  roles:
    - { role: cumulus-base, tags: [ 'cumulus-because' ] }
```

Super simple. We want to run a single role ("cumulus-base", which we'll create next) on all of our switches to configure them.

## Ansible Role

To start, we'll create the roles "tasks" directory:

```bash
$ mkdir -p /etc/ansible/roles/cumulus-base/tasks/
```

Within this directory, let's make a "main.yml" file and start adding tasks.

### Installing packages

The first thing listed is the installation of packages.  
Cumulus runs on top of Debian Jessie and uses "apt" as its' package manager, so we'll use the `apt` Ansible module for this step.

With the `apt` module, I like to start with a base state of declaring the name of each package and then the state that I want Ansible to keep it at.  
I think this is a good habit to get into as it reminds me to think about whether I want a given package to remain stable or not.

The only other option I'll include is `cache_valid_time` and set this to something a larger than the time between test runs of a playbook (I've found 15 minutes to be a good place to set this) so that the task isn't running every time I test.  

I'm just going to install some troubleshooting packages, "bwm-ng" and "mtr", but because we added the Debian repositories to apt as a part of our ZTP script we could install any others that we saw fit.

<!-- {% raw %} -->
```yaml
---
- name: Install apt packages
  apt:
    name: "{{ item.name }}"
    state: "{{ item.state }}"
    cache_valid_time: 900
  with_items:
    - { name: bwm-ng, state: latest }
    - { name: mtr, state: latest }
  tags:
    - packages
```
<!-- {% endraw %} -->

### Configuring DNS

Because we can skip installing a licence (Cumulus VX doesn't require one), next on our list is DNS configuration.

DNS configuration on a switch means
* Setting DNS Servers
* Setting the hostname of the device

We can (again) skip a step, in setting the DNS servers, this time because our switch receives these via DHCP when it boots.  

To set the hostname with Ansible we use the `hostname` module and pass it the hostname that we've entered into our Ansible "hosts" file (another reason why this is important).

<!-- {% raw %} -->
```yaml
- name: Set hostname
  hostname:
    name: "{{ inventory_hostname }}"
```
<!-- {% endraw %} -->

### Configuring NTP

NTP configuration consists of setting the timezone and then the NTP servers to get the time from.  
On Debian, both of these are set within files and then the appropriate service reloaded.

#### Configuring Timezone

This one is nice and easy as the file we're changing, "/etc/timezone/", consists of a single line that is set to a [*Continent/City*](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones) pair string.  
For me, that's "Pacific/Auckland".

Because this is a text file, we can set this in multiple ways with Ansible.
* `copy` module using the `content` option.
* `template` module and a source jinja template.

As we'll see going forward, the `copy` option would end up being incredibly messy for some of the longer files. This paired with the idea that it might be nice to use the same module throughout all of our configuration tasks means that I'll go with `template` instead.

```yaml
- name: Set Timezone
  template:
    src: timezone.j2
    dest: /etc/timezone
  notify: Reload timezone
```

This does mean that we need to create a few more directories and files for the template and handler.

```bash
$ mkdir -p /etc/ansible/roles/cumulus-base/templates
$ mkdir -p /etc/ansible/roles/cumulus-base/handlers
```

In the "templates" folder we'll create "timezone.j2", which will contain one variable which we'll call "timezone":

<!-- {% raw %} -->
```jinja
{{ timezone }}
```
<!-- {% endraw %} -->

We now need to tell Ansible what this variable should be set to. Because this is a widely applicable variable (not just applicable to an openstack lab) I'm going to put this into "/etc/ansible/group_vars/all/vars.yml" so that all hosts can reference it.  
I'll also add the NTP servers at the same time that we'll reference in the next task below.

```yaml
# NTP
timezone: "Pacific/Auckland"
ntp_servers:
  - "0.nz.pool.ntp.org"
  - "1.nz.pool.ntp.org"
  - "2.nz.pool.ntp.org"
  - "3.nz.pool.ntp.org"
```

In the "handlers" folder we'll create "main.yml" and create a handler that tells the system to reload the timezone if a change is made.  
On Debian, this is done by telling dpkg to reconfigure the "tzdata" package.

```yaml
---
- name: Reload timezone
  command: "dpkg-reconfigure --frontend noninteractive tzdata"
```

#### Configuring NTP Servers

The task to configure NTP servers is going to look very similar to the timezone task, as we're using the template module to modify an existing file and then run a handler if something changes.

```yaml
- name: Configure NTP
  template:
    src: ntp.j2
    dest: /etc/ntp.conf
  notify: Restart NTP
```

For the template, I'm simply going to copy the existing configuration file, remove the extraneous comments, and then add in our NTP servers using a jinja 'for' loop.

<!-- {% raw %} -->
```jinja
# {{ ansible_managed }}
# /etc/ntp.conf, configuration for ntpd; see ntp.conf(5) for help

driftfile /var/lib/ntp/ntp.drift

statistics loopstats peerstats clockstats
filegen loopstats file loopstats type day enable
filegen peerstats file peerstats type day enable
filegen clockstats file clockstats type day enable

# NTP Servers
{% for item in ntp_servers %}
server {{ item }}
{% endfor %}

# By default, exchange time with everybody, but don't allow configuration.
restrict -4 default kod notrap nomodify nopeer noquery
restrict -6 default kod notrap nomodify nopeer noquery

# Local users may interrogate the ntp server more closely.
restrict 127.0.0.1
restrict ::1

# Specify interfaces, don't listen on switch ports
interface listen {{ cumulus_management_interface }}
```
<!-- {% endraw %} -->

On the last line we're telling the switch to listen for NTP only on the management interface.  
We're going to be refering to the management interface again (in later posts), so I'm going to define a variable so I only have to change this in one place if the need arises.

Because the variable is only relevant to the cumulus switches, I'm going to put this into a new file - "/etc/ansible/group_vars/openstack-cumulus/vars.yml":

```yaml
# Management
cumulus_management_interface: "eth0"
```

&nbsp; <!--- Used to add a double line break --->

The handler looks a little simpler for this task though, as we can use the "service" module to restart NTP on the hosts.

```yaml
- name: Restart NTP
  service:
    name: ntp
    state: restarted
```

### Configuring Authentication

Authentication in a production environment should probably include incorporation into the existing authentication service (RADIUS/TACACS/LDAP), but because we're running a lab I'll leave it out for now.

However, Cumulus does include their own command-line-esque tool called "netd" which requires users to be given permission to run commands.  
This is controlled in the configuration file "/etc/netd.conf":

```ini
# Control which users/groups are allowed to run "add", "del",  
# "clear", "abort", and "commit" commands.  
users_with_edit = root, cumulus  
groups_with_edit = netedit

# Control which users/groups are allowed to run "show" commands.  
users_with_show = root, cumulus  
groups_with_show = netshow, netedit
```

Because I might want to play with this in the future, I'll add the user "vagrant" into the "netedit" group using the `user` module in Ansible:

<!-- {% raw %} -->
```yaml
- name: Allow administrator access to NCLU (net commands)
  user:
    name: '{{ ansible_user }}'
    groups: netedit
    append: yes
```
<!-- {% endraw %} -->

Remember that the variable `ansible_user` refers to the user we're using to log into the device with, which we set in an earlier post in the "hosts" inventory file.

# Testing

Now that our role is all put together, let's give this a test run to make sure that everything works as intended:

```bash
$ ansible-playbook /etc/ansible/playbooks/openstack-cumlus.yml
...
PLAY RECAP ****************************************************************
cumulus-leaf01             : ok=8    changed=7    unreachable=0    failed=0
cumulus-leaf02             : ok=8    changed=7    unreachable=0    failed=0
cumulus-leaf03             : ok=8    changed=7    unreachable=0    failed=0
cumulus-leaf04             : ok=8    changed=7    unreachable=0    failed=0
cumulus-spine01            : ok=8    changed=7    unreachable=0    failed=0
cumulus-spine02            : ok=8    changed=7    unreachable=0    failed=0
```

And one more run to make sure that nothing changes the second time:

```bash
$ ansible-playbook /etc/ansible/playbooks/openstack-cumlus.yml
...
PLAY RECAP ****************************************************************
cumulus-leaf01             : ok=6    changed=0    unreachable=0    failed=0
cumulus-leaf02             : ok=6    changed=0    unreachable=0    failed=0
cumulus-leaf03             : ok=6    changed=0    unreachable=0    failed=0
cumulus-leaf04             : ok=6    changed=0    unreachable=0    failed=0
cumulus-spine01            : ok=6    changed=0    unreachable=0    failed=0
cumulus-spine02            : ok=6    changed=0    unreachable=0    failed=0
```

## Running with Vagrant

Now that we've got a playbook written we can incorporate this into Vagrant.  
The aim is that this will be run when `vagrant up` is called, just like the ZTP playbook in the [last post](https://wadman.co.nz/2018/08/03/OpenStack-Lab-Network-HTTP/).  
To recap, this simply requires the `.vm.provision` block somewhere in our Vagrantfile.

However, unlike our previous post, we are running this playbook against multiple guest machines instead of just the one, so we end up with two options to have the playbook run:
* Include the `.vm.provision` block each individual Cumulus guest block, and use the [limit](https://www.vagrantup.com/docs/provisioning/ansible_common.html#limit) option that the provisioner provides. This would run the playbook once for every machine specified and only against that one machine.
* Include the `.vm.provision` block only once, in the last Cumulus guest block.

I'm going to go with the second option as I believe this to be a cleaner way of completing this.  
However, I can see that you might want to run this individually if there was a requirement to have one device fully configured before bringing up the next.

&nbsp; <!--- Used to add a double line break --->

This is what the Vagrantfile ends up looking like. I've included a skeleton of the last Cumulus machine's configuration to illustrate the provisioners' position in the configuration hierachy.

```ruby
# Leaf 4
config.vm.define "cumulus_leaf04" do |device|
  ...
  # Ansible Playbook for all Cumulus switches
  device.vm.provision :ansible do |ansible|
    ansible.playbook = "/etc/ansible/playbooks/openstack-cumulus.yml"
    ansible.inventory_path = "/etc/ansible/hosts"
    ansible.limit = "openstack_cumulus"
  end
end
```

The only thing to note here is that we also need to use `ansible.limit` to specify all of the hosts we want to run against, as otherwise Vagrant will tell Ansible to only run this against the hosts which are in the same block; which in this case would be just "cumulus_leaf04".

# Conclusion

In this post I started an Ansible playbook to configure some basic functions on Cumulus hosts and covered how to include this in our Vagrantfile.

In the next post, I'll dive into the network interface configuration of the switches, including LLDP and PTMD (I'll cover what PTMD is as well) configuration on Cumulus.

## References:

[Ansible - "apt" module](https://docs.ansible.com/ansible/latest/modules/apt_module.html)  
[Ansible - "hostname" module](https://docs.ansible.com/ansible/latest/modules/hostname_module.html)  
[Ansible - "template" module](https://docs.ansible.com/ansible/latest/plugins/lookup/template.html)  
[Ansible - "command" module](https://docs.ansible.com/ansible/latest/modules/command_module.html)  
[Ansible - "service" module](https://docs.ansible.com/ansible/latest/modules/service_module.html)  
[Ansible - "user" module](https://docs.ansible.com/ansible/latest/modules/user_module.html)  
[Vagrant - Ansible Provisioner](https://www.vagrantup.com/docs/provisioning/ansible.html)  
[Cumulus Linux - Setting the hostname](https://docs.cumulusnetworks.com/display/DOCS/Quick+Start+Guide#QuickStartGuide-ConfiguretheHostnameandTimezone)  
[Cumulus Linux - Setting Date and Time](https://docs.cumulusnetworks.com/display/DOCS/Setting+Date+and+Time)  
[Cumulus Linux - NCLU, Editing netd.conf](https://docs.cumulusnetworks.com/display/DOCS/Network+Command+Line+Utility+-+NCLU#NetworkCommandLineUtility-NCLU-Editthenetd.confFile)  

## Versions used:

Desktop Machine: *kubuntu-18.04*  
VirtualBox: *virtualbox-5.2.10*  
Vagrant: *2.2.1*  
Cumulus VX Vagrant Box: *CumulusCommunity/cumulus-vx (virtualbox, 3.7.2)*    
Ansible: *2.7.2*
