---
layout:     post
title:      "OpenStack Lab Network - Vagrant"
subtitle:   "Creating a lab environment in VirtualBox"
date:       2018-04-08
author:     "Michael Wadman"
catalog:    true
tags:
    - OpenStack
    - Networking
    - Cumulus
    - Vagrant
    - VirtualBox
---

# Overview

This post is the second in a series that plans to document my progress through installing and configuring a OpenStack Lab, using Cumulus switches for the network.

For other posts in this series, see the overview section of the [introduction post](https://www.wadman.co.nz/2018/02/08/OpenStack-Lab-Network-Introduction/#overview).

This post will cover the creation of our Virtual Machines in VirtualBox using Vagrant.  
Vagrant (by HashiCorp) can be described as a tool to spin up reproducible development environments using most mainstream hypervisors.

# Installation

Installation of Vagrant on Ubuntu (my choice of desktop distribution) requires downloading the deb file from the [official website](https://www.vagrantup.com/downloads.html), or (as I recently found out) adding a community repository which is a kind of proxy-repository as it just redirects to the packages from HashiCorp themselves.  
For instructions on how to install Vagrant using this repo, visit the website [here](https://vagrant-deb.linestarve.com/).

There is a Vagrant package that is available in the default repositories, but this isn't kept up to date.

An additional step I'll take is to copy the bash completion script for Vagrant over so that it can be used (I don't know why this isn't done by default, seeing as the file is included in the package anyway - see Github issue [#8987](https://github.com/hashicorp/vagrant/issues/8987)).

```bash
$ sudo cp /opt/vagrant/embedded/gems/2.0.3/gems/vagrant-2.0.3/contrib/bash/completion.sh /etc/bash_completion.d/vagrant.sh
```

Remember to replace the version directories with your version of Vagrant.  
You'll also need to start a new shell in order for this to take effect.

&nbsp; <!--- Used to add a double line break --->

Usually with Vagrant the next step would be to install a plugin to support the hypervisor (named a Vagrant 'provider') you are using, but VirtualBox is supported by default.

Now we need to add a 'Box', which is Vagrant's term for an image.
The difference between Vagrant boxes and most other operating system images is that boxes are pre-made and configured, from which new virtual machines are cloned (as opposed to being installed by the user).

For example, there is a [Cumulus VX Vagrant](https://app.vagrantup.com/boxes/search?utf8=%E2%9C%93&amp;sort=downloads&amp;provider=&amp;q=Cumulus) box available that we'll use in this project - named "CumulusCommunity/cumulus-vx". We'll add this now along with a Ubuntu 16.04 box.

>*Note that I'm not using the official Ubuntu box, as I ran into issues booting this on my machine (See the link [here](https://askubuntu.com/questions/771871/16-04-virtualbox-vm-from-vhd-file-hangs-at-non-blocking-pool-is-initialized), except those fixes didn't work in my case). There are other ubuntu boxes like geerlingguy/1604 that don't have the same issue.*


```bash
$ vagrant box add CumulusCommunity/cumulus-vx
$ vagrant box add geerlingguy/ubuntu1604
```

During this step, if a box has more versions for more than just one provider it will prompt to ask which version to download.

These boxes are stored globally for the user in the directory "~/.vagrant.d/boxes" and the currently installed boxes can be shown using the command below

```bash
$ vagrant box list
CumulusCommunity/cumulus-vx (virtualbox, 3.6.2)
geerlingguy/ubuntu1604      (virtualbox, 1.2.1)
```

# Setup

Now that we've got our host ready, we'll create a 'Vagrantfile'.  
As described by the [official documentation](https://www.vagrantup.com/docs/vagrantfile/):

>The primary function of the Vagrantfile is to describe the type of machine required for a project, and how to configure and provision these machines"

More simply put, the file marks the root directory of a project in Vagrant and some configuration associated with it. This is created using the `vagrant init` command in the directory that you want to use as your project root:

```bash
$ mkdir -p Vagrant/OpenStack_Lab/
$ cd Vagrant/OpenStack_Lab/
$ vagrant init
$ ls
Vagrantfile
```

By default this file is pretty empty, with only three lines uncommented.

```ruby
Vagrant.configure("2") do |config|
 config.vm.box = "base"
end
```

The `do` and `end` lines form a 'block' in Ruby, in which you define items for the function (the "Vagrant.configure" part) to invoke.  
With the Vagrantfile, all we're doing is defining configuration. Nothing fancy here!
Let's change some then.

According to the [Getting Started documentation](https://www.vagrantup.com/intro/getting-started/boxes.html#using-a-box), the first thing we should change is the base box to whichever of the boxes we've downloaded we want to use.
However because our project isn't quite as simple as using only one VM we need to [define multiple](https://www.vagrantup.com/docs/multi-machine/#defining-multiple-machines), and this is done by adding `"config.vm.define"` blocks inside the main configure block.

We'll do one block for each of the VM's we need - 2 spine switches, 4 leaf switches, 2 Ubuntu OpenStack machines and another Ubuntu VM for ZTP:

```ruby
Vagrant.configure("2") do |config|
 # ZTP Box
 config.vm.define "openstack_ztp" do |device|
	 device.vm.box = "geerlingguy/ubuntu1604"
 end

 # Cumulus Boxes
 config.vm.define "cumulus_spine01" do |device|
	 device.vm.box = "CumulusCommunity/cumulus-vx"
 end

 config.vm.define "cumulus_spine02" do |device|
	 device.vm.box = "CumulusCommunity/cumulus-vx"
 end

 config.vm.define "cumulus_leaf01" do |device|
	 device.vm.box = "CumulusCommunity/cumulus-vx"
 end

 config.vm.define "cumulus_leaf02" do |device|
	 device.vm.box = "CumulusCommunity/cumulus-vx"
 end

 config.vm.define "cumulus_leaf03" do |device|
	 device.vm.box = "CumulusCommunity/cumulus-vx"
 end

 config.vm.define "cumulus_leaf04" do |device|
	 device.vm.box = "CumulusCommunity/cumulus-vx"
 end

 # OpenStack Boxes
 config.vm.define "openstack_control" do |device|
	 device.vm.box = "geerlingguy/ubuntu1604"
 end

 config.vm.define "openstack_compute" do |device|
	 device.vm.box = "geerlingguy/ubuntu1604"
 end
end
```

With this configured, we could go ahead and provision then boot all of the machines we've specified using the `vagrant up` command, and then the `vagrant destroy` to delete the VM's.

However, there is still more configuration to do before we can be happy that our lab can be created in its entirety.

## VirtualBox configuration

VirtualBox specific configuration can be defined both globally individually for each VM using another block - `.vm.provider`

For example, we can set what each VM is named in VirtualBox:

```ruby
config.vm.define "cumulus_spine01" do |device|
	 device.vm.box = "CumulusCommunity/cumulus-vx"

	 device.vm.provider "virtualbox" do |vb|
		 vb.name = "openstack_spine01"
	 end
 end
```

More importantly, we can also specify how many CPU's and how much RAM we allocate to the VM:

```ruby
config.vm.define "cumulus_spine01" do |device|
	 device.vm.box = "CumulusCommunity/cumulus-vx"

	 device.vm.provider "virtualbox" do |vb|
		 vb.name = "openstack_spine01"
		 vb.memory = 1024
		 vb.cpus = 1
	 end
 end
```

Most other VirtualBox specific configuration needs to be done through the use of [VBoxManage](https://www.virtualbox.org/manual/ch08.html) `modifyvm` commands and a `.customize` entry.
In the below, we're adding all VM's to a group in VirtualBox so that we can easily tell that it is a part of the OpenStack Lab.

```ruby
config.vm.provider "virtualbox" do |vb|
	 vb.customize ["modifyvm", :id, "--groups", "/OpenStack Lab"]
 end
```

Note that this is done at the "config" level, outside of each host.
This is an example of the global configuration that I referenced earlier.

With the above, the VM's will now be grouped in VirtualBox, like so:

![](/img/virtualbox-groups.png)

This would usually look like the following if you we're using the VBoxManage command line utility:

```bash
VBoxManage modifyvm "cumulus_spine01" --groups "/OpenStack Lab"
```

&nbsp; <!--- Used to add a double line break --->

We'll apply the configuration completed above to the rest of our machines and we're done with VirtualBox configuration:

```ruby
# ZTP Box
config.vm.define "openstack_ztp" do |device|
 device.vm.box = "geerlingguy/ubuntu1604"

 device.vm.provider "virtualbox" do |vb|
	 vb.name = "ztp-server"
	 vb.memory = 2048
	 vb.cpus = 1
 end
end

# Cumulus Switches
config.vm.define "cumulus_spine01" do |device|
	 device.vm.box = "CumulusCommunity/cumulus-vx"

	 device.vm.provider "virtualbox" do |vb|
		 vb.name = "cumulus_spine01"
		 vb.memory = 1024
		 vb.cpus = 1
	 end
 end

 config.vm.define "cumulus_spine02" do |device|
	 device.vm.box = "CumulusCommunity/cumulus-vx"

	 device.vm.provider "virtualbox" do |vb|
		 vb.name = "cumulus_spine02"
		 vb.memory = 1024
		 vb.cpus = 1
	 end
 end

 config.vm.define "cumulus_leaf01" do |device|
	 device.vm.box = "CumulusCommunity/cumulus-vx"

	 device.vm.provider "virtualbox" do |vb|
		 vb.name = "cumulus_leaf01"
		 vb.memory = 1024
		 vb.cpus = 1
	 end
 end

 config.vm.define "cumulus_leaf02" do |device|
	 device.vm.box = "CumulusCommunity/cumulus-vx"

	 device.vm.provider "virtualbox" do |vb|
		 vb.name = "cumulus_leaf02"
		 vb.memory = 1024
		 vb.cpus = 1
	 end
 end

 config.vm.define "cumulus_leaf03" do |device|
	 device.vm.box = "CumulusCommunity/cumulus-vx"

	 device.vm.provider "virtualbox" do |vb|
		 vb.name = "cumulus_leaf03"
		 vb.memory = 1024
		 vb.cpus = 1
	 end
 end

 config.vm.define "cumulus_leaf04" do |device|
	 device.vm.box = "CumulusCommunity/cumulus-vx"

	 device.vm.provider "virtualbox" do |vb|
		 vb.name = "cumulus_leaf04"
		 vb.memory = 1024
		 vb.cpus = 1
	 end
 end

 # OpenStack Boxes
 config.vm.define "openstack_control" do |device|
	 device.vm.box = "geerlingguy/ubuntu1604"

	 device.vm.provider "virtualbox" do |vb|
		 vb.name = "openstack_control"
		 vb.memory = 4096
		 vb.cpus = 1
	 end
 end

 config.vm.define "openstack_compute" do |device|
	 device.vm.box = "geerlingguy/ubuntu1604"

	 device.vm.provider "virtualbox" do |vb|
		 vb.name = "openstack_compute"
		 vb.memory = 8192
		 vb.cpus = 2
	 end
 end
```


## Networking

Vagrant networking is as simple (or as hard) as the provider (hypervisor) you are using makes it.

There is one limitation with VirtualBox, in that it can only support a maximum of 36 interfaces per machine.  
This doesn't bother us because the maximum amount of interfaces one of our machines has is 4 (Each leaf switch has 1 management, 2 uplinks to the spine switches and 1 downlink interface to the attached OpenStack host).

One other thing to be aware of is that Vagrant sets the first network interface as a NAT interface by default.  
A NAT interface in VirtualBox allows all communication outbound (through NAT of the hosts primary interface), but only allows connections inbound (including from other VM's) if they are port-forwarded.  
While we could leave this as the default and configure port-forwards for each inbound connection, this becomes cumbersome when you start to think about DHCP/TFTP traffic (to/from the ZTP server) and OpenStack control traffic.

Instead we can change the first network interface for all hosts to be part of a "public network" (a bridged network in VirtualBox).

```ruby
config.vm.network :public_network,
	 adapter: "1", # Set the first network adapter
	 bridge: "enp6s0", # To bridge over enp6s0
	 auto_config: false # And don't use Vagrant to configure this inside the VM
```

This comes with the caveat that Vagrant can no longer configure the network interface of the VM, and so cannot give it an IP address.
This instead needs to be completed by one of the following:

* Use a Vagrant box that comes with a preconfigured network interface
* Configure the interface manually by logging in through the console
* Use a DHCP to assign network addressing

We'll use DHCP in this lab as we're going to be configuring a DHCP server for ZTP anyway.

&nbsp; <!--- Used to add a double line break --->

This does mean that we'll need to boot and set an address on the ZTP server manually before provisioning any of the other hosts. I'll cover this in the next post.

To make this easier down the road, lets set the MAC address of the first in VirtualBox statically. This is done in the `device.vm.provider` section of each guest, just like setting the RAM and the CPU:

```ruby
      vb.customize ["modifyvm", :id, "--macaddress1", "080027000001"]
```

Because Vagrant no longer configures the interfaces itself, we also need to tell it what IP address to connect to in order to finish its' provisioning.
This is done with the `.ssh.host` option under each host.

```ruby
    device.ssh.host = "192.168.11.201"
```

The last thing we need to do is configure the network interfaces for the links between the VM's.
This will be done using "private networks" (internal networks in VirtualBox).

For example, to configure the link between the first spine and leaf switches, we need to add the below to each the config of each VM:

```ruby
  device.vm.network "private_network",
		 adapter: "2", # Which adapter to configure on the VM
		 virtualbox__intnet: "spine01-to-leaf01", # The internal adapter to use
		 auto_config: false # Don't use Vagrant to configure this inside the VM
```

Note that the double underscore [isn't a typo.](https://www.vagrantup.com/docs/virtualbox/networking.html)

&nbsp; <!--- Used to add a double line break --->

After we add the above to all of or devices, our configuration should look like the below:

```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

#Start of Vagrant Configuration
Vagrant.configure("2") do |config|
  ########################
  # Global configuration #
  ########################

  # Adds all VM's to the same group for easy identification in Virtualbox
  config.vm.provider "virtualbox" do |vb|
    vb.customize ["modifyvm", :id, "--groups", "/OpenStack"]
  end

  # The first interface for all VM's will connect to a bridged interface
  config.vm.network :public_network,
    adapter: "1",
    bridge: "enp6s0",
    auto_config: false

  ##########
  # ZTP VM #
  ##########

  # ZTP VM
  config.vm.define "openstack_ztp" do |device|
    device.vm.box = "geerlingguy/ubuntu1604"
    device.ssh.host = "192.168.11.221"

    # VirtualBox Config
    device.vm.provider "virtualbox" do |vb|
      vb.name = "ztp-server"
      vb.memory = 1024
      vb.cpus = 1
      vb.customize ["modifyvm", :id, "--macaddress1", "080027000021"]
    end
  end

  ################
  # Cumulus VM's #
  ################

  # Spine 1
  config.vm.define "cumulus_spine01" do |device|
    device.vm.box = "CumulusCommunity/cumulus-vx"
    device.ssh.host = "192.168.11.201"

    # Internal Networks
    device.vm.network "private_network",
      adapter: "2",
      virtualbox__intnet: "OpenStack_spine01-to-leaf01",
      auto_config: false

    device.vm.network "private_network",
      adapter: "3",
      virtualbox__intnet: "OpenStack_spine01-to-leaf02",
      auto_config: false

    device.vm.network "private_network",
      adapter: "4",
      virtualbox__intnet: "OpenStack_spine01-to-leaf03",
      auto_config: false

    device.vm.network "private_network",
      adapter: "5",
      virtualbox__intnet: "OpenStack_spine01-to-leaf04",
      auto_config: false

    # VirtualBox Config
    device.vm.provider "virtualbox" do |vb|
      vb.name = "cumulus_spine01"
      vb.memory = 1024
      vb.cpus = 1
      vb.customize ["modifyvm", :id, "--macaddress1", "080027000001"]
    end
  end

  # Spine 2
  config.vm.define "cumulus_spine02" do |device|
    device.vm.box = "CumulusCommunity/cumulus-vx"
    device.ssh.host = "192.168.11.202"

    # Internal Networks
    device.vm.network "private_network",
      adapter: "2",
      virtualbox__intnet: "OpenStack_spine02-to-leaf01",
      auto_config: false

    device.vm.network "private_network",
      adapter: "3",
      virtualbox__intnet: "OpenStack_spine02-to-leaf02",
      auto_config: false

    device.vm.network "private_network",
      adapter: "4",
      virtualbox__intnet: "OpenStack_spine02-to-leaf03",
      auto_config: false

    device.vm.network "private_network",
      adapter: "5",
      virtualbox__intnet: "OpenStack_spine02-to-leaf04",
      auto_config: false

    # VirtualBox Config
    device.vm.provider "virtualbox" do |vb|
      vb.name = "cumulus_spine02"
      vb.memory = 1024
      vb.cpus = 1
      vb.customize ["modifyvm", :id, "--macaddress1", "080027000002"]
    end
  end

  # Leaf 1
  config.vm.define "cumulus_leaf01" do |device|
    device.vm.box = "CumulusCommunity/cumulus-vx"
    device.ssh.host = "192.168.11.211"

    # Internal Networks
    device.vm.network "private_network",
      adapter: "2",
      virtualbox__intnet: "OpenStack_spine01-to-leaf01",
      auto_config: false

    device.vm.network "private_network",
      adapter: "3",
      virtualbox__intnet: "OpenStack_spine02-to-leaf01",
      auto_config: false

    device.vm.network "private_network",
      adapter: "4",
      virtualbox__intnet: "OpenStack_leaf01-to-control01",
      auto_config: false

    # VirtualBox Config
    device.vm.provider "virtualbox" do |vb|
      vb.name = "cumulus_leaf01"
      vb.memory = 1024
      vb.cpus = 1
      vb.customize ["modifyvm", :id, "--macaddress1", "080027000011"]
    end
  end

  # Leaf 2
  config.vm.define "cumulus_leaf02" do |device|
    device.vm.box = "CumulusCommunity/cumulus-vx"
    device.ssh.host = "192.168.11.212"

    # Internal Networks
    device.vm.network "private_network",
      adapter: "2",
      virtualbox__intnet: "OpenStack_spine01-to-leaf02",
      auto_config: false

    device.vm.network "private_network",
      adapter: "3",
      virtualbox__intnet: "OpenStack_spine02-to-leaf02",
      auto_config: false

    device.vm.network "private_network",
      adapter: "4",
      virtualbox__intnet: "OpenStack_leaf02-to-control01",
      auto_config: false

    # VirtualBox Config
    device.vm.provider "virtualbox" do |vb|
      vb.name = "cumulus_leaf02"
      vb.memory = 1024
      vb.cpus = 1
      vb.customize ["modifyvm", :id, "--macaddress1", "080027000012"]
    end
  end

  # Leaf 3
  config.vm.define "cumulus_leaf03" do |device|
    device.vm.box = "CumulusCommunity/cumulus-vx"
    device.ssh.host = "192.168.11.213"

    # Internal Networks
    device.vm.network "private_network",
      adapter: "2",
      virtualbox__intnet: "OpenStack_spine01-to-leaf03",
      auto_config: false

    device.vm.network "private_network",
      adapter: "3",
      virtualbox__intnet: "OpenStack_spine02-to-leaf03",
      auto_config: false

    device.vm.network "private_network",
      adapter: "4",
      virtualbox__intnet: "OpenStack_leaf03-to-compute01",
      auto_config: false

    # VirtualBox Config
    device.vm.provider "virtualbox" do |vb|
      vb.name = "cumulus_leaf03"
      vb.memory = 1024
      vb.cpus = 1
      vb.customize ["modifyvm", :id, "--macaddress1", "080027000013"]
    end
  end

  # Leaf 4
  config.vm.define "cumulus_leaf04" do |device|
    device.vm.box = "CumulusCommunity/cumulus-vx"
    device.ssh.host = "192.168.11.214"

    # Internal Networks
    device.vm.network "private_network",
      adapter: "2",
      virtualbox__intnet: "OpenStack_spine01-to-leaf04",
      auto_config: false

    device.vm.network "private_network",
      adapter: "3",
      virtualbox__intnet: "OpenStack_spine02-to-leaf04",
      auto_config: false

    device.vm.network "private_network",
      adapter: "4",
      virtualbox__intnet: "OpenStack_leaf04-to-compute01",
      auto_config: false

    # VirtualBox Config
    device.vm.provider "virtualbox" do |vb|
      vb.name = "cumulus_leaf04"
      vb.memory = 1024
      vb.cpus = 1
      vb.customize ["modifyvm", :id, "--macaddress1", "080027000014"]
    end
  end

  ##################
  # OpenStack VM's #
  ##################

  # OpenStack Control
  config.vm.define "openstack_control" do |device|
    device.vm.box = "geerlingguy/ubuntu1604"
    device.ssh.host = "192.168.11.231"

    # Internal Networks
    device.vm.network "private_network",
      adapter: "2",
      virtualbox__intnet: "OpenStack_leaf01-to-control01",
      auto_config: false

    device.vm.network "private_network",
      adapter: "3",
      virtualbox__intnet: "OpenStack_leaf02-to-control01",
      auto_config: false

    # VirtualBox Config
    device.vm.provider "virtualbox" do |vb|
      vb.name = "openstack_control"
      vb.memory = 4096
      vb.cpus = 1
      vb.customize ["modifyvm", :id, "--macaddress1", "080027000031"]
    end
  end

  # OpenStack Compute
  config.vm.define "openstack_compute" do |device|
    device.vm.box = "geerlingguy/ubuntu1604"
    device.ssh.host = "192.168.11.232"

    # Internal Networks
    device.vm.network "private_network",
      adapter: "2",
      virtualbox__intnet: "OpenStack_leaf03-to-compute01",
      auto_config: false

    device.vm.network "private_network",
      adapter: "3",
      virtualbox__intnet: "OpenStack_leaf04-to-compute01",
      auto_config: false

    # VirtualBox Config
    device.vm.provider "virtualbox" do |vb|
      vb.name = "openstack_compute"
      vb.memory = 8192
      vb.cpus = 2
      vb.customize ["modifyvm", :id, "--macaddress1", "080027000032"]
    end
  end
end
```

# Conclusion

With this configuration file we have configured Vagrant to provision all of our VM's with all of the correct interface-to-adapter mappings.

While we could run `vagrant up` to ask Vagrant to provision and turn on the VM's, Vagrant won't finish as it will not be able to reach the hosts.  
Before it can do so, we need to configure a DHCP server to hand out addresses, which I'll cover in the next post.

## References:

[Vagrant - Getting Started](https://www.vagrantup.com/intro/getting-started/index.html)  
[Vagrant - Official Documentation](https://www.vagrantup.com/docs/)  
[VirtualBox Manual - Chapter 6: Virtual Networking](https://www.virtualbox.org/manual/ch06.html)  
[VirtualBox Manual - Chapter 8: VBoxManage](https://www.virtualbox.org/manual/ch08.html#vboxmanage-startvm)  
[Cumulus VX Documentation - Getting Started on VirtualBox](https://docs.cumulusnetworks.com/display/VX/VirtualBox)  
[Cumulus VX Documentation - Vagrant and VirtualBox](https://docs.cumulusnetworks.com/display/VX/Vagrant+and+VirtualBox)

##  Versions used:

Desktop Machine: *kubuntu-18.04*  
VirtualBox: *virtualbox-5.2.10*  
Vagrant: *2.1.2*  
Cumulus VX Vagrant Box: *CumulusCommunity/cumulus-vx (virtualbox, 3.6.2)*  
Ubuntu Server Vagrant Box: *geerlingguy/ubuntu1604 (virtualbox, 1.2.1)*   
