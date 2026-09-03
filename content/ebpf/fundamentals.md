---
title: "eBPF Fundamentals"
---

part of the [[index|eBPF]] notes. the mechanics here apply no matter which hook a program ends up attached to, [[xdp/index|xdp]] is just the first one covered.

ebpf lets you run small sandboxed programs inside the kernel without a kernel module or a recompile. a program attaches to a hook and runs in kernel context when that hook fires. [[xdp/index|xdp]] is one such hook, sitting on the network rx path, but the exact same mechanism attaches just as easily to syscalls, tracepoints, or the scheduler, not just packet filtering despite the name.

> **LSM** (Linux Security Module): a set of kernel hooks originally built for security modules like selinux/apparmor. ebpf can attach to the same hooks to enforce custom security policy, gating syscalls and other security-relevant kernel operations instead of packets, same verifier/hook mechanism as xdp underneath. likely the next hook type to get its own notes here.
> **BPF SCX** (`sched_ext`): a linux cpu scheduler framework built on ebpf, same underlying mechanism (hooks + verifier) as xdp but a completely different topic.
> **XRP** (eXpress Resubmission Path): a linux storage acceleration framework that uses ebpf to run storage functions inside the nvme driver, bypassing most of the kernel storage stack. just noted here as an example of how far past networking the same mechanism reaches.

a program tells the loader which hook it wants through a section name: `SEC("xdp")` for xdp, `SEC("lsm/bprm_check_security")` for a specific lsm hook, `SEC("kprobe/do_sys_open")` for a kprobe, and so on, one compiled object can even define several of these at once. thats a libbpf convention though, not a kernel rule, the loader reads the name back out of the elf and uses it to guess the program's type (`BPF_PROG_TYPE_XDP`, etc) before calling `BPF_PROG_LOAD`. the kernel only checks that type at attach time, not the section name itself, so a section named something else entirely still attaches fine as long as whatever loads it sets the type correctly some other way. same mechanism as `SEC(".maps")` below, just naming a program's purpose instead of a map's.

getting a program to actually run in the kernel isnt just "compile it and load it" though, theres a gate in front of that.

## the pipeline

you write restricted c, compile it to bpf bytecode with clang (`-target bpf`), and load it with a syscall.

> llvm is the compiler toolchain behind `clang -target bpf`, clang is llvm's c frontend, the actual bpf bytecode generation happens in llvm's backend.

before getting into each step, its worth seeing them all lined up on one concrete example first. say you write a program that just counts packets in a map: `clang` compiles it to bytecode, the loader creates that map and patches its fd into the bytecode, `BPF_PROG_LOAD` runs the verifier against the whole thing, once its satisfied the bytecode gets jit'd to native machine code, and the program just sits there, loaded but doing nothing, until something attaches it to a hook. the moment that hook fires, on a packet arriving for xdp, on a syscall entry for lsm, the program runs, looks up the map, increments it, and returns. userspace can read that counter back out of the map whenever it wants, completely independent of whether the program is even running right now. everything below is one of those steps in more detail.

**loading** is where that compiled bytecode turns into a live kernel object, regardless of which hook its headed for. the loader opens the elf and walks its `SEC(".maps")` definitions, issuing a `BPF_MAP_CREATE` syscall for each one (whats actually created there is covered in `## maps` below, for now: its where that packet counter from the example above comes from), then patches the prog's bytecode with the map fds it just got back (the elf only holds symbolic references to maps at compile time, the actual fd cant be known until the map exists in the kernel). once maps are wired up it calls `BPF_PROG_LOAD` with the patched bytecode.

**the verifier** runs synchronously inside that `BPF_PROG_LOAD` call, before the syscall returns. its a static analyzer, it never actually executes the program, it walks every possible path through the bytecode and proves each one is safe: no unbounded loops (cant hang the kernel), and every pointer access into memory it doesnt control has to be provably in bounds first. once its satisfied, the bytecode gets jit compiled to native machine code for the running architecture, so it runs as fast as compiled c instead of being interpreted, and only then does `BPF_PROG_LOAD` return a valid prog fd.

> what stuck with me: the verifier isnt a runtime safety net, its a proof requirement that happens once, before the program ever runs a single time. a program that would behave perfectly safely at runtime still gets rejected outright if the verifier cant prove it ahead of time, it doesnt get the benefit of the doubt. thats the actual reason bounds-checking a header pointer against `data_end` before touching it isnt optional style, its the only way to give the verifier something it can prove.

put together, the whole execution model looks like this:

![[bpf-internals.png]]

