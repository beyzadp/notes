---
title: "Networking"
---

notes from a networking camp: what an ip address actually is under the hood, how a packet gets built and read back apart, nat and hole punching, why traceroute works, and the sharper edges (dns trust, byte order) that only really click once you've seen them go wrong.

## whats here

- **[[ipv4-basics|IPv4 Basics]]**: what an ipv4 address actually is under the hood (a single 32-bit int, not "4 numbers"), the loopback and private ranges, why the address space ran out, and classes vs cidr.
- **[[encapsulation|Encapsulation]]**: how app data actually turns into a frame on the wire, one header per layer, and the ip header fields that matter day to day.
- **[[ttl-and-traceroute|TTL & Traceroute]]**: why the ttl field stops packets from looping forever, how traceroute turns that into a hop-by-hop map, and the propagation-delay floor under every rtt.
- **[[network-operations|Network Operations]]**: the hex-conversion trick behind `ping 2131000000`, the actual tools (wireshark/tcpdump/dig), dns trust (poisoning, dot/doh), ixps, and endianness gotchas.
- **[[addressing-authorities|Addressing Authorities]]**: the iana -> RIR -> isp -> end customer hierarchy behind where an address like `saddr` comes from, ietf/rfcs as the separate standards side, and dn42 as a hands-on reference.
- **[[nat/index|NAT]]** (2 parts): what nat actually does to a packet, the four nat variants and what each means for p2p, and hole punching as the general technique for getting two nat'd hosts talking directly.
- **[[ebpf/xdp/index|XDP]]**: kernel-level packet handling, the networking application of [[ebpf/index|eBPF]], from what xdp actually is up through building a full program: maps, `XDP_TX` replies, and wildcard acl rules.
