---
title: "TTL & Traceroute"
---

the ip header field table in [[encapsulation]] had one line for ttl: hop limit, decremented per router. thats the short version. the long version is what actually keeps a packet from wandering the network forever, and its also the entire mechanism traceroute is built on top of.

ttl is specified in seconds technically (anything under 1 second just rounds up to 1), but in practice every hop treats it as a hop count instead: each router that forwards the packet decrements ttl by one, and once it hits zero the router drops the packet and sends an icmp time exceeded message back to whoever sent it. thats the actual purpose of the field, stopping a packet from looping the network forever if something along the path is misconfigured into a routing loop.

> you never see what ttl a packet started at, only what it arrived with, since every hop along the way decremented it. different OSes start at different defaults though, linux at 64, windows at 128, some routers at 255, so `64 - observed_ttl` (rounded up to the nearest common default) is a rough way to guess how many hops a packet traveled, or even fingerprint the sending OS, from a single captured packet.

traceroute is built entirely on that behavior: it sends packets with increasing ttl values and reads the icmp time exceeded replies to map out every router along the path to the destination.

| ttl sent | what happens | reply comes from |
|---|---|---|
| 1 | first router decrements it to 0, drops it, replies with icmp time exceeded | hop 1 |
| 2 | first router forwards fine (ttl now 1), second router drops it at 0 and replies | hop 2 |
| n | keep incrementing | hop n, until the destination itself replies |

> how traceroute actually knows its done depends on what it sends. linux's `traceroute` defaults to udp aimed at a high, unlikely port, so the destination replies with icmp "port unreachable" instead of "time exceeded", thats the signal its the last hop. windows' `tracert` sends icmp echo requests instead, so the destination just sends back a normal echo reply. `traceroute -T` switches to tcp entirely, useful when something along the path drops icmp/udp but lets tcp through.

> turns out traceroute doesnt need any special protocol support anywhere along the path, it just exploits behavior every router already has to do the moment ttl hits zero. no router is aware its being traced, its just doing what it would do anyway.

each ttl value maps to exactly one router along the path. if some hop silently drops icmp instead of replying, that shows up as `* * *` in the output, you still get the hops after it as long as they dont also drop.

> a `* * *` is usually deliberate, not incidental. plenty of routers and security devices are configured to not generate icmp time exceeded at all, or to rate-limit it heavily, specifically so they dont show up in traceroutes or get used to map the network.

## propagation delay

even with every hop replying, theres a hard floor under how fast any of those replies can come back: the speed of light. light travels about 30 cm per nanosecond in a vacuum. in fiber its slower, closer to 20 cm/ns, since light bends and slows down going through glass instead of empty space (refractive index of common fiber is around 1.5, speed drops by roughly that factor).

either number is a hard physical floor under every rtt a traceroute prints, no amount of better hardware moves a packet from new york to tokyo faster than the distance divided by that speed.

> worked example: ~10,800 km one way over fiber at ~20 cm/ns is already about 54 ms of pure propagation delay one way, so ~108 ms round trip, before any router processing, queuing, or the destination doing anything at all.

the hundreds-of-ms rtts you see over intercontinental links are mostly just this, distance, not slow equipment along the way.
