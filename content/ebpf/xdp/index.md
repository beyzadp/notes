---
title: "XDP"
---

part of the [[../index|eBPF]] notes. [[../fundamentals|fundamentals]] covers the mechanics every ebpf program shares no matter which hook it targets, the verifier, maps, the load/attach lifecycle. this is what's actually specific to xdp as one particular hook.

## the parts

1. **[[what-is-xdp|What Is XDP]]**: the earliest hook in the rx path, why that makes drops nearly free, the four verdicts, attach modes, and the dispatcher that lets multiple programs share one interface.
2. **[[coding-xdp|Coding XDP]]**: building one xdp program up piece by piece, the skeleton and bounds-check idiom, compiling and loading it, then counting/limiting traffic with maps (race conditions, atomics vs per-cpu, per-source limits), replying with `XDP_TX`, a userspace loader, a blacklist, session tracking, and wildcard acl rules.
3. **[[maglev-load-balancer|Maglev Load Balancer]]**: what a load balancer actually is (l4/l7, inline/dsr), why xdp is the right hook for one, why naive hash-mod-n breaks on backend churn, and maglev's permutation-based lookup table, the algorithm, the math behind it, and why it keeps disruption minimal.
