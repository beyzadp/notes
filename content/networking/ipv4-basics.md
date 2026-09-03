---
title: "IPv4 Basics"
---

ipv4 is the addressing scheme the internet runs on: 32 bit addresses, 2^32 of them, ~4.3 billion, and thats the whole pool, theres no more where that came from.

it looks like 4 numbers separated by dots, but its actually a single 32-bit unsigned integer underneath, dotted decimal is just base-256 with each "digit" written back out as its own decimal number for readability:

> `192.168.1.1` as one 32-bit int: `192*256^3 + 168*256^2 + 1*256 + 1` = `3232235777`. (see [[network-operations|hex conversion and endianness]] for what happens once that same value crosses from network byte order into a machine that reads it the other way around.)

loopback isnt just the one address `127.0.0.1` either. the entire `127.0.0.0/8` block, `127.0.0.0` through `127.255.255.255`, 16 million addresses, is reserved for loopback traffic. pinging anything in that range routes straight back to the local machine, `127.0.0.1` is just the address everyone actually uses by convention.

its not the only chunk carved out of the address space for something other than "assign it to a random host on the internet" either, a handful of other ranges are reserved outright:

| range | purpose |
|---|---|
| `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` | private (RFC 1918), not routable on the public internet, this is the space [[nat/index\|nat]] hides behind |
| `169.254.0.0/16` | link-local, what a host picks for itself when DHCP fails |
| `0.0.0.0` | "no particular address" / default route |
| `255.255.255.255` | limited broadcast, everyone on the local segment |

every `192.168.1.x` example used in this note (and in the nat notes) is deliberately drawn from that private block, its address space thats assumed to only ever exist behind a router doing translation, never handed out as someone's actual public address.

## ipv4 exhaustion

that pool looked enormous once, but it wasnt built to last.

- ipv4 got designed in the early 80s, when the internet was a handful of research networks and universities. 4.3 billion addresses looked basically unlimited at that scale, nobody was planning for a world where billions of individual people would each carry a couple of internet-connected devices.
- allocation in the early days was generous to the point of waste: whole class A blocks (16 million addresses each) went out to single companies and universities that never came close to using them.
- by the time growth actually took off in the 90s/2000s there wasnt much slack left. iana handed out the last free /8 blocks in 2011.

what kept it from being a full blown crisis:

- **nat** (see [[nat/index]]): a whole home or office network hides behind one public ip
- **cidr** replacing the wasteful class system (below)
- **ipv6** existing as the actual long term fix, though adoption is still partial decades later

## classes vs cidr

before cidr, address space was split into fixed classes based on the first few bits of the address:

| class | prefix | networks | hosts each |
|-------|--------|----------|------------|
| A     | /8     | 128      | ~16M       |
| B     | /16    | ~16k     | ~65k       |
| C     | /24    | ~2M      | 254        |

two more classes existed past these three, D (`224.0.0.0/4`, multicast group addresses) and E (`240.0.0.0/4`, reserved/experimental, never handed out for general use). neither splits into network+host portions the way A/B/C do, which is why they dont fit as rows in the same table, theyre each just one flat block used a completely different way.

also worth noticing: class C says 254 hosts, not 256, even though a /24 covers 256 addresses. thats not a typo, the first address in any subnet (all host bits 0) is reserved as the network address itself, and the last one (all host bits 1) is the broadcast address for that subnet. neither is assignable to an actual host, so a /24 always has 254 usable addresses, not 256.

the problem is theres nothing in between. a company with 300 hosts doesnt fit in a class C (254 max) so they'd get bumped up to a class B (65k hosts), wasting almost all of it.

> what stuck with me: that jump from 254 to 65k with nothing in between is a big chunk of why exhaustion hit as fast as it did, blocks were handed out in chunks way bigger than what was actually needed, not because anyone was being careless with any single allocation.

cidr (classless inter domain routing) drops the fixed classes and lets the network/host split happen at any bit boundary, written as ip/prefix (`10.0.0.0/22`). the prefix length says how many bits are network, so you can carve out a block sized to what you actually need instead of rounding up to the nearest class. it also enables route aggregation (supernetting): a bunch of adjacent smaller blocks can get summarized as one line in a routing table instead of announcing each separately.

a `/prefix` is really just a compact way of writing a subnet mask, `/24` means the same thing as `255.255.255.0`, 24 bits set to 1 followed by 8 bits set to 0. to figure out which network an address belongs to, you `AND` the address against the mask bit by bit, whatever survives is the network portion, everything the mask zeroed out is the host portion.

> worked example: `192.168.1.0/24`, the /24 means 24 network bits, 8 host bits left, 2^8 = 256 addresses in that block (`192.168.1.0` - `192.168.1.255`), 254 of them actually usable once the network and broadcast addresses are set aside.
