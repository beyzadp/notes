---
title: "NAT: How It Works"
---

part of the [[index|nat]] series.

nat (network address translation) lets multiple devices on a private network share one public ip. the router sits in the middle and keeps a translation table mapping (private ip, private port) to (public ip, public port).

![[nat-translation-flow.png]]

**what it solves**: theres nowhere near enough ipv4 addresses for every device on earth to have its own public one. nat means a whole home or office full of machines only needs one. its a big part of why [[ipv4-basics|ipv4 exhaustion]] didnt just break the internet outright.

**what changes in the packet**: when a packet leaves the private network through the nat router, the source ip gets rewritten from the private address (`192.168.1.5`) to the router's public ip, and the source port gets rewritten too, to whatever port the router picks for that mapping, not necessarily the original one. the router remembers this mapping, so when a reply comes back addressed to (public ip, new port) it knows to rewrite it back and forward it to the right internal host.

> the actual insight: nat isnt free translation, its a stateful table the router has to maintain for as long as the connection lasts. without that mapping sitting in memory, an inbound reply would have no way to know which internal host it actually belongs to, the public ip alone isnt enough, it points at the whole router, not any one device behind it.

that table doesnt just map addresses one way though, it also decides who's allowed to send a packet back in through an existing mapping. the four nat variants differ in exactly one thing: how picky the router is about that. and that same pickiness is what decides how hard p2p hole punching is for each one.

| variant | who can send back in | p2p difficulty |
|---|---|---|
| full cone | any external host, to that public ip:port, doesnt matter who's sending | easy: once both sides know each other's public ip:port (from a rendezvous server), they can just send directly, the mapping accepts from anyone |
| restricted cone | only an external ip the internal host already sent something to (port on the external side doesnt matter) | each side has to send a packet toward the other first, to open the mapping for that ip before the other side's packet is let through |
| port restricted cone | only an external ip *and* port the internal host already sent to | same as restricted, but the outbound packet also has to match the exact port the other side sends from |
| symmetric | tied to (internal host, destination): the router picks a different external port for every destination | hardest: the external port changes per destination so you cant predict it in advance (not even from a packet to a stun server), pure hole punching usually doesnt work, need a relay (turn) instead |

> this taught me that the whole spectrum is really one variable, how much a mapping trusts "the same conversation" vs "the same external host" vs "nothing at all." symmetric is the strictest of the four, its mapping isnt per internal host anymore, its per (internal host, destination) pair, so the external port itself becomes unpredictable and you cant even learn it in advance from a stun server, since that same lookup would need a fresh mapping per destination too.

stun and turn are worth naming, since the table above leans on both. stun (session traversal utilities for nat) is how a host learns its own public ip:port in the first place, it asks a public stun server and reads back what the server saw the packet actually arrive from. turn (traversal using relays around nat) is the fallback for when hole punching cant work at all, typically symmetric nat on one side or both, a relay server in the middle forwards traffic for both sides instead of them ever talking directly.
