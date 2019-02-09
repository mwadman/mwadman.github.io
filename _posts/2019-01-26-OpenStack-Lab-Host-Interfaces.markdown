---
layout:     post
title:      "OpenStack Lab Network - Host Network Interfaces"
subtitle:   "Configuring network interfaces on OpenStack hosts"
date:       2019-01-26
author:     "Michael Wadman"
catalog:    true
tags:
    - OpenStack
    - Networking
    - Ansible
    - Vagrant
    - VirtualBox
    - Ubuntu
---

# Overview

This post is the eighth in a series that plans to document my progress through installing and configuring an OpenStack Lab, using Cumulus switches for the network.

For other posts in this series, see the overview section of the [introduction post](https://www.wadman.co.nz/2018/02/08/OpenStack-Lab-Network-Introduction/#overview).

In the last post, we covered the routing configuration on our Cumulus switches.  
In this post, we'll go over the network interface configuration that our OpenStack hosts require in order to participate in the network that we have set up for them.

# Configuration

In terms of networking on our OpenStack hosts, we want to accomplish a few things before we even think about installing and running OpenStack itself.  
The first, and what we'll cover in this post, will be configuring our network interfaces to support our routing protocols and our OpenStack configuration.

At this point, it might be important to remind ourselves that we're preparing our hosts for an OpenStack deployment via OpenStack-Ansible.
Their [deployment guide](https://docs.openstack.org/project-deploy-guide/openstack-ansible/latest/) contains tons of useful information when going through a first install.  
Also of note for this series of posts is their [routed environment example](https://docs.openstack.org/openstack-ansible/latest/user/l3pods/example.html), as the configuration involved and the end state of the hosts is similar to what we want to deploy.

## Ansible Playbook

As usual, our starting place is the playbook.  
Because we're no longer working with our Cumulus switches and instead on our OpenStack hosts themselves, let's separate out any roles that we use into a new playbook. It'll be called "openstack-hosts.yml"

```yaml
---
- name: Configuring OpenStack hosts
  hosts: openstack_hosts
  become: true
  gather_facts: true
  roles:
    - { role: openstack-interfaces, tags: [ 'openstack-interfaces' ] }
```

## Ansible Role

I've already spoiled the name of the role we'll create in the playbook above.  
I'll create the directory structure for it as usual.

```bash
$ mkdir -p /etc/ansible/roles/openstack-interfaces/tasks/
```

Let's start adding tasks.

## Packages

Although the OpenStack-Ansible documentation includes a [list of required packages](https://docs.openstack.org/project-deploy-guide/openstack-ansible/latest/targethosts.html#configure-ubuntu), we'll not be installing all of them in this role.  
In fact, the only package we are going to include from the documentation is "bridge-utils", which allows the configuration of network bridges.  

We've gone over installing packages in previous posts, so I won't go into too much detail now.  
Here's the first task in our file "/etc/ansible/roles/openstack-interfaces/tasks/main.yml"

<!-- {% raw %} -->
```yaml
- name: Install prerequisite packages
  apt:
    name: bridge-utils
    state: present
    cache_valid_time: 3600
```
<!-- {% endraw %} -->

Let's move on to our configuration.

## Interfaces

Just like our Cumulus switches, network interfaces on Ubuntu are configured via the file "/etc/network/interfaces" and any included files in the directory "/etc/network/interfaces.d/".
If you want to know more information about these files (such as their defaults), I went over them briefly in the post on [Cumulus network interface configuration](https://wadman.co.nz/2018/11/23/OpenStack-Lab-Network-Switch-Interfaces/#cumulus-interface-configuration).  
Our task for the configuration of these interfaces is going to look exactly the same as that in the previous post:

```yaml
- name: Configure Interfaces
  template:
    src: "interfaces_custom.j2"
    dest: "/etc/network/interfaces.d/interfaces_custom.intf"
  notify:
    - Restart networking
```

This required the presence of a handler file (“handlers/main.yml”):

```yaml
---
- name: Restart networking
  service:
    name: networking
    state: restarted
```

I'll cover the content of the template file in two parts, starting with the configuration of the loopback and physical interfaces on our hosts.

### Loopback and Physical Interfaces

As with our Cumulus switches, our loopback and physical ports are there to support the routing topology over which OpenStack will communicate.  
Our requirements for configuration will also remain the same, with our loopback interface needing an IP address so that BGP peering relationships can be established; and our physical interfaces using the same address for OSPF unnumbered (which I covered in my [previous post](https://wadman.co.nz/2018/12/09/OpenStack-Lab-Network-Switch-Routing/#ospf-unnumbered)).  
Because of the above, it'll be no surprise that the template for these interfaces looks very similar to what we've already written for our switches.

&nbsp; <!--- Used to add a double line break --->

I'm going to mix it up this time and define our variables before diving into our template.  
We're going to want to define two items as variables for both hosts - our loopback IP addresses and physical interface numbers.  

Because our hosts share the same physical port layout (through our [Vagrant/VirtualBox configuration](https://wadman.co.nz/2018/04/08/OpenStack-Lab-Network-Vagrant/#networking)), they both end up with the same network interface names for their physical ports (enp0s8 and enp0s9). We can, therefore, use the same variable for both of our hosts.  
I'll create this variable in the new file "/etc/ansible/group_vars/openstack_hosts/vars.yml":

```yaml
---
openstack_hosts_network_interfaces:
  - enp0s8
  - enp0s9

# OSPF Routing
openstack_hosts_routing_ospf_unnumbered: True
openstack_hosts_routing_ospf_enabled: True

# BGP Routing
openstack_hosts_routing_bgp_enabled: True
openstack_hosts_routing_bgp_evpn_enabled: True
```

You'll see that I've also created some other variables that we used in our Cumulus interface and routing configurations. These let us say whether we're using a given protocol on our hosts.

We also need to define the IP address that each host will use on its' loopback interface.  
Here’s what “/etc/ansible/host_vars/openstack-compute/vars.yml” looks like as an example:

```yaml
---
openstack_host_loopback_address: "192.168.11.132/32"
```

&nbsp; <!--- Used to add a double line break --->

Now that we've set ourselves up with some variables, let's begin writing our template (“templates/interfaces_custom.j2”):

<!-- {% raw %} -->
```jinja
#######################
# Loopback Interfaces #
#######################

auto lo
iface lo inet loopback
{% if openstack_host_loopback_address is defined %}
    up   ip addr add {{ openstack_host_loopback_address }} dev $IFACE label $IFACE:1
{% endif %}

#######################
# Physical Interfaces #
#######################
{% for port in openstack_hosts_network_interfaces | sort %}
auto {{ port }}
iface {{ port }} inet static
{% if openstack_hosts_routing_ospf_unnumbered == true %}
    address {{ openstack_host_loopback_address }}
{% endif %}
    mtu {{ openstack_lab_mtu }}

{% endfor %}
```
<!-- {% endraw %} -->

Again, this looks just like the interface configuration on our switches.  
The only difference is in how we've configured the loopback interfaces IP address, opting to use the recommended [iproute2 syntax](https://wiki.debian.org/NetworkConfiguration#iproute2_method) for adding an additional IP address to an interface.  

Here we're defining a sub-interface (`lo:1`) using the reference `$IFACE`, which "ifupdown" is smart enough to translate to the name of the interface we're configuring, and adding the IP address to that interface instead.  
We've needed to do this due to the difference in behaviour between the implementation of the tool "ifupdown" on Debian and Cumulus' implementation ("ifupdown2").

### OpenStack Bridges

As I eluded to before, I'm also going to include in this template the configuration of the network bridges required by OpenStack-Ansible.  
This is simply so that we don't need to configure this file again in another role.

OpenStack-Ansible utilises containers to operate the many services that are included in an OpenStack installation. To connect to the outside world, the containers need to talk via bridges that need to be set up on the host before deployment.  
These bridges are as follows:
- br-mgmt: Provides management of and communication between the infrastructure and OpenStack services.
- br-storage: Provides access between OpenStack services and storage devices.
- br-vlan: Provides communication between tenant networks that are VLAN tagged or flat (no VLAN tag).
- br-vxlan: Only required if the environment is configured create virtual networks using VXLAN. Provides communication between tenant networks that use VXLAN tunnelling.

[The documentation](https://docs.openstack.org/project-deploy-guide/openstack-ansible/rocky/targethosts.html#configuring-the-network) helps out here, by showing which bridges need to be present on a given host.  
For our lab deployment, both hosts are going to be acting as network (control), compute and storage nodes.  
This translates to needing all 4 bridges (br-mgmt, br-storage, br-vxlan and br-vlan) on both hosts, with IP addresses on all except br-vlan.

&nbsp; <!--- Used to add a double line break --->

Like in our above step, let's define variables first.  
Because the IP addresses that we configure on these bridges are going to be different on each host, let's define the variables under the "host_vars" hierarchy.  
Here’s what I've added to “/etc/ansible/host_vars/openstack-compute/vars.yml” as an example:

```yaml
openstack_host_network_bridges:
  mgmt: "172.16.0.132/24"
  storage: "172.16.1.132/24"
  vxlan: "172.16.2.132/24"
  vlan:
```

Note the blank value for the "vlan" key, because that bridge will not have an IP address configured.

&nbsp; <!--- Used to add a double line break --->

These variables come together to form the below addition to our interface configuration template:

<!-- {% raw %} -->
```jinja
###########################
# Local Bridge Interfaces #
###########################
{% for bridge, address in openstack_host_network_bridges.items() | sort %}
auto {{ bridge }}
{% if address is none %}
iface {{ bridge }} inet manual
{% else %}
iface {{ bridge }} inet static
{% endif %}
    bridge_stp off
    bridge_waitport 0
    bridge_fd 0
    bridge_ports none
{% if address is not none %}
    address {{ address | ipaddr('address') }}
    netmask {{ address | ipaddr('netmask') }}
{% endif %}

{% endfor %}
```
<!-- {% endraw %} -->

Most of this configuration was taken from [OpenStack-Ansible's examples](https://docs.openstack.org/openstack-ansible/latest/user/l3pods/example.html#host-network-configuration), with some reference to the Debian syntax for [configuration of bridges](https://wiki.debian.org/BridgeNetworkConnections).

Let's walk through the configuration of each bridge:  
<!-- {% raw %} -->
- `auto {{ bridge }}` tells ifupdown to create the interface on startup.
- Depending on whether the bridge has an IP address set (which we test with `{% if address is none %}`), the interface is configured with `manual` or `static`.
- We turn spanning tree off on the interface with `bridge_stp off` because it isn't uplinking to any other switches (rather, this bridge will be advertised as a prefix by iBGP for reachability between hosts).
- `bridge_waitport 0` tells ifupdown to bring up the interface immediately and `bridge_fd 0` says that there is to be no forwarding delay.
- `bridge_ports none` is an interesting one. Usually we would assign network interfaces that are participating in this bridge here, but instead we say that there are none. OpenStack-Ansible is going to add ports to this bridge when it creates the appropriate containers.
- If the bridge has an IP address specified, we set this using the standard syntax.
<!-- {% endraw %} -->

As simple as that, we've configured all prerequisite network interfaces.

## Vagrant

The last thing we need to do is tell Vagrant that it needs to run the playbook after it has set up both OpenStack hosts.  
I'll accomplish this by including the provision block after the last OpenStack host, just like we did with the [Cumulus provisioning block](https://wadman.co.nz/2018/11/18/OpenStack-Lab-Network-Switch-Base/#running-with-vagrant).

```ruby
config.vm.define "openstack_compute" do |device|
  ...
  # Ansible Playbook for all OpenStack hosts
  device.vm.provision :ansible do |ansible|
    ansible.inventory_path = "/etc/ansible/hosts"
    ansible.playbook = "/etc/ansible/playbooks/openstack-hosts.yml"
    ansible.limit = "openstack_hosts"
  end
end
```

# Testing

After all of our configuration, when we run `vagrant up` on our local machine, we get the following output after it has created the two OpenStack hosts:

```bash
$ vagrant up
...
==> openstack_compute: Running provisioner: ansible...
Vagrant has automatically selected the compatibility mode '2.0'
according to the Ansible version installed (2.7.6).

Alternatively, the compatibility mode can be specified in your Vagrantfile:
https://www.vagrantup.com/docs/provisioning/ansible_common.html#compatibility_mode

    openstack_compute: Running ansible-playbook...

PLAY [Configuring OpenStack hosts] *********************************************

TASK [Gathering Facts] *********************************************************
ok: [openstack-control]
ok: [openstack-compute]

TASK [openstack-interfaces : Install prerequisite packages] ********************
changed: [openstack-compute]
changed: [openstack-control]

TASK [openstack-interfaces : Configure Interfaces] *****************************
changed: [openstack-control]
changed: [openstack-compute]

RUNNING HANDLER [openstack-interfaces : Restart networking] ********************
changed: [openstack-compute]
changed: [openstack-control]

PLAY RECAP *********************************************************************
openstack-compute          : ok=1    changed=3    unreachable=0    failed=0   
openstack-control          : ok=1    changed=3    unreachable=0    failed=0
```

Looks like everything was configured, let's check that it's taken effect on the hosts.

## Interfaces

Using `vagrant ssh $host` we can connect to both of our hosts and find out whether everything looks as it should.

Firstly, let's check whether our bridges exist:

```bash
$ brctl show
bridge name   bridge id           STP enabled   interfaces
mgmt          8000.000000000000   no		
storage       8000.000000000000   no		
vlan          8000.000000000000   no		
vxlan         8000.000000000000   no
```

Although the command doesn't show a lot of information (yet - `brctl` becomes a very useful command once MAC addresses are being learnt by each bridge), it does confirm that we've created the bridges on the host.

We can confirm this with the command `ip address show`:

```bash
$ ip address show | grep "UP\|inet" | grep -v "inet6"
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    inet 127.0.0.1/8 scope host lo
    inet 192.168.11.132/32 brd 192.168.11.132 scope global lo:1
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    inet 192.168.11.232/24 brd 192.168.11.255 scope global enp0s3
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc pfifo_fast state UP group default qlen 1000
    inet 192.168.11.132/32 brd 192.168.11.132 scope global enp0s8
4: enp0s9: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc pfifo_fast state UP group default qlen 1000
    inet 192.168.11.132/32 brd 192.168.11.132 scope global enp0s9
13: mgmt: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    inet 172.16.0.132/24 brd 172.16.0.255 scope global mgmt
14: storage: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    inet 172.16.1.132/24 brd 172.16.1.255 scope global storage
15: vlan: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
16: vxlan: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    inet 172.16.2.132/24 brd 172.16.2.255 scope global vxlan
```

This command shows us each interface present and its' associated IP address.

# Conclusion

In this post, we covered some basic network interface configuration on our OpenStack hosts, including the bridges that OpenStack-Ansible needs present.

In the next post, we'll build on top of this configuration by bringing up OSPF and iBGP.

## References:

[Ansible - "template" module](https://docs.ansible.com/ansible/latest/plugins/lookup/template.html)  
[Ansible - "service" module](https://docs.ansible.com/ansible/latest/modules/service_module.html)   
[Ansible - "command" module](https://docs.ansible.com/ansible/latest/modules/command_module.html)   
[Ansible - "apt" module](https://docs.ansible.com/ansible/latest/modules/apt_module.html)   
[Debian Network Configuration](https://wiki.debian.org/NetworkConfiguration)  
[Debian Bridge Configuration](https://wiki.debian.org/BridgeNetworkConnections)  
[OpenStack Ansible - Deployment Guide](https://docs.openstack.org/project-deploy-guide/openstack-ansible/latest/)  
[OpenStack Ansible - Routed Environment Example](https://docs.openstack.org/openstack-ansible/latest/user/l3pods/example.html)  

## Versions used:

Desktop Machine: *kubuntu-18.04*  
VirtualBox: *virtualbox-5.2.18*  
Vagrant: *2.2.3*  
Cumulus VX Vagrant Box: *CumulusCommunity/cumulus-vx (virtualbox, 3.7.2)*  
Ubuntu Server Vagrant Box: *geerlingguy/ubuntu1804 (virtualbox, 1.0.6)*  
Ansible: *2.7.6*
