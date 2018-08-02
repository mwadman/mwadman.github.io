---
layout:     post
title:      "Cisco iBGP vs. eBGP Route Selection"
subtitle:   "How route selection works within a route process"
date:       2016-11-03
author:     "Michael Wadman"
catalog:    true
tags:
    - Cisco
    - BGP
---

The other day my colleague stumbled upon a question in his studies (towards the CCNP route exam) and posed it to me. Initially I had no clue what the answer was, but figured it would make for a good topic of a first post.

# The Question
Say you administer a network with Cisco routers that uses BGP to peer with two upstream providers (eBGP) who both advertise you the same destination (i.e. a default route, 0.0.0.0/0). You also use BGP as your internal routing protocol (iBGP) and are re-advertising routes to all internal peers. I've included a quick diagram below.

![iBGP-vs-eBGP](/img/ibgp-vs-ebgp.png)

Given the above, you as the admin want to use AS200 as the primary exit point for all traffic. How would you complete this?
# The Problem
Thinking about the problem and looking at the diagram, my first thought (and my mistake, which I'll elaborate on later) was to compare the AD ([administrative distance](http://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/15986-admin-distance.html#topic2)) values of the routes received, since the length of the route received from both the iBGP and eBGP peers is the same and because I remembered with Cisco they have different values; with eBGP (AD of 20) being preferred over iBGP (AD of 200) as a lower value is preferred over a higher one.

With that thought process, I decided that the routes received via eBGP would always be preferred as the lower AD would take preference, and so the only way to make the router connected to AS300 use its' iBGP path would be to change the AD of iBGP to be lower (and therefore better) than that of eBGP, right?
# Testing

## Initial Setup
I've set up GNS3 with the topology above and have given the routers serial links between each other. I'm using Cisco 7200's as my virtual routers and am running 15.2(4)M7 code.

![iBGP-vs-eBGP-GNS](/img/ibgp-vs-ebgp-gns.png)

For addressing I've gone with a /24 between each device in the format 192.168.x.y/24, where x is the interface number and y is the router number. So R1's address on the interface to R2 would be 192.168.1.1/24:

```
R1(config)#interface serial 2/0
R1(config-if)#no shut
R1(config-if)#ip address 192.168.0.1 255.255.255.0
R1(config-if)#interface serial 2/1
R1(config-if)#no shutdown
R1(config-if)#ip address 192.168.1.1 255.255.255.0
```

Next we'll configure BGP on each of the devices. Don't forget to configure `next-hop-self` on the routers we control otherwise we won't be able to reach the routes that are re-advertised by our routers. I've again shown the configuration of R1:

```
R1(config)#router bgp 100
R1(config-router)#neighbor 192.168.1.2 remote-as 100
R1(config-router)#neighbor 192.168.1.2 next-hop-self
R1(config-router)#neighbor 192.168.0.3 remote-as 200
R1(config-router)#neighbor 192.168.0.3 next-hop-self
```

And lastly we'll confirm that all peers are up before we start advertising anything:

```
R1#show ip bgp summary
BGP router identifier 192.168.1.1, local AS number 100
BGP table version is 1, main routing table version 1

Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
192.168.0.3     4          200       4       4        1    0    0 00:00:04        0
192.168.1.2     4          100       7       7        1    0    0 00:03:02        0
```

```
R2#show ip bgp summary
BGP router identifier 192.168.2.2, local AS number 100
BGP table version is 1, main routing table version 1

Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
192.168.1.1     4          100       9       9        1    0    0 00:04:35        0
192.168.2.4     4          300       4       4        1    0    0 00:00:33        0
```

Next we will advertise the default route from R3 and R4. Configuration of R3 below:

```
R3(config)#router bgp 200
R3(config-router)#neighbor 192.168.0.1 default-originate
```

We can now check that both routers are receiving the advertisements:

```
R1#show ip route 0.0.0.0
Routing entry for 0.0.0.0/0, supernet
 Known via "bgp 100", distance 20, metric 0, candidate default path
 Tag 200, type external
 Last update from 192.168.0.3 00:15:11 ago
 Routing Descriptor Blocks:
 * 192.168.0.3, from 192.168.0.3, 00:15:11 ago
 Route metric is 0, traffic share count is 1
 AS Hops 1
 Route tag 200
 MPLS label: none
```


```
R2#show ip route 0.0.0.0
Routing entry for 0.0.0.0/0, supernet
 Known via "bgp 100", distance 20, metric 0, candidate default path
 Tag 300, type external
 Last update from 192.168.2.4 00:08:40 ago
 Routing Descriptor Blocks:
 * 192.168.2.4, from 192.168.2.4, 00:08:40 ago
 Route metric is 0, traffic share count is 1
 AS Hops 1
 Route tag 300
 MPLS label: none
```

Both routers are preferring the route that they received via their external peer, which is what I expected.

## Modifying the AD
Now that we have everything set up, lets attempt to use AS200 as the preferred path on both routers.

As we already know this is the case on R1, we simply need to change the administrative distance on R2 for routes received from R1. We can do this by using the command `distance 19 192.168.1.1 0.0.0.0` which sets all routes received from the neighbour 192.168.1.1 with an AD of 19 (and therefore better than the eBGP AD of 20)

```
R2(config)#router bgp 100
R2(config-router)#distance 19 192.168.1.1 0.0.0.0
```

But wait, this hasn't changed the preferred route?

```
R2#show ip route 0.0.0.0
Routing entry for 0.0.0.0/0, supernet
 Known via "bgp 100", distance 20, metric 0, candidate default path
 Tag 300, type external
 Last update from 192.168.2.4 00:45:42 ago
 Routing Descriptor Blocks:
 * 192.168.2.4, from 192.168.2.4, 00:45:42 ago
 Route metric is 0, traffic share count is 1
 AS Hops 1
 Route tag 300
 MPLS label: none
```


# The Solution
To figure out why this happens, we need to think about how a route is installed in the routing table. There are several parts to this process:

- Routing processes (protocols) determine the best route for each destination (using their metric) and attempt to install these into the routing table.

- The router then receives all routes attempting to be installed by all processes and if there are any conflicting routes (routes to the same destination) then AD is used as a tie breaker before placing one into the routing table to be used for forwarding.

Simply put, AD is only evaluated when there are multiple paths to the same destination attempting to be installed by more than one routing process. So in our situation, because iBGP and eBGP are a part of the same process on a Cisco router, the BGP process chooses only one of the 0.0.0.0/0 routes to attempt to install into the routing table and therefore AD is not evaluated.

However, if we did have the same route (0.0.0.0/0) being put forward by another routing process, i.e. OSPF, AD would be the tie breaker.

## So how *does* BGP choose the best path then?
Going back to our example; before our routers can install the route 0.0.0.0/0 into their routing table, it needs to be received by the BGP process which picks the route to install by the attributes that a route has.

In our case, BGP has gone through the route [selection process](http://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/13753-25.html) and has reached step 7, where routes learnt by eBGP are preferred over routes learnt via iBGP.

I won't cover how BGP makes its' route selection here as that is another topic entirely. All you need to know is that BGP's metric is actually a lot of different metrics (called attributes) that are compared one by one with other contender routes (other routes to the same destination received from other peers) until there is a difference and the tie is broken for a best route to be chosen.

To solve our problem, we can use the metric called 'local preference'. This one is number 2 on the list of attributes, meaning it is looked at second (after a Cisco proprietary attribute called weight) to see if it can be used as a tie breaker.

We can check the local preference of each route that we have received using the command `show ip bgp`:

```
R2#show ip bgp 0.0.0.0
BGP routing table entry for 0.0.0.0/0, version 4
Paths: (2 available, best #2, table default)
 Advertised to update-groups:
 9
 Refresh Epoch 1
 200
 192.168.1.1 from 192.168.1.1 (192.168.1.1)
 Origin IGP, metric 0, localpref 100, valid, internal
 Refresh Epoch 1
 300
 192.168.2.4 from 192.168.2.4 (192.168.2.4)
 Origin IGP, localpref 100, valid, external, best
```

Here we can see that the local preference of both routes received is 100 (lines 9 and 13), which is also the default local preference value for all BGP routes.

In order to make the local preference for one of our routes higher than the other (a higher number is preferred with local preference), we would apply a route-map to the neighbor we want to change. For this problem I'll set the local preference of the peering to AS200 higher than 100 to prefer this over all other peers.

```
R1(config)#route-map set-lp-120
R1(config-route-map)#set local-preference 120
R1(config-route-map)#exit
R1(config)#router bgp 100
R1(config-router)#neighbor 192.168.0.3 route-map set-lp-120 in
```

We need to clear the neighbor for this to take effect as well:

```
R1#clear ip bgp 192.168.0.3
R1#
*Nov  3 16:28:37.946: %BGP-5-ADJCHANGE: neighbor 192.168.0.3 Down User reset
*Nov  3 16:28:37.950: %BGP_SESSION-5-ADJCHANGE: neighbor 192.168.0.3 IPv4 Unicast topology base removed from session  User reset
*Nov  3 16:28:38.738: %BGP-5-ADJCHANGE: neighbor 192.168.0.3 Up
```

Now, when we check BGP, we can see that it has chosen the iBGP route instead:

```
R2#show ip bgp 0.0.0.0
BGP routing table entry for 0.0.0.0/0, version 11
Paths: (2 available, best #1, table default)
  Advertised to update-groups:
     5
  Refresh Epoch 1
  200
    192.168.1.1 from 192.168.1.1 (192.168.1.1)
      Origin IGP, metric 0, localpref 120, valid, internal, best
  Refresh Epoch 1
  300
    192.168.2.4 from 192.168.2.4 (192.168.2.4)
      Origin IGP, localpref 100, valid, external
```

Lastly, AD comes back into play when the route is installed into the routing table. If the installed route was learnt from an eBGP peer, the route would have an AD of 20 and if it was instead learnt by iBGP it would instead have an AD of 200.

If we check the routing table on R2, we can see this has now been installed; with an AD of 19 that we set earlier as well:

```
R2#show ip route 0.0.0.0
Routing entry for 0.0.0.0/0, supernet
 Known via "bgp 100", distance 19, metric 0, candidate default path
 Tag 200, type internal
 Last update from 192.168.1.1 00:02:13 ago
 Routing Descriptor Blocks:
 * 192.168.1.1, from 192.168.1.1, 00:02:13 ago
   Route metric is 0, traffic share count is 1
   AS Hops 1
   Route tag 200
   MPLS label: none
```
