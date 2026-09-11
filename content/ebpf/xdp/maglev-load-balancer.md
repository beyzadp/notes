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

$$\text{client} \xrightarrow{\text{req}} \text{lb} \xrightarrow{\text{req}} \text{backend}$$
$$\text{client} \xleftarrow{\text{reply}} \text{lb} \xleftarrow{\text{reply}} \text{backend}$$

**direct server return**, the reply chain just has one less hop in it, the backend's arrow points straight back at the client instead of through the lb:

$$\text{client} \xrightarrow{\text{req}} \text{lb} \xrightarrow{\text{req}} \text{backend}$$
$$\text{client} \xleftarrow{\text{reply}} \text{backend}$$

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

getting all three of those (minimal disruption, fair load, O(1) lookup) out of one data structure comes down to two pieces: how each backend gets its own permutation over the table's slots, and how those permutations get merged into one shared lookup table.

imagine every backend gets handed its own private, shuffled wish-list ranking all `M` slots in the table from most to least wanted, a different shuffle per backend, no two backends sharing a list. filling the table is just going around the backends one at a time, each one claiming its next favorite still-empty slot off its own list, round after round, until every slot belongs to someone. thats the whole mechanism, before any math shows up at all.

say the table has `M = 7` slots (a real one runs to thousands, 7 is just enough to trace by hand) and there are 3 backends, a, b, c, and each one already showed up with its own ranked wish-list over the 7 slots. tracing the fill round by round:

| round | server | action | table state |
|---|---|---|---|
| start | - | initial empty state | [ -, -, -, -, -, -, - ] |
| 1 | a | claims 1st choice: **slot 3** | [ -, -, -, a, -, -, - ] |
| 1 | b | wants 1st choice **slot 3** (already claimed by a) -> claims 2nd choice: **slot 0** | [ b, -, -, a, -, -, - ] |
| 1 | c | claims 1st choice: **slot 1** | [ b, c, -, a, -, -, - ] |
| 2 | a | claims 2nd choice: **slot 5** | [ b, c, -, a, -, a, - ] |
| 2 | b | claims 3rd choice: **slot 4** | [ b, c, -, a, b, a, - ] |
| 2 | c | wants 2nd choice **slot 4** (already claimed by b), 3rd choice **slot 0** (already claimed by b), 4th choice **slot 3** (already claimed by a) -> claims 5th choice: **slot 6** | [ b, c, -, a, b, a, c ] |
| 3 | a | wants 3rd choice **slot 0** (already claimed by b) -> claims 4th choice: **slot 2**, table full | [ b, c, a, a, b, a, c ] |

that one conflict for b in round 1 and the run of three for c in round 2 are exactly what "already claimed" looks like in practice: a backend just keeps walking down its own list until it finds an empty slot, nobody asks permission, nobody coordinates. a ends up with 3 slots, b and c with 2 each, out of a possible even split of `7/3 ≈ 2.33`. off by one, not off by half the table.

> the round-robin fill is what makes it fair: every backend gets exactly one shot at a slot per round, so no backend can pull ahead of another by more than one slot, no matter how many of its early choices are already taken.

**now remove one backend.** drop c, keep a and b, their lists dont change at all, they never depended on c to begin with. only the round-robin fill changes shape now that there's one fewer backend competing for slots:

| round | server | action | table state |
|---|---|---|---|
| start | - | initial empty state | [ -, -, -, -, -, -, - ] |
| 1 | a | claims 1st choice: **slot 3** | [ -, -, -, a, -, -, - ] |
| 1 | b | wants 1st choice **slot 3** (already claimed by a) -> claims 2nd choice: **slot 0** | [ b, -, -, a, -, -, - ] |
| 2 | a | claims 2nd choice: **slot 5** | [ b, -, -, a, -, a, - ] |
| 2 | b | claims 3rd choice: **slot 4** | [ b, -, -, a, b, a, - ] |
| 3 | a | wants 3rd choice **slot 0** (already claimed by b) -> claims 4th choice: **slot 2** | [ b, -, a, a, b, a, - ] |
| 3 | b | claims 4th choice: **slot 1** | [ b, b, a, a, b, a, - ] |
| 4 | a | wants 5th choice **slot 4** (already claimed by b) -> claims 6th choice: **slot 6**, table full | [ b, b, a, a, b, a, a ] |

lined up against the original 3-backend table:

| slot | 0 | 1 | 2 | 3 | 4 | 5 | 6 |
|---|---|---|---|---|---|---|---|
| before (a, b, c) | b | c | a | a | b | a | c |
| after (a, b) | b | b | a | a | b | a | a |
| changed? | no | yes | no | no | no | no | yes |

only 2 of the 7 slots changed, and both of them, slot 1 and slot 6, were c's own slots to begin with. a and b keep every single slot they already held, not one extra slot moved on their side. this particular run happens to land on zero collateral ripple, thats not a guarantee every single time, but even with a little ripple the disruption stays sized to the departing backend's own slice of the table, nowhere near what naive mod-n does. 

