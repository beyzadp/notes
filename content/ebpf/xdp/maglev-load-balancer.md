---
title: "Maglev Load Balancer"
---

part of the [[index|xdp]] series. [[coding-xdp|coding-xdp]] built up an xdp program piece by piece for filtering/counting traffic, this is the same toolkit pointed at a different job: picking which backend a packet goes to. notes from actually implementing one.

## what is a load balancer

a load balancer sits in front of a pool of backend servers. for every incoming connection it decides which one of them actually handles it, instead of all the traffic landing on a single machine. the pool itself is nothing special, just a list of backend ip's the lb knows about. the interesting part is entirely in how it picks one and what it does with the packet once its picked.

"how it picks one" comes down to which layer the lb actually reads, same layers [[../../networking/encapsulation|encapsulation]] covers for building a packet in the first place, just read back off instead of stacked on. that "how" splits along two axes.

first, which layer the decision happens at. an l7 load balancer terminates the actual application protocol. it opens a real tcp connection with the client, reads the http request, and only then decides where to send it. that means it can route on things like a url path or a header. an l4 load balancer never looks that deep, it only ever sees ip's and ports, the tcp payload is opaque to it. that's a real tradeoff: l7 gets to make smarter decisions but pays for a full tcp/tls stack per connection. l4 is cheap precisely because it refuses to look.

| layer | protocol example | what a lb sees there | header field it would key on |
|---|---|---|---|
| l2 (link) | ethernet | just mac addresses | dst mac, rewritten by dsr to point at the chosen backend |
| l3 (network) | ip | src/dst ip | daddr for the client's target, saddr as part of the hash |
| l4 (transport) | tcp/udp | src/dst port | the other half of the hash, plus tcp flags for connection state |
| l7 (application) | http | url, headers, cookies, body | whatever the routing rule actually cares about |

only l4 and l7 turn into an actual lb category. l2 (mac) dies at the first router hop, nothing left there to route on, it only comes back for the dsr rewrite. l3 never splits off from l4 either, ports dont exist without an ip header underneath, so the two get bundled into one name. l5/l6 dont even show up, that's an osi-model split, [[../../networking/encapsulation|encapsulation]] skips them too since tcp/ip never gave them their own header.

| | l7 | l4 |
|---|---|---|
| sees | full http request: url, headers, body | just src/dst ip and port |
| does it terminate tcp | yes, a real connection with the client | no, payload stays opaque |
| can route on | url path, headers, cookies | nothing past ip/port |
| cost per connection | full tcp/tls stack | just a lookup |
| example | haproxy, nginx | ipvs, maglev |

second, what happens to the reply. an inline (nat mode) lb sits in the path of both directions. it rewrites the destination on the way in and the source on the way back out, so every single packet of every connection, both ways, passes through it. direct server return (dsr) only puts the lb in the inbound path: it rewrites the packet enough to get it to the right backend, and the backend replies straight to the client itself. the lb never sees the response at all. dsr is the only one of the two that scales past "the lb's own nic is the bottleneck", since return traffic (usually the bigger half, most protocols answer with more bytes than they receive) skips it entirely.

**inline / nat mode**, every hop both ways goes through the lb:

```
client --req--> lb --req--> backend
client <-reply- lb <-reply- backend
```

**direct server return**, the reply chain just has one less hop in it, the backend's arrow points straight back at the client instead of through the lb:

```
client --req--> lb --req--> backend
client <---------reply--------- backend
```

> an l4/dsr load balancer's job description is almost suspiciously small. look at a packet's ip/port, pick a backend, rewrite enough to get it there, and never touch that connection's reply traffic again. all the actual hard part is making that pick consistent, which is exactly the problem [[#introducing maglev|maglev]] exists to solve.

maglev, the thing this whole note is about, is one specific answer to that "which layer, which direction" question. its an l4, dsr load balancer. that's the shape everything below assumes.

## why xdp

an l4/dsr load balancer's entire job is a decision made once per packet, on the way in, at whatever the nic's actual line rate is. that's the same shape of problem [[what-is-xdp|what-is-xdp]] already covers: "xdp is the earliest hook in the rx path, it runs before the kernel even builds an skb for the packet... a drop costs basically nothing." a load balancer isnt dropping the packet, its redirecting it. but the economics are the same: the fewer things the kernel has done to a packet before your program gets to decide its fate, the cheaper that decision is. at lb scale (millions of packets/sec) a cost thats negligible per-packet stops being negligible in aggregate.

