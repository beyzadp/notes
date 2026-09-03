---
title: "Coding XDP"
---

part of the [[index|xdp]] series. [[what-is-xdp|what-is-xdp]] covered what the four verdicts mean and how a packet actually reaches an xdp program, [[../fundamentals|fundamentals]] covered the verifier and maps in general. this is the part where all of that turns into an actual program: building one up piece by piece, starting from basically nothing and adding one capability at a time.

## the skeleton

every xdp program starts from the same shape, no packet logic yet:

```c
#include <linux/bpf.h>
#include <linux/if_ether.h>
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_endian.h>

SEC("xdp")
int my_prog(struct xdp_md *ctx)
{
	return XDP_PASS;
}

char _license[] SEC("license") = "GPL";
```

`SEC("xdp")` is the section name the loader reads back out of the elf, covered generically in [[../fundamentals|fundamentals]], `xdp` specifically is what tells it "this one attaches to the xdp hook." the license line at the bottom is boilerplate every program needs, gpl-compatible bpf helpers refuse to load into a non-gpl program otherwise, and the compiler wont warn you about it, the rejection only shows up at load time.

> only the `SEC("license")` tag actually matters here, the variable name (`_license`) is arbitrary, the loader never reads it by name. worth knowing since its spelled `_licence` in a lot of code floating around, that typo genuinely never breaks anything.

passes literally everything through right now, thats the point, its just the shape to build on top of. it also compiles and loads as-is, nothing here touches packet data yet so theres nothing for the verifier to reject, that only becomes a concern once we start reading something, next section.

## pulling the packet out of ctx

`ctx` doesnt hand you a pointer to the packet directly:

```c
void *data = (void *)(long)ctx->data;
void *data_end = (void *)(long)ctx->data_end;
```

`ctx->data`/`ctx->data_end` are `__u32` in the actual `xdp_md` struct (see [[what-is-xdp|the full struct]]), not pointers, so they get cast through `long` first to widen to pointer size before becoming a real `void *`. `data` is where the packet starts, `data_end` is one past where it ends, everything the program is allowed to touch sits between the two.

## the bounds-check idiom

the obvious next step is casting `data` straight to a header struct and reading it:

```c
struct ethhdr *eth = data;

if (eth->h_proto == bpf_htons(ETH_P_IP))
	return XDP_DROP;
```

the verifier rejects this at load time. it never runs the program to find out if `eth` is actually in bounds, it has to be *proven* in bounds ahead of time (the "proof requirement" from [[../fundamentals|fundamentals]]), and nothing here proves anything, this reads whatever's at `data` regardless of how much packet actually arrived. the fix is a check before the read, not after:

```c
struct ethhdr *eth = data;

if ((void *)(eth + 1) > data_end)
	return XDP_PASS;

if (eth->h_proto == bpf_htons(ETH_P_IP))
	return XDP_DROP;

return XDP_PASS;
```

`eth + 1` is "one past the end of an ethhdr starting at eth", so the check reads as "if the header would run past the end of the packet, bail." doesnt matter that a real ethernet frame always has a full header, the verifier doesnt know whats "real", it only knows what this specific check proves.

> i was told the fallback here can drop or pass the packet, and that confused me, why does it matter to the verifier which one i pick. teacher explained it: we have to handle the packet right then, we cant come back to that data later.

> the takeaway: bounds-checking against `data_end` isnt a style preference or defensive-programming habit, its the literal price of admission. skip it and the program doesnt compile-and-maybe-crash later, it just never loads at all.

put together, thats a complete, loadable xdp program: drops ipv4 traffic, passes everything else through untouched.

## the verdict

`XDP_PASS`/`XDP_DROP` are the only two verdicts this program returns. the full four-verdict table (`XDP_TX`, `XDP_REDIRECT` included) is already in [[what-is-xdp|what-is-xdp]], not repeating it here, but worth knowing those two exist too, `XDP_TX` bounces the packet back out the same interface, `XDP_REDIRECT` sends it somewhere else entirely, neither one shows up until a program actually needs to.

## compiling it

```
clang -target bpf -c xdp_prog.c -o xdp_prog.o
```

`-target bpf` is what makes clang emit bpf bytecode instead of native code for whatever machine youre compiling on, `-c` stops before linking since theres nothing to link against, the `.o` is the whole output.

## loading it and checking its actually running

