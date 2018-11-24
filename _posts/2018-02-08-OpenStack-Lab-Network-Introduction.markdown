---
layout:     post
title:      "OpenStack Lab Network - Introduction"
subtitle:   ""
date:       2018-02-08
author:     "Michael Wadman"
catalog:    true
tags:
    - OpenStack
    - Neutron
    - Networking
    - BGP
    - VXLAN
    - EVPN
    - Cumulus
    - Vagrant
    - VirtualBox
---

# Overview

This post is the first in a series that plans to document my progress through installing and configuring an OpenStack Lab, focusing predominantly on the configuration of the network components.

The series will roughly look like the following:
1. [Setup VirtualBox and virtual machines using Vagrant.](https://wadman.co.nz/2018/04/08/OpenStack-Lab-Network-Vagrant/)
2. Install and configure ZTP server and boot switches.
  * [Part 1 - DHCP](https://wadman.co.nz/2018/06/08/OpenStack-Lab-Network-DHCP/)
  * [Part 2 - HTTP](https://wadman.co.nz/2018/08/03/OpenStack-Lab-Network-HTTP/)
3. Configure network underlay (eBGP) using Ansible
  * [Part 1 - Base Switch Configuration](https://wadman.co.nz/2018/11/18/OpenStack-Lab-Network-Switch-Base/)
  * [Part 2 - Interface Configuration](https://wadman.co.nz/2018/11/23/OpenStack-Lab-Network-Switch-Interfaces/)
4. Configure network overlay (VXLAN with EVPN) using Ansible.
5. Deployment of OpenStack servers using OpenStack-Ansible

If you're reading this sentence, the above is subject to change as I haven't yet written all posts yet.

# Subject

I've recently taken an interest in a few different topics.
These are, in no particular order:

* OpenStack: A cloud computing software platform. [Neutron](https://wiki.openstack.org/wiki/Neutron), the networking project, is especially interesting.
* Cumulus Linux: A network operating system that you can load onto 'white box' switches.
* Vagrant: Reproducible Virtual Machine deployment as code for labs.
* ZTP (Zero Touch Provisioning): Used to provision a network device in the same way that a server can be, using DHCP.
* VXLAN (Virtual Extensible LAN): A layer 2 overlay/tunnelling network protocol used primarily in data centres.
* EVPN (Ethernet VPN): A MP-BGP address family used to route layer 2 addresses, commonly used in conjunction with VXLAN for routable layer 2 data centre fabrics.

As I'm ~~a masochist~~ a person who likes to learn as much as possible when I'm not procrastinating, I thought it best to build a lab network that would use all above technologies and document my progress.

# Diagrams

I've drawn some quick diagrams that I'll be using when setting up the lab.

The [management network diagram](https://www.draw.io/?lightbox=1&amp;highlight=0000ff&amp;edit=_blank&amp;layers=1&amp;nav=1#R1Vpbc5s4FP41fmxGIK6Ptdttd2Y7mxnv9PYmgwyaYMQIOXb211cYyYAEieMQG%2FvFcHRD33d0bjCDi83%2BC0NF%2Bo3GOJvZIN7P4KeZbVuB7Yq%2FSvJUS3wb1IKEkVh2agRL8j%2BWQtVtS2JcdjpySjNOiq4wonmOI96RIcbortttTbPuqgVK5IqgESwjlGGj2w8S87SWBrbfyL9ikqRqZcsL65YVih4SRre5XG9mw%2FXhVzdvkJpLrlumKKa7lgh%2BnsEFo5TXV5v9AmcVtgq2etxfA63H52Y456cMsOsBjyjbyq0vC5JjIZJTlfxJgXLYFq4Gghmc71LC8bJAUdW6E2ogZCnfZOLOEpdrkmULmlF2GAtjFwexI%2BQlZ%2FQBt1oCewU9T7SYjy4f4REzjvctkdzKF0w3mLMn0UW22r5TD5Fqd4R515BoKVnaItCTMiT1JjlO3WAnLiR8%2FVBCA8p%2FMBK8A3iDSDoakg64IJLOEJK3qJPXBNIdAtK%2BQSCvimQ4hKRzg0h610QSgkGXc4taqRvKi7qcwLSUv%2F%2B7F4IlZmILo8K5DiIcRX1wrgLXccE4cIZdNC3HRDPoAdOCY6BpmsvvhPEtyuZ0L%2BRzEaomlZ5%2BjFHBe%2BAdBm9x%2BPWB5wWBNYeiJWEoJgI41ZZTcSYa8SfCRKBLaH5oYhUYcjIVoNqVJEVF9TCbfVIF5nc55jvKHsq71bY8b401bbo3wSycF5gRgTCu5FXQuxJT3TeyuZLJ2N4eST8s0FUQCE0F8bweDbHHUBDPUBBDB3Aef6wSkAbdllbgPeE%2Fq9N358q7X7JlEJqSblmEu8edI5Zg3lVaHKuMZgDAFkBuDz5KxnCGOHns5kF9oMkV7ikRT9w4aYWzbg7VFPV%2B5Kh2YqJP5HcnsnxtohoEY6IDh8dtn0ar%2F1Zac7FWi9fq9kVi2yTak%2BIQwoEz9loOoacpQ%2Fh%2BHAbXPZoKoolQ6OnIu2dS6L6kCyNSaMbYY1D4rONpUzh946qZRPfcc2mF3Ymgrh3jkapWuhap4fRJ1TymY53Jqq%2BpB9QzjBFZVXHblVh1J88qtEZiFQYaq%2B%2FnQ0P7uqzCybPq6KyeG906mn%2BG7xfdqoy6xepXWlbP8g1F6aHwonH8pirBGnv9VYLYD1dgpCzQ08sEit9LlAlCs0zwymNyTrrQyQN77J9CZCInxdNDDN0ZnezVHG0i3ZCOeFJel95HGSpLEp3O7eGmVSM53TL2Eu5PinDdNBq53smmUSP8WAJ8B8LNxP%2FvXFCTY24QP%2BHaXpTRbfxMqW4Egws1g%2BuZ9jZ8RrfeYm4t9XLiuWNZvXIvBjZ1%2FEoArVR3MADJ4P4%2F6Pm0554EgKG8ZzmcE2obF6ns90DbgRCAfhAv4pXN6sG%2FBc6XHEUPM%2FVCpLwt3I41765BvGCsY4GxMj11%2FavrG2tXCYDddpaiHbzgMDWv93IdYLAQcC2H%2BcHSMrvjW9hXl3OAFiKN5jHFbfMpUt29%2Bd4Lfv4D):

![OpenStack Lab Management Network](/img/openstack-lab-management.png)

The first network interface (as adapter 1 is Cumulus' management interface - "eth0") on all switches will connect to a bridged adapter on my host machine (which also has access out to the internet).

I'll also be creating three Ubuntu server 16.04 VM's - one ZTP server and two to take part in the OpenStack 'cloud'.
Each of the server VM's will have its' first network interface connected to the bridge adapter too.

And the [production network diagram](https://www.draw.io/?lightbox=1&amp;highlight=0000ff&amp;edit=_blank&amp;layers=1&amp;nav=1#R1VrLcpswFP0aL%2BsBJPFYNm6aLtLHjBdtlwrIhgYjRpZju19fEcRDwriEYIyzCTpCF%2Bncc6WjSWZgsTk8MJyGX2lA4pllBIcZ%2BDSzLMuwbfErQ445YpqGlSNrFgUSq4Bl9JdI0JDoLgrIVnmRUxrzKFVBnyYJ8bmCYcboXn1tRWP1qylekwaw9HHcRH9GAQ9z1LWcCv9ConVYfNm0vbznCfvPa0Z3ifzezAKr15%2B8e4OLWHKh2xAHdF%2BDwP0MLBilPH%2FaHBYkzsgtaMvHfW7pLefNSMK7DJBpecHxTi59mUYJEZAMteXHgpTXZZFsoDEDd%2Fsw4mSZYj%2Fr3QsdCCzkm1i0TPG4iuJ4QWPKXseCABE3gALfckafSa3HtZ6A0Au4a05dTuGFME4ONUgu5YHQDeHsKF4plGdLWqXuSpr3VRLNAgtrCSzGYambdRm64k48SPpOUwkaVD4SLPJugBtkEmpMQmNEJmEbk7eoyWsSidqItG6QyKsy6bUxCW%2BQSfuaTJqowRgJxOErm5TxkK5pguP7Cr1TOa3xRw4R%2FyXh7Pl39jx3UNZMxMzKvqxR6%2FxDOD9K34F3nAqo%2BvIjpamM38r2lu6YT5RDlGO2JlyBspWdzQgjMebRi2o73kPvic2zyXcSfMxMkmglNCEtjM4tVCdVYXRu1Tk1i8YPwiIxY8LkgE7swf7s1fSKTsi1wDqTLL%2Fwg0ZivmW5mEgtF2BpZZCvRo6qmy0tkOVCNZChBco5aAR6zXm57G4yaG79l5ABHE4GaOoyAKajZg95%2FWQAtO0XwMvJwBlFBvZwMgBTlwG0tMMT9NwNAPjPtjKgDNxRZOAOJwNv6jKwHVeVgdlTBkjTk%2BVdTAaF4E7c9m%2FxQqDfUce97TfvorduYwt51IuuwMY3smA4I%2Btczch2pm%2BcTavcW%2FSL31s3LTiekwXvdrKD2dUJZxaggTKLxjOnwB6lwi9rTiemA3uoCkfjuVMw3CXlnA4u604npgN3KB0449lTZDV08D0lyZZj%2F1nACyryReNYpGpIr7pyfeL7p7zqk4sgOquJ7l7VdFQay2227lWNE3ow9etlr78DvNtLVWWEGmVUVp9efJ1qqch6vZjgtGpJS51eAV1LydS2VHS5o7VQV2spbdIdz%2B5932hAbrCc0FXLyWuy2%2FfAOnNeNQqtdzmhSZUTMJCaOxPOUb%2BKgkgL5TZCDVdTXqHSCW6i9mn3oVmUa2Ydei0V%2B%2Bac685GDzRgxsc5Nm1QS%2FkHY24YrgR6mdNOWvCuqQUbDqQFR7%2F1DqYF0az%2BCSt%2FvfpXN3D%2FDw%3D%3D):

![OpenStack Lab Production Network](/img/openstack-lab-network.png)

After some initial research into both OpenStack and Cumulus (and using them together) I've decided to go with a leaf/spine network design.
I've chosen this to become familiar with the most common data centre network design out there in 2018.

The two OpenStack servers will each have two interfaces connected to two 'top of rack' switches, which they will use talk to using BGP to advertise their respective leaf switches.

I'll go over the more specific choices when we get around to configuring the switches.

# Conclusion

This was a quick post, meant as a foundation for the rest of the series and for me to document before I start configuring anything.

The [next post](https://wadman.co.nz/2018/04/08/OpenStack-Lab-Network-Vagrant/) in the series covers the use Vagrant to set up the environment in VirtualBox.