put next to the alternatives that argument gets more concrete. ipvs (the kernel's own l4 lb, linux virtual server) makes its decision from netfilter. by the time ipvs even sees a packet, an skb has already been allocated for it, and it has gone through conntrack, a hash table lookup, and a state entry per connection. haproxy is even further out, its a userspace process, so every packet crosses the kernel/userspace boundary through a socket before any decision gets made at all. xdp's whole pitch is skipping straight to the front of that queue: the program runs on the raw buffer the nic dma'd in, before any of that machinery has spent a single cycle on the packet.

> this isnt a hypothetical argument. its the exact reason facebook's katran (their production l4 lb) is an xdp/ebpf program instead of sitting on ipvs. its the same design google's original maglev paper describes running as software on commodity servers, just moved down to the earliest hook available once ebpf made that an option. the "make the decision before the kernel builds an skb" argument from [[what-is-xdp|what-is-xdp]] is precisely why.

## the problem with naive hashing

xdp makes the decision cheap. it doesnt make the decision good. the actual pick still has to happen, and the obvious way to write it is: hash the packet's 5-tuple, take that number mod the backend count, use the result as an index into the backend list.

that works fine as long as the backend count never changes. the moment it does, either a backend dies, or you scale the pool up, almost every key's mod result changes, not just the keys that were near the backend that got added or removed.

a small example, 4 keys against 4 backends vs the same 4 keys against 3 backends:

```
key   n=4: key mod 4   n=3: key mod 3
10    2                1
11    3                2
12    0                0
13    1                1
```

only key 12 lands on the same index both times. the other three all get reassigned, even though nothing about those three connections changed at all.

for a stateless request that reassignment costs nothing, the next backend just answers it fresh. for a live tcp connection its fatal: the new backend never saw the handshake, has no socket for it, and just resets or silently drops the packet. from the client's side the connection just dies, for no reason it can see, and this happens to a large fraction of every connection on the box at once; not only the ones that were actually using the backend that changed.

> in brief, modulo hashing ties every key's backend to the total backend count. touch that count by one, and the table reshuffles almost entirely. fatal for anything holding a connection open.

## introducing maglev

that reshuffling problem is exactly what maglev was built to remove. it comes out of a 2016 google paper ("maglev: a fast and reliable software network load balancer"), describing the l4/dsr software lb google runs in front of basically everything.

the specific piece of it this note cares about is the hashing scheme, a way of building the backend lookup table so that losing or adding one backend only remaps the connections that have to move, roughly `1/n` of them, instead of nearly all of them like the mod-n version above. its usually filed under "consistent hashing", though the actual construction (a permutation table per backend, covered in [[#how it works|how it works]] below) looks nothing like the ring-based consistent hashing used elsewhere, its googles own scheme built specifically for this lookup-table shape.

the paper actually names three design goals, not just the one above: minimal disruption on backend churn, roughly equal load across backends even when theyre weighted unevenly, and an O(1) lookup at packet time. only the first is a real fix over naive mod-n, mod-n already gives close to even distribution and O(1) lookup as long as the hash function itself is decent. the other two matter once maglev gets compared against the *other* usual answer to the disruption problem, ring-based consistent hashing, which fixes disruption too but costs an O(log n) lookup and skews load unless you add virtual nodes. maglev's whole pitch is getting the disruption fix without giving up either of those.

## how it works

two programs sharing one array. the data path reads it per packet, the control path builds it on backend churn. start with the reader, since its four lines.

### the data path

the array is `table[M]`, every slot holding a backend ip. `M` is fixed, never changes, has nothing to do with how many backends you have.

```
on packet:
    key  = (src_ip, src_port, dst_ip, dst_port, proto)
    i    = hash(key) % M
    dst  = table[i]
    rewrite packet to dst, send it
```

one hash, one array read. no loops, no per-connection state, nothing recomputed. same 5-tuple always lands on the same slot, so every packet of a connection goes to the same backend without anyone remembering that connection exists.

note whats *not* here: the backend count. the divisor is `M`. thats the entire difference from `hash % n` back in [[#the problem with naive hashing|the problem with naive hashing]], where the divisor is the backend count and so touching the count moves every key.

### prefilling the table

**what the table has to look like.** if the data path spreads packets evenly over `M` slots, load is even exactly when the *slots* are even. three backends and `M = 9` means the array needs three of each backend's ip, scattered anywhere, the hash doesnt care about order.

second requirement: when one backend dies and you rebuild, its slots get handed out to whoever's left, and everything else stays put. slots that werent the dead backend's shouldnt move.

**how to build it.** go around the backends in turn. each one hashes its own ip to a slot and claims it. if that slot's already taken, hash again for the next candidate, and again, until it finds a free one. one claim per turn, then the next backend goes. repeat until the array is full.


```
while table not full:
    for backend in backends:
        i = next_candidate(backend)
        while table[i] != empty:
            i = next_candidate(backend)
        table[i] = backend.ip
```

`next_candidate` is doing real work under the hood, some math that gives each backend its own fixed sequence of slots to try, covered further down in "generating the candidates". for now just treat it as a function that hands back a backend's next preferred slot.

traced out with real numbers, `M = 7`, three backends a, b, c, each already carrying its own candidate sequence:

| round | server | action                                                                                                                                                                          | table state             |
| ----- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------- |
| start | -      | initial empty state                                                                                                                                                             | [ -, -, -, -, -, -, - ] |
| 1     | a      | next_candidate(a) -> 3<br>claims 1st choice: **slot 3**                                                                                                                         | [ -, -, -, a, -, -, - ] |
| 1     | b      | next_candidate(b) -> 3<br>wants 1st choice **slot 3** (already claimed by a) <br>reruns next_candidate(b) -> 0<br>claims 2nd choice: **slot 0**                                 | [ b, -, -, a, -, -, - ] |
| 1     | c      | claims 1st choice: **slot 1**                                                                                                                                                   | [ b, c, -, a, -, -, - ] |
| 2     | a      | claims 2nd choice: **slot 5**                                                                                                                                                   | [ b, c, -, a, -, a, - ] |
| 2     | b      | claims 3rd choice: **slot 4**                                                                                                                                                   | [ b, c, -, a, b, a, - ] |
| 2     | c      | wants 2nd choice **slot 4** (already claimed by b), 3rd choice **slot 0** (already claimed by b), 4th choice **slot 3** (already claimed by a) -> claims 5th choice: **slot 6** | [ b, c, -, a, b, a, c ] |
| 3     | a      | wants 3rd choice **slot 0** (already claimed by b) -> claims 4th choice: **slot 2**, table full                                                                                 | [ b, c, a, a, b, a, c ] |

that one conflict for b in round 1 and the run of three for c in round 2 are exactly what the inner `while` loop above is doing: a backend just keeps calling `next_candidate` until it lands on something empty, nobody asks permission, nobody coordinates.

the construction gives three things for free.

**even.** everyone gets exactly one slot per pass through the outer loop, so no backend can pull ahead of another by more than one slot, no matter how many of its early choices are already taken. here a ends up with 3 slots, b and c with 2 each, out of a possible even split of `7/3 ≈ 2.33`, off by one, not off by half the table.

**stable.** each backend's candidate sequence depends only on its own ip, not on who else is in the pool. drop c, keep a and b, and only the round-robin shape changes:

| round | server | action                                                                                          | table state             |
| ----- | ------ | ----------------------------------------------------------------------------------------------- | ----------------------- |
| start | -      | initial empty state                                                                             | [ -, -, -, -, -, -, - ] |
| 1     | a      | claims 1st choice: **slot 3**                                                                   | [ -, -, -, a, -, -, - ] |
| 1     | b      | wants 1st choice **slot 3** (already claimed by a) -> claims 2nd choice: **slot 0**             | [ b, -, -, a, -, -, - ] |
| 2     | a      | claims 2nd choice: **slot 5**                                                                   | [ b, -, -, a, -, a, - ] |
| 2     | b      | claims 3rd choice: **slot 4**                                                                   | [ b, -, -, a, b, a, - ] |
| 3     | a      | wants 3rd choice **slot 0** (already claimed by b) -> claims 4th choice: **slot 2**             | [ b, -, a, a, b, a, - ] |
| 3     | b      | claims 4th choice: **slot 1**                                                                   | [ b, b, a, a, b, a, - ] |
| 4     | a      | wants 5th choice **slot 4** (already claimed by b) -> claims 6th choice: **slot 6**, table full | [ b, b, a, a, b, a, a ] |

a and b produce the exact same candidate sequence they did with c in the pool, they just stop losing races to it. lined up against the original:

| slot             | 0   | 1   | 2   | 3   | 4   | 5   | 6   |
| ---------------- | --- | --- | --- | --- | --- | --- | --- |
| before (a, b, c) | b   | c   | a   | a   | b   | a   | c   |
| after (a, b)     | b   | b   | a   | a   | b   | a   | a   |
| changed?         | no  | yes | no  | no  | no  | no  | yes |

only 2 of the 7 slots changed, and both of them, slot 1 and slot 6, were c's own slots to begin with. a and b keep every single slot they already held, not one extra slot moved on their side. this particular run happens to land on zero collateral ripple, thats not a guarantee every single time, but even with a little ripple the disruption stays sized to the departing backend's own slice of the table, nowhere near what naive mod-n does. compare it to the mod-n example from [[#the problem with naive hashing|the problem with naive hashing]]: going from 4 backends to 3 moved 3 of 4 keys, basically the whole table, over one backend leaving. here, one of three backends left and only 2 of 7 slots moved, both of them slots that had to move. thats the "roughly `1/n`" claim from a few sections back showing up as an actual count.

**deterministic.** nothing in the loop above reads a clock or a counter. every load balancer in the fleet runs this independently from the same backend list and gets a byte-identical table, no coordination, no gossip.

**generating the candidates.** `next_candidate` could literally be rehashing, but maglev uses something cheaper that gives the same guarantee: two hashes of the backend's ip, once, up front.

```
offset = h1(ip) % M            where this backend starts
skip   = h2(ip) % (M-1) + 1    how far it jumps each time
candidates: offset, offset+skip, offset+2*skip, ... all % M
```

`offset` is a backend's first choice, `skip` is the fixed distance to every choice after that, two numbers standing in for a list of thousands. worked out for the three backends above:

| backend | h1 | offset = h1 mod 7 | h2 | skip = h2 mod 6 + 1 |
|---|---|---|---|---|
| a | 87 | 3 | 25 | 2 |
| b | 17 | 3 | 45 | 4 |
| c | 15 | 1 | 20 | 3 |

a and b landing on the same offset, 3, out of two completely unrelated numbers, `87 mod 7` and `17 mod 7` just happen to both land on 3, is pure coincidence of the modulo. its exactly what caused their round 1 collision above:

```
a: offset = 87 mod 7 = 3     skip = 25 mod 6 + 1 = 2     candidates: 3, 5, 0, 2, 4, 6, 1
b: offset = 17 mod 7 = 3     skip = 45 mod 6 + 1 = 4     candidates: 3, 0, 4, 1, 5, 2, 6
```

both start at slot 3, then immediately diverge, matching the round 1 collision and the two different paths they took afterward in the trace above. c's candidates come out of the same two lines of arithmetic on its own `h1`/`h2`, `1, 4, 0, 3, 6, 2, 5`, the sequence behind its 3-deep collision chain in round 2.

the point of `skip` landing anywhere in `1..M-1` and `M` being prime is that this walk hits every one of the `M` slots exactly once before repeating, so a backend can never run out of candidates while empty slots remain. ([[#the fine print|the fine print]] on what breaks otherwise.)

### where each half runs

the fill loop a few paragraphs up is unbounded, so it cant be an xdp program, the verifier rejects that outright ([[../fundamentals|fundamentals]] covers the "no unbounded loops" rule). its a userspace control program instead: runs once whenever the backend set changes, writes the finished table into a bpf map thats already fixed-size the moment its created (same `max_entries` is a hard limit point from [[../fundamentals|fundamentals]]), then goes back to sleep. the xdp side, the data path from the top of this section, never builds anything, it just indexes, staying exactly as dumb and fast as [[#why xdp|why xdp]] needed it to be in the first place.

> thats the actual reason this is usable at line rate: the expensive half runs on backend churn, the cheap half runs per packet.

## the fine print

[[#how it works|how it works]] glossed over two things to keep the algorithm readable: why `M` specifically has to be prime, and how big it actually needs to be. both matter more once youre actually building this than the algorithm itself does.

**why prime.** the whole thing rests on `(offset + j*skip) mod M` visiting all `M` slots before it repeats. that only happens when `skip` and `M` share no common factor, `gcd(skip, M) = 1`. a prime `M` guarantees that automatically: its only factors are 1 and itself, and `skip` is always smaller than `M`, so the `gcd` can never come out to anything but 1. no exceptions, nothing to check.

pick a composite `M` instead and this breaks silently. say `M = 6` and a backend lands on `skip = 2`:

```
gcd(2, 6) = 2
(offset + j*2) mod 6, offset = 0:   0, 2, 4, 0, 2, 4, ...
```

only 3 of the 6 slots (0, 2, 4) ever get visited. the other 3 (1, 3, 5) are unreachable by that backend, forever. a naive implementation doesnt crash on this, it just spins looking for an empty slot that will never show up.

making `M` prime isnt an optimization, its what makes every possible `skip` safe to use without checking it first.

**how big.** `M` cant just be barely bigger than the backend count. the whole point of the round-robin fill was spreading slots thin enough that being off by one never matters, and that stops being true the moment "one slot" is a meaningful fraction of what a backend gets. too small an `M` quietly defeats the "roughly equal" guarantee from the section above.

the actual maglev paper settles on `M = 65537` (`2^16 + 1`, prime). the general guidance is the same idea scaled up: keep `M` many times larger than the backend count, not just barely past it.

