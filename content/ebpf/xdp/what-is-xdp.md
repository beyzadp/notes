---
title: "What Is XDP"
---

the load -> verify -> jit pipeline in [[../fundamentals|fundamentals]] is the same no matter which hook a program ends up on. xdp is the first hook actually covered here, and its where all of that turns into packet filtering.

xdp is the earliest hook in the rx path, it runs before the kernel even builds an skb for the packet. that timing is the whole point: a drop costs basically nothing, no skb allocation, no walking the rest of the network stack, which is why xdp is what gets used for ddos-limiting/dropping instead of doing it later with iptables or nftables.

ebpf attaches at plenty of other points too, and seeing them stacked up against the actual network layers they sit at makes it obvious why xdp specifically is the fast one:

![[bpf-hooks-in-the-stack.png]]

xdp sits at the very bottom, on the netdevice/driver itself, before a packet is anything more than raw bytes in a buffer. tc hooks sit one step up, after traffic shaping, still cheap but already past the point xdp intercepts at. cgroups and sockmap/sockops sit much higher, at layer 3 and up at the socket layer, by which point the kernel has already done all the work xdp exists to skip: building the skb, running it through netfilter, making a routing decision. none of that has happened yet when an xdp program runs, which is exactly why a drop at this point is nearly free and a drop anywhere higher up the stack isnt.

> compared to something like dpdk (bypasses the kernel entirely with a userspace polling driver, faster ceiling but you lose the kernel network stack), xdp stays inside the kernel. `XDP_PASS` still hands the packet to the normal stack whenever you dont want to touch it, you're not opting out of the kernel network stack, just getting a chance to intercept packets before it.

> further reading: [wikipedia](https://en.wikipedia.org/wiki/Express_Data_Path), the [original xdp presentation slides](https://github.com/tohojo/xdp-paper/blob/master/xdp-presentation.pdf), and the [xdp-tutorial repo](https://github.com/xdp-project/xdp-tutorial/tree/main/basic01-xdp-pass#first-step-setup-dependencies) for a hands-on walkthrough starting from an `XDP_PASS`-only program.

## the context: xdp_md

every xdp program gets handed the same context struct:

```c
struct xdp_md {
	__u32 data;
	__u32 data_end;
	__u32 data_meta;
	/* Below fields are only available in newer kernels */
	__u32 ingress_ifindex;
	__u32 rx_queue_index;
	__u32 egress_ifindex;
};
```

`data`/`data_end` bound the actual packet bytes, `data_meta` is a small scratch area a program can use to pass data to whatever runs after it, and the last three (ingress/rx queue/egress ifindex) are metadata about where the packet arrived, not always present depending on kernel version.

## the four verdicts

whatever a program returns decides what happens to the packet immediately:

| verdict | what happens |
|---|---|
| `XDP_PASS` | dispatcher moves on to the next chained program, or if its the last one, the kernel finally allocates an skb and hands the packet to the normal network stack, same as if xdp wasnt involved at all |
| `XDP_DROP` | frees the packet right there, no skb ever gets built |
| `XDP_TX` | retransmits whatever's currently in the buffer back out the same interface/queue it arrived on. doesnt rewrite anything on its own, so bouncing a packet back out correctly means manually swapping mac/ip addresses first |
| `XDP_REDIRECT` | sends the packet out a different interface, or into a userspace socket (af_xdp) instead |

## attach modes

three ways a program can actually attach to an interface: native (driver level, needs driver support, fastest), generic/skb (works on any driver, but the kernel builds an skb first, so its slower), or hardware offload (runs on the nic itself).

attaching itself goes through netlink (`RTM_SETLINK` with an `IFLA_XDP` attribute carrying the prog fd), not another `bpf()` syscall, thats a separate mechanism from loading. once a program is attached, the interface holds its own kernel reference to it, independent of whatever process loaded it, which is why the program keeps running after the loader process exits.

## the dispatcher

a network interface can only have one native xdp program attached directly. tools like `xdp-loader` (and the underlying libxdp) work around that by never attaching your program directly, instead they generate and attach a small wrapper program called `xdp_dispatcher`, and your actual program gets loaded as a "sub-program" the dispatcher calls into, at a priority you choose.

> the real lesson: `xdp-loader status` always showing `xdp_dispatcher` as the top-level attached program, with your own programs listed underneath it, isnt a quirk of the tool. its the actual mechanism that lets several independent xdp programs run on the same interface at once, when the raw xdp hook only ever allows one.

## how a packet actually moves through this

a packet arrives, the nic dma's it into a buffer, and before the kernel does anything else with it (no skb allocation, no walking the normal network stack) the driver calls the xdp hook if one is attached. control goes to `xdp_dispatcher` first, which calls each attached sub-program in priority order, passing along the same `xdp_md` context, the same `data`/`data_end`/`data_meta` fields covered above, to each one. whatever verdict a program returns decides what happens next, per the table above, right there, before the kernel would otherwise have spent any time on the packet at all. thats the actual performance argument for doing this at the xdp layer instead of filtering later in the stack.

drawn out, the kernel/userspace boundary and the verdicts line up like this:

![[xdp-hook-diagram.png]]

`skb alloc` only happens on the `PASS` path, thats the cost this whole hook exists to let you skip. `DROP` and `REDIRECT` never reach it at all, `DROP` just ends there, `REDIRECT` can send the packet straight back out a nic (as drawn) or into af_xdp (below), bypassing the normal network stack box entirely either way.

`XDP_REDIRECT` into af_xdp is worth one more note: it hands packets to a userspace socket, letting an application read them straight out of the nic's buffer without the kernel network stack touching them at all. not covered in depth here, just worth knowing the name for when `XDP_REDIRECT` shows up.