`xdp-loader load <iface> xdp_prog.o` attaches it, `xdp-loader status <iface>` shows whats currently attached:

```
beyza@debian:~$ sudo xdp-loader status ens3
CURRENT XDP PROGRAM STATUS:

Interface        Prio  Program name      Mode     ID   Tag               Chain actions
--------------------------------------------------------------------------------------
ens3                   xdp_dispatcher    skb      90   ba2e1a15f08cd656
 =>              50     drop_icmp                 99   0fb981d4a8cf833e  XDP_PASS
```

`xdp_dispatcher` on top is the dispatcher covered in [[what-is-xdp|what-is-xdp]], your own program (`drop_icmp` here, would be whatever you named yours) is the one listed underneath it. `bpftool prog show` and `bpftool map show` work too if you want to check the same thing without going through `xdp-loader`.

## looking further into the packet

matching on the ethertype alone only gets you so far. say you actually want to look inside the ip header, add the same idiom one layer deeper:

```c
struct iphdr *iph = (void *)(eth + 1);

if ((void *)(iph + 1) > data_end)
	return XDP_PASS;
```

`eth + 1` was "right after the ethernet header", so thats exactly where the ip header starts, and it needs its own bounds check before anything reads from it, the eth check from before only proved `eth` was safe, not `iph`.

want to go one header further and only act on icmp echo replies instead of every ipv4 packet? same pattern again, on top of the ip check:

```c
if (iph->protocol != IPPROTO_ICMP)
	return XDP_PASS;

struct icmphdr *icmph = (void *)(iph + 1);

if ((void *)(icmph + 1) > data_end)
	return XDP_PASS;

if (icmph->type == ICMP_ECHOREPLY)
	return XDP_DROP;
```

three headers deep now, eth, ip, icmp, and each one got its own check before getting touched. thats the whole idiom, it doesnt get more complicated than "check, then read", it just repeats once per header.

## counting packets with a map

dropping isnt the only thing to do with a match, counting works just as well, and it only needs a single-entry array map to hold the number:

```c
struct {
	__uint(type, BPF_MAP_TYPE_ARRAY);
	__uint(max_entries, 1);
	__type(key, __u32);
	__type(value, __u64);
} pkt_count SEC(".maps");
```

`BPF_MAP_TYPE_ARRAY` is about as simple as a map gets, fixed size, integer-indexed. with `max_entries` of 1 theres only one slot, so the key is always just `0`. declaring/creating/loading a map generically is already covered in [[../fundamentals|fundamentals]], this is just the specific type used here.

so the `return XDP_DROP;` from earlier becomes a lookup and an increment instead:

```c
if (eth->h_proto == bpf_htons(ETH_P_IP)) {
	__u32 key = 0;
	__u64 *cnt = bpf_map_lookup_elem(&pkt_count, &key);

	if (cnt)
		(*cnt)++;
}

return XDP_PASS;
```

`bpf_map_lookup_elem` can come back `NULL`, so the increment sits behind an `if (cnt)`, cant deref straight away. userspace reads the same map back out with `bpftool map dump`, also covered in fundamentals.

## race conditions on the counter

that increment isnt safe under concurrent packets. xdp programs run per-cpu, so two packets landing on different cores at the same moment can both run this at once.

say the counter is at 50 and two packets land on cpu 0 and cpu 1 at the same instant: both read 50, both compute 51, both write 51 back. two packets went through, the counter only moved by one.

it gets worse once theres a threshold check involved, like `if (*cnt >= 100) return XDP_DROP;` added before the increment. if the count is at 99 and 50 packets land across 50 cores at once, all 50 read 99, all 50 see `99 >= 100` as false, all 50 pass through and then increment. meant to cap it at exactly 100, ended up letting 149 through.

> turns out the bug isnt in the increment or the check individually, both are correct on their own. its that reading, deciding, and writing arent one atomic step, another core can slip in between any of them.

### fixing it: atomics vs per-cpu arrays

two ways to actually fix it.

`__sync_fetch_and_add` is one of them, it turns the whole read-modify-write into a single instruction nothing else can slip into the middle of:

```c
__sync_fetch_and_add(cnt, 1);
```

every core writes the same address, so any core sees the true total immediately, which matters if you need to drop on exactly the 100th packet. 

