---
layout:     post
title:      "OpenStack Lab Network - Switch Configuration (Part 2)"
subtitle:   "Configuring the Interfaces of Cumulus switches with Ansible"
date:       2018-11-23
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

This post is the sixth in a series that plans to document my progress through installing and configuring an OpenStack Lab, using Cumulus switches for the network.

For other posts in this series, see the overview section of the [introduction post](https://www.wadman.co.nz/2018/02/08/OpenStack-Lab-Network-Introduction/#overview).

In the last post, we finished the base configuration of our Cumulus switches using Ansible.
This post will cover the configuration of network interfaces on Cumulus switches.

# Configuration

For me, configuring the interfaces on a network device isn't just enabling the required physical ports.  
I think of this stage as laying the framework that our routing configuration will sit on top of.

What do we need to do before we bring up OSPF or BGP? I can think of a few things:
* Configuration of the loopback interface
* Setting the physical properties of required ports
* Setting the MTU of all interfaces
* Naming/Describing all interfaces
* Configuring IP addresses on interfaces
* Bringing up LLDP

Outside of our topology, you might also include some other steps. For example:
* Creation of VLANs/Bridges
* Configuration of LAGs (Including MCLAG)
* Configuration of STP
* Configuring and testing [Cumulus PTM](https://docs.cumulusnetworks.com/display/DOCS/Prescriptive+Topology+Manager+-+PTM)
* Configuring 802.1X for port authentication

## Ansible Playbook/Role

We've already created the playbook in the previous post, which references the "cumulus-base" role.  
All we need to do now is create a new role for the interface configuration, "cumulus-interface", and reference this in our existing playbook.

Creating the role directory:

```bash
$ mkdir -p /etc/ansible/roles/cumulus-interface/tasks/
```

And adding this to our playbook - "openstack-cumulus.yml":

```yaml
---
- name: Configuring Cumulus switches
  hosts: openstack_cumulus
  become: true
  gather_facts: true
  roles:
    - { role: cumulus-base, tags: [ 'cumulus-because' ] }
    - { role: cumulus-interface, tags: [ 'cumulus-interface' ] }
```

&nbsp; <!--- Used to add a double line break --->

I'd like to make a quick digression here.  
While we could have made only one role, named something like "cumulus-configure", I believe that it is better to try and separate out the functions of roles.

I can think of two examples of why this might be useful
* Say I wanted to include the base configuration of the switches in the ZTP script instead. With separate roles this is as easy as removing the role from the playbook, whereas with a single role you would need to manually prune the tasks in the role.
* In the case that you wanted to deploy an utterly different topology onto the switches, where the difference in interface configuration would be too complex to bundle into a single role, you can simply rename the existing role to "cumulus-interface-openstack" and create a new role for the new topology.  
(It's important to think about the tradeoff you're making with this reduction in complexity because you might end up making more work in the future maintaining two roles instead of just the one.)

## Cumulus Interface Configuration

For network interface configuration, Cumulus again follows closely to how [Debian does it.](https://wiki.debian.org/NetworkConfiguration)

On Debian, network interface configuration is held in the file "/etc/network/interfaces". Optionally, this file can also include ("source" is the appropriate configuration term) configuration files, usually held in the directory "/etc/network/interfaces.d/".  
The Debian network interface management tool "ifupdown" is then run (with this file as an input) to configure the interfaces.

The only difference when it comes to Cumulus is that "ifupdown" is replaced with their own implementation - "[ifupdown2](https://github.com/CumulusNetworks/ifupdown2)".

&nbsp; <!--- Used to add a double line break --->

Let's have a look at the default "/etc/network/interfaces" file on a Cumulus box:

```config
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*.intf

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto eth0
iface eth0 inet dhcp
```

I'll also check whether there are any files already in "/etc/network/interfaces.d/"

```bash
$ ls -alh /etc/network/interfaces.d/
total 0
drwxr-xr-x 1 root root   0 Nov  7 09:56 .
drwxr-xr-x 1 root root 132 Nov 14 07:47 ..
```

Now that we know where to configure our interfaces, let's jump into our first task.

### Loopback Interface Configuration

As you can see in the above section, the loopback interface is already defined by default.  
However, this interface doesn't have an IP address - which is needed for connectivity in our implementation of BGP.  
Let's change that.

In the new role, under "tasks/main.yml", I've created the following task:

```yaml
---
- name: Configure Interfaces
  template:
    src: "interfaces_custom.j2"
    dest: "/etc/network/interfaces.d/interfaces_custom.intf"
  notify:
    - Restart networking
```

The jinja template file ("templates/interfaces_custom.j2") looks like the following:

<!-- {% raw %} -->
```jinja
# {{ ansible_managed }}

######################
# Loopback Interface #
######################
auto lo
iface lo inet loopback
{% if cumulus_host_loopback_address is defined %}
    address {{ cumulus_host_loopback_address }}
{% endif %}
```
<!-- {% endraw %} -->

Because of the above, I'm also going to create the variable "cumulus_host_loopback_address" for all of our Cumulus hosts.  
Because each switch will have a different loopback address, I'm going to create six different host variable files (one for each switch) and create the variable for each.  
Here's what "/etc/ansible/host_vars/cumulus-spine01/vars.yml" looks like as an example:

```yaml
---
cumulus_host_loopback_address: "192.168.11.101/32"
```

Finally, we need to restart the "networking" service on the switches if there are any changes made.  
Here's what my "handlers/main.yml" file looks like:

```yaml
- name: Restart networking
  service:
    name: networking
    state: restarted
```

### Physical Interface Configuration

Physical network configuration is completed in the same way as the loopback, just with different naming and a few more options to consider.  
The ones that we're going to start with are "auto *port*" and "iface *port*", which both need to be present for ifupdown to load a given port.

Because we've already written the task to copy our interfaces template file into the sourced directory, we simply need to append the physical interface configuration to our existing template:

<!-- {% raw %} -->
```jinja
#########################
# Switchport Interfaces #
#########################
{% for port, value in cumulus_switchgroup_switchports.items() | sort %}
auto {{ port }}
iface {{ port }}

{% endfor %}
```
<!-- {% endraw %} -->

If we remember back to when the [Ansible hosts file](https://wadman.co.nz/2018/06/08/OpenStack-Lab-Network-DHCP/#hosts) was created, the groups "openstack_cumulus_spines", "openstack_cumulus_leafgroup1" and "openstack_cumulus_leafgroup2". This was because the switches in each group have something in common - every switch connects to the same devices.  
As an example, the leaf switches 01 and 02 both have one uplink port to each spine switch and one downlink port to the connected OpenStack host.

Before we dive into the logic of the above template, I think it might be better to show you the variable files that I'm creating for each group first.  
Here's what the "/etc/ansible/group_vars/openstack_cumulus_spines/vars.yml" looks like:

```yaml
---
cumulus_switchgroup_switchports:
  swp1:
    alias: downlink-cumulus-leaf01
  swp2:
    alias: downlink-cumulus-leaf02
  swp3:
    alias: downlink-cumulus-leaf03
  swp4:
    alias: downlink-cumulus-leaf04
```

And "/etc/ansible/group_vars/openstack_cumulus_leafgroup1/vars.yml":

```yaml
---
cumulus_switchgroup_switchports:
  swp1:
    alias: uplink-cumulus-spine01
  swp2:
    alias: uplink-cumulus-spine02
  swp3:
    alias: downlink-openstack-control01
```

&nbsp; <!--- Used to add a double line break --->

Now back to the logic of the template.
* For every key (ports/"swp*X*" entries) in the dictionary "cumulus_switchgroup_switchports", and their associated values:
    * First sort them, so that they are being iterated over in numerical order.
    * Write the key string into the file, first after "auto" and again after "iface".

This might look a little more complicated than it needs to be, and you would be right if you thought this way. However, this is because we're going to be expanding on this loop in the next configuration steps.

### MTU

MTU on Cumulus needs to be configured under each interface individually - unless you're okay with using the default of 1500.  
This is set by adding an indented line of "mtu *value*" under the "iface *port*" line for the port.

Now that we have a base template for each switch port, this shouldn't be too hard.  
First, let's create a variable for the MTU, which will be shared across all ports.  
In "/etc/ansible/group_vars/openstack_lab/vars.yml", we'll make the variable "openstack_lab_mtu":

```yaml
...
# Network Interfaces
openstack_lab_mtu: 9000
```

Next is to include it in the template:

<!-- {% raw %} -->
```jinja
#########################
# Switchport Interfaces #
#########################
{% for port, value in cumulus_switchgroup_switchports.items() | sort %}
auto {{ port }}
iface {{ port }}
    mtu {{ openstack_lab_mtu }}
{% endfor %}
```
<!-- {% endraw %} -->

Note that because this is a global variable we didn't need to include this in each switch group's variable file.  
Likewise, the reference in the template is to the global variable and not to either the port or value in the loop.

### Interface Descriptions

Interface descriptions in Debian network configuration are referred to as "aliases", and configured as such.  
Like with the MTU for a port, the alias is configured with another indented line under "iface *port*", this time with "alias *value*".

If you've been observant, you'll notice that we have already created a key named "alias" under each switch port in the switch group variables.  
That way we can use this in our template, which now becomes:

<!-- {% raw %} -->
```jinja
#########################
# Switchport Interfaces #
#########################
{% for port, value in cumulus_switchgroup_switchports.items() | sort %}
auto {{ port }}
iface {{ port }}
    mtu {{ openstack_lab_mtu }}
{% if 'alias' in value %}
{# Description of the interface #}
    alias {{ value.alias }}
{% endif %}

{% endfor %}
```
<!-- {% endraw %} -->

Let's come back to the logic for the loop again and expand on it to cover the alias inclusion.
* For every key (ports/"swp*X*" entries) in the dictionary "cumulus_switchgroup_switchports", and their associated values:
    * First sort them, so that they are being iterated over in numerical order.
    * Write the key string into the file, first after "auto" and again after "iface".
    * Write "mtu" followed by what the variable "openstack_lab_mtu" is set to.
    * If the key has a value inside of it with the name of "alias" then write "alias" followed what the value is set to.

### Interface IP Addresses

The last interface configuration change we'll make is to add an IP address to each interface.  
This won't be just any IP address though, as we're going to reuse the loopback address so that we can take advantage of [OSPF unnumbered](https://docs.cumulusnetworks.com/display/DOCS/Open+Shortest+Path+First+-+OSPF+-+Protocol#OpenShortestPathFirst-OSPF-Protocol-ospf_unnumUnnumberedInterfaces).

I'll cover this in detail in the next post on routing configuration.  

<!-- {% raw %} -->
```jinja
#########################
# Switchport Interfaces #
#########################
{% for port, value in cumulus_switchgroup_switchports.items() | sort %}
auto {{ port }}
iface {{ port }}
    mtu {{ openstack_lab_mtu }}
{% if cumulus_routing_ospf_unnumbered == true %}
    address {{ cumulus_host_loopback_address }}
{% endif %}
{% if 'alias' in value %}
{# Description of the interface #}
    alias {{ value.alias }}
{% endif %}

{% endfor %}
```
<!-- {% endraw %} -->

In "/etc/ansible/group_vars/openstack_cumulus/vars.yml", we'll make the variable "cumulus_routing_ospf_unnumbered" and set this to `True`:

```yaml
---
# Management
cumulus_management_interface: "eth0"

# Routing
cumulus_routing_ospf_unnumbered: True
```

## Configuring LLDP

LLDP on Cumulus is implemented with [lldpd](https://vincentbernat.github.io/lldpd/) and is enabled by default on all interfaces.
Therefore, all I'm going to configure in this step is what interfaces LLDP should *not* run on.  
This is done with the file "/etc/lldpd.d/exclude.conf":

```yaml
- name: Configure LLDP
  template:
    src: lldpexclude.conf.j2
    dest: /etc/lldpd.d/exclude.conf
  notify: Restart LLDP
```

This is a pretty simple file, with just one line of configuration:

<!-- {% raw %} -->
```jinja
# {{ ansible_managed }}
configure system interface pattern-blacklist {{ cumulus_management_interface }}
```
<!-- {% endraw %} -->

And the handler simply restarts the "lldp" service:

```yaml
- name: Restart LLDP
  service:
    name: lldpd
    state: restarted
```

If we were deploying this into a production environment, I would probably want to add [Cumulus PTM](https://docs.cumulusnetworks.com/display/DOCS/Prescriptive+Topology+Manager+-+PTM) configuration to this step as well.  
As this is a lab, where I might be changing connections at any time, I'm going to leave this for now.

# Conclusion

In this post, I covered some basic configuration of interfaces on a Cumulus switch.

In the next post, I'll start to dive into the routing configuration required for our OpenStack deployment.

## References:

[Ansible - "template" module](https://docs.ansible.com/ansible/latest/plugins/lookup/template.html)  
[Ansible - "service" module](https://docs.ansible.com/ansible/latest/modules/service_module.html)  
[Debian Network Configuration](https://wiki.debian.org/NetworkConfiguration)  
[Cumulus Linux - Interface Configuration](https://docs.cumulusnetworks.com/display/DOCS/Interface+Configuration+and+Management)  
[Cumulus Linux - Switchport Attributes](https://docs.cumulusnetworks.com/display/DOCS/Switch+Port+Attributes)  
[Cumulus Linux - OSPF Unnumbered](https://docs.cumulusnetworks.com/display/DOCS/Open+Shortest+Path+First+-+OSPF+-+Protocol#OpenShortestPathFirst-OSPF-Protocol-ospf_unnumUnnumberedInterfaces)  
[Cumulus Linux - LLDP](https://docs.cumulusnetworks.com/display/DOCS/Link+Layer+Discovery+Protocol)   

## Versions used:

Desktop Machine: *kubuntu-18.04*  
VirtualBox: *virtualbox-5.2.10*  
Vagrant: *2.2.1*  
Cumulus VX Vagrant Box: *CumulusCommunity/cumulus-vx (virtualbox, 3.7.2)*  
Ubuntu Server Vagrant Box: *geerlingguy/ubuntu1604 (virtualbox, 1.2.3)*  
Ansible: *2.7.2*
