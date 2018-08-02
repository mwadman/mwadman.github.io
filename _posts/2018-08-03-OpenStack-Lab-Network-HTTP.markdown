---
layout:     post
title:      "OpenStack Lab Network - HTTP Server (Part 2)"
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

This post is the fourth in a series that plans to document my progress through installing and configuring a OpenStack Lab, using Cumulus switches for the network.

For other posts in this series, see the overview section of the [introduction post](https://www.wadman.co.nz/2018/02/08/OpenStack-Lab-Network-Introduction/#overview).

In the last post, we configured the DHCP part of the ZTP server.
This post will cover the installation of the HTTP server part on the ZTP server using Ansible.

# Configuration

## HTTP Server

### Ansible Playbook

### Ansible Role

### Ansible Variables

# Conclusion

In the next post

## References:

[Cumulus Technical Documentation - Zero Touch Provisioning](https://docs.cumulusnetworks.com/display/DOCS/Zero+Touch+Provisioning+-+ZTP)
[Cumulus Networks Example ZTP Scripts on Github](https://github.com/CumulusNetworks/example-ztp-scripts)

## Versions used:

Desktop Machine: *kubuntu-18.04*  
VirtualBox: *virtualbox-5.2.10*  
Vagrant: *2.1.2*  
Cumulus VX Vagrant Box: *CumulusCommunity/cumulus-vx (virtualbox, 3.6.2)*  
Ubuntu Server Vagrant Box: *geerlingguy/ubuntu1604 (virtualbox, 1.2.1)*  
Ansible: *2.6.2*  