cost is cache contention, going atomic means a core has to grab exclusive ownership of that cache line and invalidate every other core's copy of it, so 16 cores hammering the same counter means 15 stall waiting on 1. at xdp packet rates that alone kills throughput. and it only makes the increment atomic, the check is still a separate step, so the race from the threshold example above is still there unless you go all the way to a compare-and-swap loop.

`BPF_MAP_TYPE_PERCPU_ARRAY` goes the other way:

```c
struct {
	__uint(type, BPF_MAP_TYPE_PERCPU_ARRAY);
	__uint(max_entries, 1);
	__type(key, __u32);
	__type(value, __u64);
} pkt_count SEC(".maps");
```

same declaration as the plain array, just the type changed. the kernel gives every core its own private copy of the counter, so theres no contention at all, no locking, caches stay hot. tradeoff is no single core knows the real total anymore, cant check `total >= 100` from inside the kernel program since that means reading every other core's slice first, userspace has to sum it up itself after reading every per-cpu value back out. also multiplies memory use by core count, fine for one counter, less fine for a hash map tracking millions of ips.

> atomic vs per-cpu isnt "correct vs incorrect", both give you a genuinely accurate count. the tradeoff is just where the cost lands, contention on every packet vs a summation step in userspace afterward.

## counting per source instead of globally

one counter for the whole interface doesnt say much though, so give every source ip its own slot instead, and cut a source off once it crosses a limit:

```c
struct stat {
	__u64 packets;
	__u64 bytes;
};

struct {
	__uint(type, BPF_MAP_TYPE_HASH);
	__uint(max_entries, 1024);
	__type(key, __u32);
	__type(value, struct stat);
} pkt_count SEC(".maps");
```

`BPF_MAP_TYPE_HASH` takes arbitrary keys instead of a fixed index range, `max_entries` here caps how many distinct keys can exist at once, not a slot count like the array's. the value grew into a small struct too, tracking bytes alongside packets.

the lookup itself barely changes, just keyed by the packet's real source ip now, with a limit check added and a fresh entry made the first time an ip shows up:

```c
__u32 key = ip->saddr;
struct stat *st = bpf_map_lookup_elem(&pkt_count, &key);

if (st) {
	if (st->packets >= 10)
		return XDP_DROP;

	__sync_fetch_and_add(&st->packets, 1);
	__sync_fetch_and_add(&st->bytes, data_end - data);
} else {
	struct stat initial = {.packets = 1, .bytes = data_end - data};
	bpf_map_update_elem(&pkt_count, &key, &initial, BPF_ANY);
}

return XDP_PASS;
```

`ip->saddr` needs the ip header pulled out first, same bounds-checked cast as the ip check earlier in this note. the `else` branch is new too, `bpf_map_update_elem` inserts an entry for a key that doesnt have one yet, `BPF_ANY` just means "write it whether or not something is already there", the other flags (`BPF_NOEXIST`, `BPF_EXIST`) let you require one or the other, not needed here since this branch only runs after a lookup already came back empty.

the increment itself is atomic now, but the check-then-act gap from the race-condition section is still technically there, reading `st->packets` and comparing it to 10 isnt combined with the increment into one op. its just spread across up to 1024 separate per-source counters instead of one shared global one now, so triggering it takes enough concurrent packets from the *same* source ip specifically, a much narrower window in practice than the global version.

## making the limit dynamically configurable

the 10 above was hardcoded, recompiling just to change a number gets old fast. put the limit inside the struct instead, so it can be set from userspace:

```c
struct stat {
	__u64 packets;
	__u64 bytes;
	__u64 limit;
};
```

```c
if (st) {
	if (st->limit > 0 && st->packets >= st->limit)
		return XDP_DROP;

	__sync_fetch_and_add(&st->packets, 1);
	__sync_fetch_and_add(&st->bytes, data_end - data);
}

return XDP_PASS;
```

no `else` branch anymore either. an ip with no entry just passes through uncounted now, someone has to seed it first with `bpftool map update` before it starts getting tracked at all.

## replying instead of dropping

different direction now: instead of dropping an icmp echo request, turn it into a reply and send it straight back with `XDP_TX`. same bounds-checked chain as before gets `icmph` this time, just checking for `ICMP_ECHO` (a request) instead of `ICMP_ECHOREPLY`.

`XDP_TX` retransmits whatever's sitting in the buffer, it doesnt rewrite anything on its own, so the addresses need swapping and the type needs flipping first:

```c
__u32 tmp_ip = ip->saddr;
ip->saddr = ip->daddr;
ip->daddr = tmp_ip;

unsigned char tmp_mac[ETH_ALEN];
__builtin_memcpy(tmp_mac, eth->h_source, ETH_ALEN);
__builtin_memcpy(eth->h_source, eth->h_dest, ETH_ALEN);
__builtin_memcpy(eth->h_dest, tmp_mac, ETH_ALEN);

__u16 old_type_code = *(__u16 *)icmph;
icmph->type = ICMP_ECHOREPLY;
__u16 new_type_code = *(__u16 *)icmph;
```

`ETH_ALEN` is just `6`, a mac address is always 6 bytes. `__builtin_memcpy` is the compiler's built-in version of `memcpy`, theres no libc here to call the real one from.

before touching the checksum, worth saying why it exists at all: its there so the receiving side can tell if a packet got corrupted somewhere along the way, nothing more than that, its not authenticating anything. whoever sends a packet computes a checksum over some range of bytes, whoever receives it recomputes the same sum and compares, a mismatch means a byte changed in transit. so change any byte inside the range a checksum covers and the old checksum stops matching, unless you go fix it up to match again.

swapping `saddr`/`daddr` doesnt touch the ip checksum at all though, a checksum is just a sum, swapping two values that are both already inside it doesnt change the total. the icmp checksum isnt so lucky, `type` genuinely changed value, so the old checksum really is wrong now:

```c
__u32 csum = (~icmph->checksum) & 0xffff;
csum += (~old_type_code) & 0xffff;
csum += new_type_code;
csum = (csum & 0xffff) + (csum >> 16);
csum = (csum & 0xffff) + (csum >> 16);
icmph->checksum = ~csum;
```

reads type+code together as one 16-bit unit since the checksum works in 16-bit words, not single bytes. undoes the old checksum, swaps in the two changed bytes, folds the carry twice (one fold can itself overflow), inverts back. cheaper than recomputing over the whole packet since only those two bytes actually changed.

skipped this once just to see what happens:

```
64 bytes from 127.0.0.1: icmp_seq=0 ttl=64 time=0.036 ms
checksum mismatch from 127.0.0.1
```

replies still came back. `XDP_TX` doesnt care whether the checksum is right, it retransmits regardless, `ping` on the other end is the one flagging it, every single reply.

## a userspace loader, in code

`xdp-loader`/`bpftool` are convenient, but sometimes the load/attach needs to happen from your own code instead. same three steps from [[../fundamentals|fundamentals]]'s loading writeup, just as actual libbpf calls now:

```c
struct bpf_object *obj = bpf_object__open_file(file_path, NULL);
bpf_object__load(obj);

struct bpf_program *prog = bpf_object__next_program(obj, NULL);
int prog_fd = bpf_program__fd(prog);

bpf_xdp_attach(ifindex, prog_fd, 0, NULL);
```

`open_file` reads the elf, `load` is the step that actually calls `BPF_PROG_LOAD` and runs the verifier. `bpf_object__next_program` pulls the first program definition back out of the opened object (the only one here), and `bpf_program__fd` is what turns that loaded program into an actual usable file descriptor, thats the fd `bpf_xdp_attach` wants. `bpf_xdp_attach` itself is the netlink attach call from [[what-is-xdp|what-is-xdp]], just done from code now instead of `xdp-loader load`. `ifindex` is the interface's numeric index, `if_nametoindex("ens3")` turns a name like that into one.

detaching later is `bpf_xdp_detach(ifindex, 0, NULL)`.

## reading a map back from userspace

a map doesnt need a program attached to be read, `bpftool map dump` already proves that. same walk, from your own code:

```c
int fd = bpf_map_get_fd_by_id(id);

unsigned int *prev_key = NULL, next_key;
unsigned long long val;

while (bpf_map_get_next_key(fd, prev_key, &next_key) == 0) {
	bpf_map_lookup_elem(fd, &next_key, &val);
	prev_key = &next_key;
}
```

maps and programs each get a kernel-wide id the moment theyre created ([[../fundamentals|fundamentals]] covers this), completely separate from any file. `bpf_map_get_fd_by_id` takes that id and hands back a usable fd for the map, same idea as opening a file by path, just for something living in the kernel instead of on disk. `bpf_map_get_next_key` then walks it one key at a time, `NULL` as the starting key means "give me the first one", each key after that feeds the next call, a nonzero return means the map is exhausted.