bytecode goes into the verifier first, no exceptions. once it passes, it either gets jit compiled to native machine code (the normal case, and the reason a bpf program runs about as fast as compiled c) or falls back to being interpreted, on architectures or configs where jit isnt available. either way, execution gets two things nothing outside the sandbox can touch directly: 11 registers to work with, and the map storage covered below (labeled "Mbytes" in the diagram for a reason, a map's size is fixed and capped up front when its created, its not a heap that grows as you go, `max_entries` is a hard limit, not a hint). `bpf helpers` are the only way out of that sandbox, a fixed set of kernel functions the verifier already knows the exact signature and behavior of, so a program can do genuinely useful things (read a map, grab the current time, redirect a packet) without being handed arbitrary access to the rest of the kernel. `bpf context` is whatever triggered the run in the first place, an incoming packet for xdp, a syscall entry for an lsm hook, the actual shape of it depends on which hook the program is attached to.

**attaching** is hook-specific, and happens separately from loading, a program can sit loaded in the kernel with an id and do nothing until something actually attaches it to a hook. xdp does this over netlink, other hook types have their own mechanism, thats covered in each hook's own notes rather than here.

## maps

maps are how state persists across invocations, since the program itself is stateless per call, each run starts fresh with nothing carried over except through a map. the same map is reachable from both sides: helpers like `bpf_map_lookup_elem`/`bpf_map_update_elem` from inside the program, `bpftool map dump` or the equivalent library calls from userspace outside. thats the actual point of a map, its how a counter or any other piece of state survives across calls and how you read it back out afterward.

`bpftool map dump id 161` (or `prog show`, same idea) is worth pausing on: `161` there isnt a file descriptor, its a kernel-wide id, stable and listable independent of any process. every loaded map and program gets one the moment `BPF_MAP_CREATE`/`BPF_PROG_LOAD` succeeds, and that object stays alive in the kernel as long as *something* is holding a reference to it, an attached hook, an open fd somewhere, or a pin. pinning is the explicit version of that: `bpftool map pin` (or the equivalent library call) attaches a map or program to a path under `bpffs` (usually mounted at `/sys/fs/bpf`), so it survives and stays reachable by name even after the process that loaded it is long gone, same idea as a file on disk outliving the process that wrote it.

## debugging: bpf_printk and trace_pipe

`bpf_printk(fmt, ...)` (from `bpf/bpf_helpers.h`) is a helper that formats a string and writes it into the kernel's trace ring buffer, the closest thing to a debug `printf` thats actually usable inside a verifier-restricted program, since a normal `printf` doesnt exist in this context, no libc, no stdout to write to. it only takes a handful of format args and the format string itself eats into the prog's stack budget, so its meant for quick "is this branch even hit" checks while writing a program, not for anything thats meant to ship.

output doesnt show up on the terminal that loaded the program, it goes to a shared kernel trace buffer instead, read with:

```
sudo cat /sys/kernel/debug/tracing/trace_pipe
```

`trace_pipe` is a live streaming read, blocks and prints new lines as they arrive, same idea as `tail -f`. its shared across every bpf program on the system currently using `bpf_printk`, not scoped to just the one prog youre debugging, so lines from something else on the box also using it show up mixed in.

## bcc

a separate tracing ecosystem from the libbpf/loader path above: [bcc](https://github.com/iovisor/bcc/blob/master/docs/tutorial_bcc_python_developer.md) (bpf compiler collection) still writes the kernel side in restricted c, same language, same verifier, but drives loading/attaching/reading maps from a python script instead of a compiled c binary. it compiles that c on the fly at runtime through its own embedded clang, no separate `clang -target bpf -c foo.c -o foo.o` build step. bcc's classic use case is attaching to syscalls system-wide with kprobes/uprobes/tracepoints, e.g. tracing every `execve` or `open` call on the machine as it happens, a good fit for a quick one-off tracing script, not really the shape of thing youd attach permanently the way an xdp prog sits on an interface.

bcc ships with a whole library of these ready-made, each one just a small ebpf program plus a python wrapper, mapped here to the exact kernel subsystem each one taps into:

![[bcc-bpf-tracing-tools.png]]

worth just sitting with this one for a second: every single tool on that diagram is the same three things covered above, restricted c, a verifier, a hook, just aimed at a different kprobe/tracepoint/syscall. `execsnoop` hooks process creation, `tcplife` hooks tcp socket lifecycle, `biolatency` hooks the block device layer, none of them are special cases, theyre all just ebpf programs someone already wrote so you dont have to.
