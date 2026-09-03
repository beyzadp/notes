---
title: "NAT: Traversal"
---

part of the [[index|nat]] series.

zooming out from any one variant in [[01-introduction]], that "send first to open the mapping" trick is a general technique with a name: nat hole punching, for getting two devices behind nat(s) to talk directly instead of relaying everything through a third party.

a public rendezvous server both sides already know about tells each side the other's public ip:port, then both sides send a packet to each other at roughly the same time. each outbound packet opens or refreshes the local nat mapping, so the inbound packet from the other side is allowed through right as that mapping exists.

need it because a direct connection attempt without this just gets dropped: private ips arent reachable from outside, and neither side knows the other's public mapping ahead of time. hole punching gets p2p working without manually setting up port forwarding on either side.

> the takeaway: hole punching doesnt bypass nat, it works entirely within the rules nat already enforces. both sides just make sure they've each already "sent first" before the other side's packet arrives, so by the time it does, theres already a valid mapping sitting there ready to let it through. no router involved ever does anything it wasnt already going to do.