## a blacklist written from userspace

every map so far got written by the kernel program itself. this one flips that around, userspace pushes entries in, the xdp side only ever reads:

```c
struct {
	__uint(type, BPF_MAP_TYPE_HASH);
	__uint(max_entries, 1024);
	__type(key, __u32);
	__type(value, __u8);
} blacklist SEC(".maps");
```

```c
__u32 key = ip->saddr;
__u8 *blocked = bpf_map_lookup_elem(&blacklist, &key);

if (blocked)
	return XDP_DROP;
```

the value is a single byte that never actually gets read, presence in the map is the signal, not whats stored there. `bpftool map update` (or the equivalent library call) adds entries from outside, before any matching traffic even shows up.

## tracking flows instead of single counters

one counter per ip is still pretty coarse. a flow is really the 5-tuple, source/dest ip, source/dest port, protocol, and getting the ports means going one header past ip, into tcp or udp:

```c
struct five_tuple {
	__u32 src_ip;
	__u32 dst_ip;
	__u16 src_port;
	__u16 dst_port;
	__u8 protocol;
};

struct session_stat {
	__u64 packets;
	__u64 bytes;
};

static __always_inline int parse_five_tuple(struct iphdr *ip, void *data_end,
                                             struct five_tuple *key) {
	key->src_ip = ip->saddr;
	key->dst_ip = ip->daddr;
	key->protocol = ip->protocol;

	if (ip->protocol == IPPROTO_TCP) {
		struct tcphdr *tcp = (struct tcphdr *)(ip + 1);
		if ((void *)(tcp + 1) > data_end)
			return -1;
		key->src_port = tcp->source;
		key->dst_port = tcp->dest;
	} else if (ip->protocol == IPPROTO_UDP) {
		struct udphdr *udp = (struct udphdr *)(ip + 1);
		if ((void *)(udp + 1) > data_end)
			return -1;
		key->src_port = udp->source;
		key->dst_port = udp->dest;
	} else {
		return -1;
	}

	return 0;
}
```

same bounds-check idiom as ever, just one header further depending on the protocol. `session_stat` is the same shape as `stat` from before, just renamed since its tracking a flow now, not a single ip. pulled into its own function (`__always_inline` so it actually gets inlined into the caller instead of becoming a separate bpf-to-bpf call) since the program needs it more than once.

the map itself is `BPF_MAP_TYPE_LRU_HASH` instead of a plain hash:

```c
struct {
	__uint(type, BPF_MAP_TYPE_LRU_HASH);
	__uint(max_entries, 65536);
	__type(key, struct five_tuple);
	__type(value, struct session_stat);
} sessions SEC(".maps");
```

a plain hash just fails once its full. flows come and go constantly though, so LRU evicts the least-recently-used entry automatically to make room instead, no manual cleanup needed for connections that already ended.

## wildcard rules on an exact-match map

`BPF_MAP_TYPE_HASH` only does exact-match lookups, key in, exact same key out, nothing fuzzy about it. that becomes a problem the moment you want a rule like "block anything to port 80", since a rule like that only cares about one field (`dst_port`) and wants everything else, whatever source ip, whatever protocol, to match. store that rule with the other fields zeroed out as a wildcard, and a real packet's 5-tuple, which always has every field filled in, just never matches it, the hash of an all-fields-set key isnt the hash of a partially-zeroed one, exact match means exact.

the rule map itself needs the same rule-stat-per-key shape used everywhere else so far:

```c
struct rule_stat {
	__u64 packets;
	__u64 bytes;
};

struct {
	__uint(type, BPF_MAP_TYPE_HASH);
	__uint(max_entries, 1024);
	__type(key, struct five_tuple);
	__type(value, struct rule_stat);
} acl_rules SEC(".maps");
```

an admin seeds this from userspace with whatever fields a given rule actually cares about set, and the rest left at `0`, same presence-as-signal idea as the blacklist earlier.

one way around the exact-match problem: since the map cant be told to ignore fields, mask the packet's own tuple instead and try the map several times, once per possible combination of fields zeroed out:

```c
for (int mask = 0; mask < 32; mask++) {
	struct five_tuple probe = key;

	if (mask & 1)  probe.src_ip = 0;
	if (mask & 2)  probe.dst_ip = 0;
	if (mask & 4)  probe.src_port = 0;
	if (mask & 8)  probe.dst_port = 0;
	if (mask & 16) probe.protocol = 0;

	if (bpf_map_lookup_elem(&acl_rules, &probe))
		return XDP_DROP;
}
```

a `five_tuple` has 5 fields, so `2^5 = 32` possible subsets of them to zero out, `mask` just counts through all of them in binary, bit 0 controls `src_ip`, bit 1 `dst_ip`, and so on. `mask == 0` tries the packet's real, fully-specific tuple (an exact-match rule), `mask == 31` zeros every field (a rule matching literally everything). somewhere in between covers "block this source ip regardless of port" (`src_ip` set, rest zeroed) or "block this port regardless of source" (`dst_port` set, rest zeroed), and every other combination in between. whichever probe happens to land on a rule thats actually stored wins.

`#pragma unroll` has to sit right above that `for` loop for it to actually compile (left out above, but needed). the verifier wont accept a loop unless it can prove up front exactly how many times it runs, `#pragma unroll` tells the compiler to unroll it into 32 straight-line copies of the loop body at compile time instead of a real jump-back loop, so by the time the verifier sees it theres nothing left to prove about termination, its just 32 lookups in a row.

a second map tracks which real source ip actually triggered a drop:

```c
struct {
	__uint(type, BPF_MAP_TYPE_HASH);
	__uint(max_entries, 1024);
	__type(key, __u32);
	__type(value, struct rule_stat);
} drop_stats SEC(".maps");
```

needed because the rule that matched might itself have had `src_ip` wildcarded to `0`, `acl_rules` alone cant tell you who actually got dropped, only which rule fired. `drop_stats` is keyed by the packet's real, unmasked `saddr` instead, so stats stay attributable to an actual ip even when the rule that blocked it didnt care which one it was.

> `acl_rules` stays a plain hash on purpose, not LRU like the session map. these are admin-configured rules, not flow state, a rule silently getting evicted to make room would be a bug. an insert failing loudly once the table is full is the correct behavior instead.

worth being upfront about this though: its one way to solve the wildcard problem, not the only one, and its worth knowing where it actually falls short.

the big one is **priority**. the loop returns on the very first mask that finds a hit, and mask order is just counting in binary, it has nothing to do with how specific a rule actually is. say both of these are sitting in `acl_rules` at once:

| rule | fields kept | fields zeroed | mask |
|---|---|---|---|
| block protocol icmp, anything else | `protocol` | `src_ip`, `dst_ip`, `src_port`, `dst_port` | `1+2+4+8 = 15` |
| block `1.2.3.4` to port `80` specifically | `src_ip`, `dst_ip`, `dst_port` | `src_port`, `protocol` | `4+16 = 20` |

a packet from `1.2.3.4` to port `80` matches both. the loop reaches mask `15` before mask `20`, so the broad "any icmp" rule wins, purely because `15 < 20`, not because it was ever meant to outrank the much more specific rule sitting right next to it in the table. swap which fields two rules happen to wildcard and the outcome flips, theres no actual concept of "more specific wins" here, just whichever mask number the scan trips over first.

a real acl or firewall almost always needs that kind of precedence, specific rules overriding general ones, or an admin explicitly ordering rules. getting that here would mean not returning on the first hit at all, scanning all 32 regardless, keeping track of whichever match has the best priority so far (which means `rule_stat` would need a priority field to compare against in the first place), and only deciding after the full scan finishes. thats a real change to how this works, not a small tweak.

the other real cost is that its **exponential in the number of fields**:

| fields in the key | lookups per packet |
|---|---|
| 5 (`five_tuple`, this note) | 32 |
| 6 | 64 |
| 7 | 128 |
| 8 | 256 |

every field you add to what a rule is allowed to wildcard on doubles the lookups every single packet has to do, whether or not anything ever matches. real acl/routing code that needs this kind of matching usually reaches for something else instead: `BPF_MAP_TYPE_LPM_TRIE` is a map type built specifically for longest-prefix-match lookups, the kind of thing an ip routing table or a cidr-based firewall rule actually needs, and it finds the most specific applicable entry natively, without an application enumerating combinations by hand. it doesnt generalize to wildcarding arbitrary fields the way this code does though, its specifically about matching variable-length ip prefixes, so it solves a narrower problem than this one, it just solves its problem properly instead of working around a limitation.

