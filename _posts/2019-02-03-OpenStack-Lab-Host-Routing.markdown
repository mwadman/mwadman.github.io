---
layout:     post
title:      "OpenStack Lab Network - Host Routing"
subtitle:   "Configuring OSPF and BGP on OpenStack hosts"
date:       2019-02-03
author:     "Michael Wadman"
catalog:    true
tags:
    - OpenStack
    - Networking
    - OSPF
    - BGP
    - EVPN
    - Ansible
    - Vagrant
    - VirtualBox
    - Ubuntu
---

# Overview

This post is the ninth in a series that plans to document my progress through installing and configuring a small OpenStack Lab.

For other posts in this series, see the overview section of the [introduction post](https://www.wadman.co.nz/2018/02/08/OpenStack-Lab-Network-Introduction/#overview).

In the last post, we configured the network interfaces on our OpenStack hosts.
In this post, we'll cover the installation and configuration of our routing protocol suite, Free Range Routing (FRR), on our OpenStack hosts.

# Configuration

As I've [already gone over](https://wadman.co.nz/2018/12/09/OpenStack-Lab-Network-Switch-Routing/#free-range-routing) what FRR is and how it is configured, I'm not going to cover that in too much detail in this post.  
What I will cover is the differences between FRR on our switches and our OpenStack hosts, mainly the installation of FRR as this is not already present on our hosts.

At the end of this post, we'll have:
- OSPF neighborships from the hosts to their connected leaf switches.
- BGP peerings from the hosts to the spine switches/route reflectors, with the EVPN family/capability negotiated.

## Ansible Playbook and Role

Let's start with our playbook.

```yaml
---
- name: Configuring OpenStack hosts
  hosts: openstack_hosts
  become: true
  gather_facts: true
  roles:
    - { role: openstack-interfaces, tags: [ 'openstack-interfaces' ] }
    - { role: openstack-routing, tags: [ 'openstack-routing' ] }
```

And our role creation.

```bash
$ mkdir -p /etc/ansible/roles/openstack-routing/tasks/
```

## FRR Prerequisites

### FRR Packages

Because FRR does not come preinstalled on our hosts, our first tasks will cover the installation of the required packages.  
This step is made a little trickier for us because the latest versions of FRR are not present in any public repository (not even their own one).  
Instead, the installation (and any upgrade) of FRR will need to be done by manually downloading the packages from their [releases page on Github](https://github.com/FRRouting/frr/releases) (or directly from the [projects CI site](https://ci1.netdef.org/browse/FRR), by navigating to the "artifacts" page for the version you want to install - [Ubuntu 18.04 x86_64, as an example](https://ci1.netdef.org/browse/FRR-FRRV601-2/artifact/shared/Ubuntu-18.04-x86_64-Packages/)).

> Side note: If you're not looking for the features from the newest release, then Cumulus do provide a public repository for version 4.0. My recommendation would be to avoid this if at all possible, as this doesn't seem to be updated frequently.  
> Instructions on how to use their repository can be found on their website [here](https://docs.cumulusnetworks.com/display/HOSTPACK/Installing+FRRouting+on+the+Host).

Luckily for us, some people have already created a role that includes tasks for the installation of FRR (and supports Ubuntu 18.04 hosts like ours) - [ansible-frr](https://github.com/mrlesmithjr/ansible-frr).  
However, the role they've written also includes tasks for steps that we either want to complete ourselves ([configuration of FRR](https://github.com/mrlesmithjr/ansible-frr/blob/master/tasks/config.yml)) or don't want to include at all ([Installation of their custom scripts for monitoring](https://github.com/mrlesmithjr/ansible-frr/blob/master/tasks/monitor.yml)), so instead of calling the entire role in our playbook we're going to "include" this role in our tasks file and ask that only the installation tasks are completed.

The first thing we need to do is install the ansible-frr role as a git submodule so that we can reference it:

```bash
/etc/ansible$ git submodule add https://github.com/mrlesmithjr/ansible-frr.git roles/ansible-frr
```

After which, we can use Ansible's [`include_role`](https://docs.ansible.com/ansible/latest/modules/include_role_module.html) module with the argument `tasks_from` to specify that we only want to run the tasks in the "[debian.yml](https://github.com/mrlesmithjr/ansible-frr/blob/master/tasks/debian.yml)" tasks file (the ones needed to install FRR on Ubuntu).  
This is what our "tasks/main.yml" looks like:

```yaml
- name: Install FRR
  include_role:
    name: ansible-frr
    tasks_from: debian
```

By default, the above tasks file will download and install the latest stable version of FRR on our hosts.  
Thanks, ansible-frr!

### Kernel Version

Before we move on to the configuration of FRR, there's one other prerequisite that the above role doesn't consider - the kernel version of the host.  
The version being used on our OpenStack hosts becomes important when we take a look at the minimum kernel required for specific features of FRR, as outlined on either on FRR's [Github Wiki](https://github.com/FRRouting/frr/wiki/Features-and-Kernel-Support) or on their [official documentation site](http://docs.frrouting.org/en/latest/overview.html#supported-protocols-vs-platform).

According to these pages, to be able to run EVPN (with full feature support/at full performance) on our Linux based Ubuntu hosts we need to be running a minimum kernel version of 4.18.  
Unfortunately, if we look at our Ubuntu hosts we see that these are installed with kernel 4.15:

```bash
$ uname -r
4.15.0-29-generic
```

Luckily, Ubuntu 18.04 has what is called a "hardware enablement kernel", that we can install via a package from the default repositories, which will bring us up to the desired 4.18 kernel version.

To install this using Ansible, I am first going to write a task that will check our hosts to see if they need to be upgraded, and place this before our "Install FRR" task:

```yaml
---
- name: Check if minimum linux kernel version is met
  include_tasks: kernel-upgrade.yml
  when: ansible_kernel is version (openstack_routing_kernel_minimum, '<')

- name: Install FRR
  include_role:
    name: ansible-frr
    tasks_from: debian
```

Similar to the above use of the `include_role` module, we're going to use the module `include_tasks` to run the tasks inside of another file ("kernel-upgrade.yml") that we'll create next.

Also note the use of the ansible fact "ansible_kernel", which is really helpful with this task.
The `when` clause of the action takes that fact and checks whether it is less than the variable "openstack_routing_kernel_minimum" to see if the included tasks need to be run.  
We haven't yet defined the variable "openstack_routing_kernel_minimum" though, so let's do that before we move on:

In defaults/main.yml:

```yaml
---
openstack_routing_kernel_minimum: 4.18
```

Finally, this is what the "tasks/kernel-upgrade.yml" file ends up looking like:

```yaml
---
- name: Upgrade to required linux kernel version
  apt:
    name: linux-generic-hwe-18.04
    state: present
    cache_valid_time: 3600

- name: Reboot host after kernel upgrade
  reboot:
```

This is also pretty simple. Install the hardware enablement kernel package and then use the `reboot` module afterwards so that the new kernel is used (installing the package enables the kernel on the next boot).

We're done with our prerequisites. Onto configuration.

## FRR Configuration

Because I've covered the major points of FRR, OSPF and BGP in a previous post from the switch point of view, I'll try and make the next sections brief and again only cover the differences.

### Kernel Parameters

One of those differences is that our Ubuntu hosts are not readily built to act as "routers", due to the default settings of the kernel in Ubuntu 18.04 server.

One good example of this is the switch "net.ipv4.ip_forward", which by default is set to "0" (or "false" in boolean terms).  
With this setting left as is, the host will never forward packets that it receives either from other external devices (switches) or from internal virtual devices (virtual machines). This is pretty bad if we want virtual machines on our two hosts to be able to talk to each other.

Not to worry, these settings are quite easy to change.  
We'll use the trusty `template` module, along with a handler to load any changes made.

First, our task:

<!-- {% raw %} -->
```yaml
- name: Create sysctl file for routing tweaks
  template:
    src: sysctl.j2
    dest: "{{ openstack_routing_sysctl_file }}"
  notify:
    - Load sysctl changes
```
<!-- {% endraw %} -->

Then we need to define the variable "openstack_routing_sysctl_file".  
I'll do so in "defaults/main.yml":

```yaml
openstack_routing_sysctl_file: "/etc/sysctl.d/99free_range_routing.conf"
```

Our template file (I won't cover every setting. If you're interested, have a google!):

```yaml
# {{ ansible_managed }}
# /etc/sysctl.d/99free_range_routing.conf

# Enables Routing
net.ipv4.ip_forward=1

# Routing Options
net.ipv4.conf.all.ignore_routes_with_linkdown=1

# Enables Unnumbered BGP/OSPF
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
net.ipv4.conf.lo.rp_filter=0
net.ipv4.conf.all.forwarding=1
net.ipv4.conf.default.forwarding=1
net.ipv4.conf.default.arp_announce=2
net.ipv4.conf.default.arp_notify=1
net.ipv4.conf.default.arp_ignore=1
net.ipv4.conf.all.arp_announce=2
net.ipv4.conf.all.arp_notify=1
net.ipv4.conf.all.arp_ignore=1
net.ipv4.icmp_errors_use_inbound_ifaddr=1

# ARP/NDP Garbage Collection
net.ipv4.neigh.default.gc_thresh2=7168
net.ipv4.neigh.default.gc_thresh3=8192
net.ipv4.neigh.default.base_reachable_time_ms=14400000

# Use neighborship info on selection of nexthop address for multipath hops
net.ipv4.fib_multipath_use_neigh=1

# Allows applications to work with VRF
net.ipv4.tcp_l3mdev_accept=1
```

Last, our "handlers/main.yml" file:

```yaml
- name: Load sysctl changes
  command: "sysctl -p {{ openstack_routing_sysctl_file }}"
```

### Base Configuration

Just like our switches, we need to tell FRR which of its' daemons to load for the host to be active in our topology.  
A template will do the trick here too:

```yaml
- name: Set FRR daemons file
  template:
    src: "daemons.j2"
    dest: "/etc/frr/daemons"
  notify: Restart FRR
```

With our template looking like the below:

<!-- {% raw %} -->
```jinja
# {{ ansible_managed }}
# This file tells the frr package which daemons to start.
#
# Sample configurations for these daemons can be found in
# /usr/share/doc/frr/examples/.
#
# ATTENTION:
#
# When activation a daemon at the first time, a config file, even if it is
# empty, has to be present *and* be owned by the user and group "frr", else
# the daemon will not be started by /etc/init.d/frr. The permissions should
# be u=rw,g=r,o=.
# When using "vtysh" such a config file is also needed. It should be owned by
# group "frrvty" and set to ug=rw,o= thendough. Check /etc/pam.d/frr, too.
#
# The watchfrr and zebra daemons are always started.
#
bgpd={{   "yes" if openstack_hosts_routing_bgp_enabled   else "no" }}
ospfd={{  "yes" if openstack_hosts_routing_ospf_enabled  else "no" }}
ospf6d={{ "yes" if openstack_hosts_routing_ospf6_enabled else "no" }}
ripd={{   "yes" if openstack_hosts_routing_rip_enabled   else "no" }}
ripngd={{ "yes" if openstack_hosts_routing_ripng_enabled else "no" }}
isisd={{  "yes" if openstack_hosts_routing_isis_enabled  else "no" }}
pimd={{   "yes" if openstack_hosts_routing_pim_enabled   else "no" }}
ldpd={{   "yes" if openstack_hosts_routing_ldp_enabled   else "no" }}
nhrpd={{  "yes" if openstack_hosts_routing_nhrp_enabled  else "no" }}
eigrpd={{ "yes" if openstack_hosts_routing_eigrp_enabled else "no" }}
babeld={{ "yes" if openstack_hosts_routing_babel_enabled else "no" }}
sharpd={{ "yes" if openstack_hosts_routing_sharp_enabled else "no" }}
pbrd={{   "yes" if openstack_hosts_routing_pbr_enabled   else "no" }}
bfdd={{   "yes" if openstack_hosts_routing_bfd_enabled   else "no" }}

#
# If this option is set the /etc/init.d/frr script automatically loads
# the config via "vtysh -b" when the servers are started.
# Check /etc/pam.d/frr if you intend to use "vtysh"!
#
vtysh_enable=yes
zebra_options="  --daemon -A 127.0.0.1 -s 90000000"
bgpd_options="   --daemon -A 127.0.0.1"
ospfd_options="  --daemon -A 127.0.0.1"
ospf6d_options=" --daemon -A ::1"
ripd_options="   --daemon -A 127.0.0.1"
ripngd_options=" --daemon -A ::1"
isisd_options="  --daemon -A 127.0.0.1"
pimd_options="   --daemon -A 127.0.0.1"
ldpd_options="   --daemon -A 127.0.0.1"
nhrpd_options="  --daemon -A 127.0.0.1"
eigrpd_options=" --daemon -A 127.0.0.1"
babeld_options=" --daemon -A 127.0.0.1"
sharpd_options=" --daemon -A 127.0.0.1"
pbrd_options="   --daemon -A 127.0.0.1"
staticd_options="--daemon -A 127.0.0.1"
bfdd_options="   --daemon -A 127.0.0.1"

# The list of daemons to watch is automatically generated by the init script.
watchfrr_options="-r '/usr/lib/frr/watchfrr.sh restart %s' -s '/usr/lib/frr/watchfrr.sh start %s' -k '/usr/lib/frr/watchfrr.sh stop %s'"

# for debugging purposes, you can specify a "wrap" command to start instead
# of starting the daemon directly, e.g. to use valgrind on ospfd:
#   ospfd_wrap="/usr/bin/valgrind"
# or you can use "all_wrap" for all daemons, e.g. to use perf record:
#   all_wrap="/usr/bin/perf record --call-graph -"
# the normal daemon command is added to this at the end.
```
<!-- {% endraw %} -->

There are two differences in the above file, as compared to our switches "daemons" file:
- On our hosts, we do not need to specify that the "zebra" daemon is to run, as this is presumed by default in version 6.0.2 (the latest version at the time of writing).
- We need to include the options that our daemons will be started ("$DAEMON_options="). I've just used the [defaults](http://docs.frrouting.org/en/latest/setup.html#daemons-configuration-file).

&nbsp; <!--- Used to add a double line break --->

Before we dive into our routing protocol configuration, let's also get our "frr.conf" file started.  
The task:

```yaml
- name: Set FRR running configuration
  template:
    src: "frr.conf.j2"
    dest: "/etc/frr/frr.conf"
  notify: Reload FRR
```

The handler:

```yaml
- name: Reload FRR
  service:
    name: frr
    state: reloaded
```

And the template "frr.conf.j2":

<!-- {% raw %} -->
```jinja
! {{ ansible_managed }}
!
frr defaults datacenter
!
hostname {{ inventory_hostname }}
!
service integrated-vtysh-config
!
log syslog informational
!
line vty
!
```
<!-- {% endraw %} -->

Nothing has changed here from our switches initial FRR configuration.

We'll also add a task at the end of our role to start FRR after the configuration has been set:

```yaml
- name: Start and Enable FRR
  service:
    name: frr
    state: started
    enabled: yes
```

### OSPF

Seeing as our goal with OSPF is very close to that of our switches, with the only addition being that we're advertising the bridge IP addressing, it'll be no surprise that our configuration is very close on each as well:

<!-- {% raw %} -->
```jinja
!
router-id {{ openstack_host_loopback_address | ipaddr('address') }}
!
!
{% if openstack_hosts_routing_ospf_enabled == true %}
{# Configure switchports as OSPF point-to-point #}
{% for port, mac in openstack_hosts_network_interfaces.items() | sort %}
interface {{ port }}
  ip ospf network point-to-point
!
{% endfor %}
!
{% for bridge, address in openstack_host_network_bridges.items() | sort %}
{% if address is not none %}
interface {{ bridge }}
  no link-detect
!
{% endif %}
{% endfor %}
{% set host_number = (openstack_host_loopback_address | ipaddr('address')).split('.')[3] %}
!
router ospf
  ospf router-id {{ openstack_host_loopback_address | ipaddr('address') }}
  network {{ openstack_host_loopback_address }} area 0
  passive-interface lo
{% for bridge, address in openstack_host_network_bridges.items() | sort %}
{% if address is not none %}
  network {{ address | ipaddr('network/prefix') }} area {{ host_number }}
  passive-interface {{ bridge }}
{% endif %}
{% endfor %}
  area {{ host_number }} range 10.{{ host_number }}.0.0/16
{% endif %}
!
```
<!-- {% endraw %} -->

&nbsp; <!--- Used to add a double line break --->

I'll cover some sections of note.

<!-- {% raw %} -->
```jinja
{% for bridge, address in openstack_host_network_bridges.items() | sort %}
{% if address is not none %}
interface {{ bridge }}
  no link-detect
!
{% endif %}
{% endfor %}
```
<!-- {% endraw %} -->

FRR considers the bridge interfaces we've configured as 'down', because don't have any member interfaces (yet).  
To bypass this, we need tell FRR to not care (detect) whether the link is up or down when deciding whether to advertise the associated networks.

<!-- {% raw %} -->
```jinja
{% set host_number = (openstack_host_loopback_address | ipaddr('address')).split('.')[3] %}
...
  area {{ host_number }} range 10.{{ host_number }}.0.0/16
```
<!-- {% endraw %} -->

Because we've addressed our bridge interfaces in such a way that they can be summarised on each host, we can use the `area x range 10.x.0.0/16` command.  
This will tell the host to send out a summary route (/16) instead of individual routes.  
To determine how to summarise, we take the take octet of the loopback address and set this to a variable using Jinja2's `set` command.

### BGP

Our BGP is going to be very similar too, with the only changes being:
- We don't require logic to check if we need BGP because it'll be present on all (well, "both" in our lab) hosts.
- We don't need the route reflector configuration, because we're configuring our hosts as clients only.

<!-- {% raw %} -->
```jinja
!
{% if openstack_hosts_routing_bgp_enabled == true %}
router bgp {{ ibgp_autonomous_system }}
  bgp router-id {{ openstack_host_loopback_address | ipaddr('address') }}
{# Defines a peer group where common configuration to be defined for all peers #}
  neighbor iBGP-RRs peer-group
  neighbor iBGP-RRs remote-as internal
{# Forms a neighborship with spine switches/route reflectors #}
{% for spine in groups['openstack_cumulus_spines'] %}
  neighbor {{ hostvars[spine].cumulus_host_loopback_address | ipaddr('address') }} peer-group iBGP-RRs
{% endfor %}
{% if openstack_hosts_routing_bgp_evpn_enabled == true %}
{# Enables advetisement of EVPN address family #}
  address-family l2vpn evpn
    neighbor iBGP-RRs activate
    advertise-all-vni
  exit-address-family
{% endif %}
{% endif %}
!
```
<!-- {% endraw %} -->

# Testing

After another `vagrant up`, the above configuration is applied and we can log into our hosts with `vagrant ssh` to check everything is working.  
The below is taken from the perspective of one of the hosts.

The first check we should complete is to ensure FRR and its' daemons are running:

```bash
$ sudo systemctl status frr.service
● frr.service - FRRouting
   Loaded: loaded (/lib/systemd/system/frr.service; enabled; vendor preset: enabled)
   Active: active (running) since Sun 2019-02-17 03:13:41 UTC; 1h 8min ago
     Docs: https://frrouting.readthedocs.io/en/latest/setup.html
  Process: 570 ExecStart=/usr/lib/frr/frrinit.sh start (code=exited, status=0/SUCCESS)
    Tasks: 11 (limit: 4915)
   CGroup: /system.slice/frr.service
           ├─603 /usr/lib/frr/watchfrr -d -r /usr/lib/frr/watchfrr.sh restart %s -s /usr/lib/frr/watchfrr.sh start %s -k /usr/lib/frr/watchfrr.sh stop %s zebra bgpd ospfd staticd
           ├─631 /usr/lib/frr/zebra -d --daemon -A 127.0.0.1 -s 90000000
           ├─710 /usr/lib/frr/bgpd -d --daemon -A 127.0.0.1
           ├─841 /usr/lib/frr/ospfd -d --daemon -A 127.0.0.1
           └─968 /usr/lib/frr/staticd -d --daemon -A 127.0.0.1

Feb 17 03:13:41 vagrant watchfrr[603]: bgpd state -> up : connect succeeded
Feb 17 03:13:41 vagrant watchfrr[603]: ospfd state -> up : connect succeeded
Feb 17 03:13:41 vagrant watchfrr[603]: staticd state -> up : connect succeeded
Feb 17 03:13:41 vagrant watchfrr[603]: all daemons up, doing startup-complete notify
```

Looking good. Now let's confirm our routing protocols look happy.

## OSPF Testing

We'll start with OSPF because that's our routing underlay (BGP won't come up without OSPF working first).

Let's check neighborship to the leaf switches:

```
$ sudo vtysh

Hello, this is FRRouting (version 6.0.2).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

vagrant# show ip ospf neighbor

Neighbor ID     Pri State           Dead Time Address         Interface            RXmtL RqstL DBsmL
192.168.11.113    1 Full/DROther      37.310s 192.168.11.113  enp0s8:192.168.11.132     0     0     0
192.168.11.114    1 Full/DROther      37.518s 192.168.11.114  enp0s9:192.168.11.132     0     0     0
```

Are we receiving OSPF routes?

```
vagrant# show ip route ospf
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, D - SHARP,
       F - PBR,
       > - selected route, * - FIB route

O>* 192.168.11.101/32 [110/200] via 192.168.11.113, enp0s8 onlink, 00:09:45
  *                             via 192.168.11.114, enp0s9 onlink, 00:09:45
O>* 192.168.11.102/32 [110/200] via 192.168.11.113, enp0s8 onlink, 00:09:45
  *                             via 192.168.11.114, enp0s9 onlink, 00:09:45
O>* 192.168.11.111/32 [110/300] via 192.168.11.113, enp0s8 onlink, 00:09:45
  *                             via 192.168.11.114, enp0s9 onlink, 00:09:45
O>* 192.168.11.112/32 [110/300] via 192.168.11.113, enp0s8 onlink, 00:09:45
  *                             via 192.168.11.114, enp0s9 onlink, 00:09:45
O>* 192.168.11.113/32 [110/100] via 192.168.11.113, enp0s8 onlink, 00:09:45
O>* 192.168.11.114/32 [110/100] via 192.168.11.114, enp0s9 onlink, 00:09:45
O>* 192.168.11.131/32 [110/400] via 192.168.11.113, enp0s8 onlink, 00:09:46
  *                             via 192.168.11.114, enp0s9 onlink, 00:09:46
O   192.168.11.132/32 [110/0] is directly connected, lo, 01:13:49
```

This is great. Not only can we see that we're receiving routes all the way from the other OpenStack host (192.168.11.131/32), but all of the routes are installed as ECMP routes too (indicated by the presence of two "routes" to each prefix and the asterisk at the start of each of the route lines).

## BGP Testing

Now that we know that we have IP connectivity to the spines/route reflectors, we shouldn't have any issues with BGP right?  
We can confirm this from vtysh as well:

```
$ sudo vtysh

Hello, this is FRRouting (version 6.0.2).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

vagrant# show ip bgp summary

IPv4 Unicast Summary:
BGP router identifier 192.168.11.132, local AS number 65000 vrf-id 0
BGP table version 0
RIB entries 11, using 1760 bytes of memory
Peers 2, using 41 KiB of memory
Peer groups 1, using 64 bytes of memory

Neighbor        V         AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd
192.168.11.101  4      65000    1562    1562        0    0    0 00:10:55            3
192.168.11.102  4      65000    1562    1562        0    0    0 00:10:55            3

Total number of neighbors 2
```

Both neighbours are showing as up, which is great.

We're also receiving prefixes from both peers (as indicated by the "3" under the header "State/PfxRcd").  
We can find what these are using `show ip route bgp`:

```
vagrant# show ip route bgp
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, D - SHARP,
       F - PBR,
       > - selected route, * - FIB route

B>  172.16.131.0/24 [200/0] via 192.168.11.131 (recursive), 00:12:01
  *                           via 192.168.11.113, enp0s8 onlink, 00:12:01
  *                           via 192.168.11.114, enp0s9 onlink, 00:12:01
B>  172.17.131.0/24 [200/0] via 192.168.11.131 (recursive), 00:12:01
  *                           via 192.168.11.113, enp0s8 onlink, 00:12:01
  *                           via 192.168.11.114, enp0s9 onlink, 00:12:01
B>  172.18.131.0/24 [200/0] via 192.168.11.131 (recursive), 00:12:01
  *                           via 192.168.11.113, enp0s8 onlink, 00:12:01
  *                           via 192.168.11.114, enp0s9 onlink, 00:12:01
```

This is mimicked by the kernel routing table, which we can look at by using iproute2's "ip route show" command:

```bash
$ ip route show
default via 192.168.11.1 dev enp0s3 proto dhcp src 192.168.11.232 metric 100
172.16.131.0/24 proto bgp metric 20
	nexthop via 192.168.11.113 dev enp0s8 weight 1 onlink
	nexthop via 192.168.11.114 dev enp0s9 weight 1 onlink
172.16.132.0/24 dev mgmt proto kernel scope link src 172.16.132.1 linkdown
172.17.131.0/24 proto bgp metric 20
	nexthop via 192.168.11.113 dev enp0s8 weight 1 onlink
	nexthop via 192.168.11.114 dev enp0s9 weight 1 onlink
172.17.132.0/24 dev storage proto kernel scope link src 172.17.132.1 linkdown
172.18.131.0/24 proto bgp metric 20
	nexthop via 192.168.11.113 dev enp0s8 weight 1 onlink
	nexthop via 192.168.11.114 dev enp0s9 weight 1 onlink
172.18.132.0/24 dev vxlan proto kernel scope link src 172.18.132.1 linkdown
192.168.11.0/24 dev enp0s3 proto kernel scope link src 192.168.11.232
192.168.11.1 dev enp0s3 proto dhcp scope link src 192.168.11.232 metric 100
192.168.11.101 proto ospf metric 20
	nexthop via 192.168.11.113 dev enp0s8 weight 1 onlink
	nexthop via 192.168.11.114 dev enp0s9 weight 1 onlink
192.168.11.102 proto ospf metric 20
	nexthop via 192.168.11.113 dev enp0s8 weight 1 onlink
	nexthop via 192.168.11.114 dev enp0s9 weight 1 onlink
192.168.11.111 proto ospf metric 20
	nexthop via 192.168.11.113 dev enp0s8 weight 1 onlink
	nexthop via 192.168.11.114 dev enp0s9 weight 1 onlink
192.168.11.112 proto ospf metric 20
	nexthop via 192.168.11.113 dev enp0s8 weight 1 onlink
	nexthop via 192.168.11.114 dev enp0s9 weight 1 onlink
192.168.11.113 via 192.168.11.113 dev enp0s8 proto ospf metric 20 onlink
192.168.11.114 via 192.168.11.114 dev enp0s9 proto ospf metric 20 onlink
192.168.11.131 proto ospf metric 20
	nexthop via 192.168.11.113 dev enp0s8 weight 1 onlink
	nexthop via 192.168.11.114 dev enp0s9 weight 1 onlink
```

# Conclusion

In this post, we covered how to set up FRR on our OpenStack hosts in preparation of them advertising their VXLAN networks to each other.  

In the next post, we'll be preparing our hosts for deployment of OpenStack-Ansible.

## References:

[Ansible - "include_tasks" module](https://docs.ansible.com/ansible/latest/modules/include_tasks_module.html)  
[Ansible - "apt" module](https://docs.ansible.com/ansible/latest/modules/apt_module.html)  
[Ansible - "reboot" module](https://docs.ansible.com/ansible/latest/modules/reboot_module.html)  
[Ansible - "include_role" module](https://docs.ansible.com/ansible/latest/modules/include_role_module.html)  
[Ansible - "template" module](https://docs.ansible.com/ansible/latest/plugins/lookup/template.html)  
[Ansible - "service" module](https://docs.ansible.com/ansible/latest/modules/service_module.html)  
[ansible-frr role](https://github.com/mrlesmithjr/ansible-frr)  
[FRR - Features and Kernel Support](https://github.com/FRRouting/frr/wiki/Features-and-Kernel-Support)

## Versions used:

Desktop Machine: *kubuntu-18.04*  
VirtualBox: *virtualbox-5.2.18*  
Vagrant: *2.2.3*  
Cumulus VX Vagrant Box: *CumulusCommunity/cumulus-vx (virtualbox, 3.7.2)*  
Ubuntu Server Vagrant Box: *geerlingguy/ubuntu1804 (virtualbox, 1.0.6)*  
Ansible: *2.7.6*
