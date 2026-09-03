---
title: "NAT: Traversal"
---

part of the [[index|nat]] series.

zooming out from any one variant in [[01-introduction]], that "send first to open the mapping" trick is a general technique with a name: nat hole punching, for getting two devices behind nat(s) to talk directly instead of relaying everything through a third party.

a public rendezvous server both sides already know about tells each side the other's public ip:port, then both sides send a packet to each other at roughly the same time. each outbound packet opens or refreshes the local nat mapping, so the inbound packet from the other side is allowed through right as that mapping exists.

need it because a direct connection attempt without this just gets dropped: private ips arent reachable from outside, and neither side knows the other's public mapping ahead of time. hole punching gets p2p working without manually setting up port forwarding on either side.

> hole punching doesnt bypass nat, it works entirely within the rules nat already enforces. both sides just make sure they've each already "sent first" before the other side's packet arrives, so by the time it does, theres already a valid mapping sitting there ready to let it through. no router involved ever does anything it wasnt already going to do.

## learning your own public ip:port

that rendezvous server needs each side's public ip:port to hand out in the first place, and a host behind nat has no way to know that on its own, its private ip is all it sees locally. stun (session traversal utilities for nat) is how it finds out: send a public stun server a small formatted request (a stun binding request), and it reads back the source ip:port it actually saw the packet arrive from. thats it, one request, one response, no auth, no negotiation, which is exactly the (possibly-rewritten-by-nat) public mapping the host needs to hand to the rendezvous server.

## when stun isnt enough: symmetric nat

stun tells you your current mapping, but for symmetric nat that mapping is only good for the one destination it was learned against ([[01-introduction|the variant table]] covers why), the router hands out a different external port for every destination, so the port a stun server saw isnt the port thatll show up when you send toward the actual peer instead.

some routers allocate those ports predictably though, sequentially, or with a small fixed offset each time a new mapping gets created. probe a couple of them in a row (two different stun-like servers, say) and you can sometimes guess the *next* port well enough to aim a hole-punch packet at it before the peer's own mapping window closes. it doesnt always work, plenty of routers randomize the port allocation on purpose these days, mainly a security measure, a predictable mapping is easier for an off-path attacker to guess and hijack, but it happens to break port prediction just as well as it blocks anything else. once that happens no amount of probing gets you there. a sometimes-works trick, not a real fix.

## the fallback, and the real name for all of this

when port prediction doesnt pan out (or isnt even worth trying), turn (traversal using relays around nat) is the reliable fallback: a relay server in the middle forwards traffic for both sides instead of them ever talking directly. costs bandwidth on a third party and adds latency, direct is always better when it works, but turn doesnt depend on either side's nat cooperating at all, its just two ordinary client-to-server connections, which is what makes it reliable where hole punching isnt.

ice (interactive connectivity establishment) is the actual protocol name for tying all of this together instead of hand-rolling it: gather every way you might be reachable (your local address, your stun-derived public mapping, a turn relay address as a last resort), swap that candidate list with the other side over some signaling channel, then run connectivity checks on the candidate pairs concurrently rather than trying them one at a time and waiting on each. direct pairs get the highest priority (cheapest, fastest), hole-punched pairs next, relay pairs last since they cost a third party's bandwidth, whichever pair actually succeeds and ranks highest is the one that gets used. webrtc and most real p2p software do exactly this instead of implementing hole punching from scratch.