compare it to the mod-n example from the section before this one: going from 4 backends to 3 moved 3 of 4 keys, basically the whole table, over one backend leaving. here, one of three backends left and only 2 of 7 slots moved, both of them slots that had to move. thats the "roughly `1/n`" claim from a few sections back showing up as an actual count.

all of that assumed each backend already walked in with its list. getting that list in the first place isnt magic, its just arithmetic:

| backend | h1 | offset = h1 mod 7 | h2 | skip = h2 mod 6 + 1 |
|---|---|---|---|---|
| a | 87 | 3 | 25 | 2 |
| b | 17 | 3 | 45 | 4 |
| c | 15 | 1 | 20 | 3 |

`offset` and `skip` are just two numbers, per backend, that together describe its whole list without ever writing the list out. `offset` is where that backend's list starts, its first choice, `h1` and `h2` are two independent hashes of the backend's own identifier (its ip:port, its name, whatever gets picked as the key), reduced down to fit the table with a mod. `skip` is how far apart each next choice is from the last one, always the same fixed distance around the table. a backend with `offset=3, skip=2` on a 7-slot table starts at 3, then jumps 2 every time: 3, 5, 0 (7 wraps back to 0), 2, 4, 6, 1, until it eventually visits every slot once. two numbers standing in for a list of thousands.

worth noticing before the formula even shows up: a and b land on the same offset, 3, out of two completely unrelated numbers, `87 mod 7` and `17 mod 7` just happen to both land on 3. thats pure coincidence of the modulo, and its exactly what caused their round 1 collision above.

**the permutation.** the list itself is just `(offset + j*skip) mod M` worked out for `j = 0, 1, 2, ..., M-1` in order. because `M` is prime and `skip` is never 0 mod `M`, that sequence visits every one of the `M` slots exactly once before it repeats, a full cycle, not a handful of shorter ones. (why it actually needs to be prime for that to hold is in [[#the fine print|the fine print]].)

worked out for a and b, side by side, since those are the two that collided:

```
a: offset = 87 mod 7 = 3     (7 x 12 = 84, remainder 3)
   skip   = 25 mod 6 + 1 = 1 + 1 = 2     (6 x 4 = 24, remainder 1)
   list   = (3 + j*2) mod 7 for j=0..6  ->  3, 5, 0, 2, 4, 6, 1

b: offset = 17 mod 7 = 3     (7 x 2 = 14, remainder 3)
   skip   = 45 mod 6 + 1 = 3 + 1 = 4     (6 x 7 = 42, remainder 3)
   list   = (3 + j*4) mod 7 for j=0..6  ->  3, 0, 4, 1, 5, 2, 6
```

both start at slot 3, same offset, different skip, so they immediately diverge after their first entry, exactly matching the round 1 collision and the two different paths they took afterward in the trace above. c's list comes out of the same two lines of arithmetic on its own `h1`/`h2`, `(1 + j*3) mod 7` for `j=0..6`, giving `1, 4, 0, 3, 6, 2, 5`, the list behind its 3-deep collision chain in round 2.

that formula is the only thing missing from the traces above, the round-robin fill itself works exactly as already shown.

at runtime none of this recomputing happens per packet, the table above is already built and sitting in memory. a packet just gets hashed once and the table does the rest:

$$\huge \text{packet} \xrightarrow{h(\text{5-tuple}) \bmod M} \text{slot index} \xrightarrow{\text{table}[\text{index}]} \text{backend}$$

one hash, one array index, done, thats the O(1) lookup half of the promise, and the minimal-disruption half is the table comparison a few paragraphs up.

## the fine print

[[#how it works|how it works]] glossed over two things to keep the algorithm readable: why `M` specifically has to be prime, and where all this computation actually happens once xdp is in the picture. both matter more once youre actually building this than the algorithm itself does.

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

**where this actually runs.** none of the round-robin filling happens inside the xdp program. it cant.

the fill loop's keep-trying-until-empty step is exactly the kind of unbounded loop the verifier rejects outright ([[../fundamentals|fundamentals]] covers that "no unbounded loops" rule). so the table gets built in userspace instead, once, whenever the backend set changes, then pushed into a bpf map thats already fixed-size the moment its created (same `max_entries` is a hard limit point from [[../fundamentals|fundamentals]]).

the xdp program on the hot path never rebuilds anything. it does exactly one thing per packet: hash the 5-tuple, mod `M`, look the result up in that map. all the complexity from this entire note lives in a userspace control program that runs rarely, the data-path program stays exactly as dumb and fast as [[#why xdp|why xdp]] needed it to be in the first place.

> the whole reason this algorithm is usable at packet rate is that its expensive half never has to run at packet rate. building the table costs orders of magnitude more than looking one value up in it, and the design just makes sure that cost only shows up on backend churn, not on every packet.


