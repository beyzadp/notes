---
title: "eBPF"
---

ebpf lets you run small sandboxed programs inside the linux kernel without writing a kernel module or recompiling anything. the camp work so far has been on the networking side (xdp), but the same underlying mechanism, restricted c, a verifier, maps for state, shows up anywhere a hook exists: syscalls (lsm), tracepoints, the scheduler (sched_ext), even storage (xrp). more of those are probably getting their own notes here eventually, which is the reason this lives in its own folder instead of buried under networking.

## whats here

- **[[fundamentals|Fundamentals]]**: what ebpf actually is, the verifier, maps, and the load/verify/attach lifecycle every program goes through no matter which hook it ends up on.
- **[[xdp/index|XDP]]** (2 parts): the networking hook, what problem it solves, and building a program for it from the skeleton up through maps, `XDP_TX` replies, and wildcard acl rules.
