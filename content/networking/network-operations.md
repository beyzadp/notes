---
title: "Network Operations"
---

a few things that dont fit neatly under one heading: the hex-conversion trick behind a weird ping trick, the actual tools for looking at real traffic instead of just reading header field tables, how much you can actually trust a dns answer, and a byte-order gotcha thats caused more than one confusing debugging session.

## hex conversion

computers store everything in binary, but binary is miserable to read or write by hand, a single byte is 8 digits of 0s and 1s. hex exists as a shorthand for that: base 16, digits 0-9 then a-f for 10-15, where each hex digit lines up with exactly 4 bits, so a full byte is always exactly 2 hex digits. decimal is just the base we count in day to day, and converting between the three is just changing which base youre reading a value in, not changing the value itself.

decimal to hex works by repeatedly dividing by 16 and reading the remainders back bottom to top, and once youre in hex, going to binary is direct, digit by digit, no math needed (`0xF` = `1111`, `0xA` = `1010`, etc), which is the whole reason hex is worth going through as a middle step instead of converting decimal straight to binary (same idea, dividing by 2 each time, just more error prone by hand).

that machinery is exactly what makes `ping 2131000000` actually reply. handing ping a random 10-digit decimal number and getting a real response back only makes sense once you run it through that conversion: decimal to hex, hex split into bytes, each byte read back as decimal, and what you land on is just an ipv4 address in disguise.

`2131000000` in hex is `0x7F047AC0`. split into bytes, 2 hex digits each, since an ipv4 address is 4 bytes, then read each byte back as decimal:

| hex byte | decimal |
|----------|---------|
| `7F`     | 127     |
| `04`     | 4       |
| `7A`     | 122     |
| `C0`     | 192     |

left to right thats `127.4.122.192`, exactly what ping printed below:

```
beyza ❯ ping 2131000000
PING 2131000000 (127.4.122.192) 56(84) bytes of data.
64 bytes from 127.4.122.192: icmp_seq=1 ttl=64 time=0.044 ms
64 bytes from 127.4.122.192: icmp_seq=2 ttl=64 time=0.073 ms
^C
--- 2131000000 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1051ms
rtt min/avg/max/mdev = 0.044/0.058/0.073/0.014 ms
```

> turns out `ping` never validates that its argument "looks like" an address, it just runs whatever number you hand it through the same decimal -> hex -> byte -> dotted-quad path every address goes through. `127.4.122.192` starts with `127`, so its inside the `127.0.0.0/8` loopback block (see [[ipv4-basics]]), which resolves straight back to the local machine no matter what the rest of the number is. thats why a random 10-digit number handed to ping still gets a reply instead of erroring out, its not a special case, its just an ip address that happened to be spelled in decimal.

## tools

all of the above, headers, ttl, hops, is easiest to actually believe once youve looked at real traffic doing it, which is what these tools are for. theyre the actual software used for looking at traffic on the wire, not protocols themselves.

wireshark is a gui packet capture/analysis tool, captures traffic on an interface and lets you inspect every header field of every packet, good for actually seeing what these headers (ip, tcp, icmp, etc) look like on real traffic instead of just reading struct definitions.

tcpdump is the same idea from the command line, captures and prints packets matching a filter expression. `sudo tcpdump dst 1.1.1.1 -d` for example captures traffic destined for 1.1.1.1, the `-d` flag there dumps the compiled bpf filter as human readable instructions instead of actually capturing, useful for seeing what the filter expression compiles down to.

two more flags worth knowing, both about how much tcpdump translates for you before printing:

| flag | effect |
|---|---|
| `-n` (repeatable up to `-nnn`) | turns off name resolution a level at a time: hostnames stay numeric ips, port numbers stay numeric instead of getting mapped to service names (`80` instead of `http`), and at `-nnn` protocol/as numbers stay numeric too |
| `-X` | dumps each packet's contents as hex plus ascii side by side, skips the link layer/ethernet header |
| `-XX` | same as `-X`, but includes the link layer/ethernet header too |

`-n` mainly earns its keep since reverse dns lookups for every packet can be slow or just noise you dont want in the output. the hex dump flags give you the same kind of raw byte view as the header field tables in [[encapsulation]], just read straight off the wire instead of parsed field by field.

dig is for dns specifically, sends a query and prints the response, good for checking what a domain resolves to or which nameserver is authoritative for it, without going through a browser.

## dns trust

and since dig's whole job is asking a resolver a question, its worth knowing how much that answer can actually be trusted, dns is plaintext and trust based by default, which is weaker than it sounds.

dns poisoning is when an attacker feeds a resolver a forged response before the real one gets back, so it caches the wrong ip for a domain. everyone using that resolver gets sent to the attacker's server until the bad entry expires. the classic off path version is a race, the attacker cant see the real query going out, but if they can guess two numbers fast enough, they can still win:

```
1. you      -> resolver     "whats the ip for example.com?"
2. resolver -> real server  query sent, tagged with a transaction id and
                             a source port (say txid=4471, port=51820)
3. attacker fires a forged reply before the real server can answer:
   attacker -> resolver     "example.com = 6.6.6.6"
                             src address spoofed to look like the real server
                             txid and port guessed blind, has to land on 4471/51820
4. resolver checks: does this reply's txid/port match the query i sent?
   if the guess landed -> accepted, caches example.com -> 6.6.6.6
5. real server -> resolver  the actual answer shows up a moment later,
                             too late, the resolver already has a cached entry
6. resolver -> you          "example.com = 6.6.6.6"   (wrong, attacker's server)
```

the whole attack lives or dies on step 3: a resolver has no way to tell a forged reply from a real one except "does the txid and source port match what i just sent out." guess those two numbers before the real server replies and the forged answer wins the race. a vulnerable/misconfigured resolver can also just get told the wrong answer directly, no guessing needed, but the off-path race is the interesting case since the attacker never sees the traffic at all.

what an attacker actually does with a poisoned entry varies: point a bank or login domain at a lookalike page to harvest credentials, point a software update or package mirror domain at a server serving malware instead of the real update, or redirect ad/traffic to somewhere that pays them instead of the real site. the resolver doesnt have to be some random home router either, a handful of documented cases are isps or national-level resolvers redirecting specific domains for censorship or surveillance, same mechanism as above, just poisoning a resolver that millions of people happen to use instead of one.

dns over tls/https close part of that gap. normal dns is plaintext udp on port 53, so anyone on the path (isp, a mitm) can read or spoof it. dot wraps the query in tls (port 853), doh wraps it in an https request (port 443, blends in with regular web traffic).

> worth remembering: dot and doh protect the wire, not the cache. both stop on-path eavesdropping/tampering between the client and the resolver, but neither does anything about the resolver's own cache getting poisoned, thats a separate problem they dont solve.

internet exchange point (ixp) is physical infrastructure where multiple networks (isps, cdns, big platforms) plug into the same switch fabric to hand traffic to each other directly, instead of routing it out through a transit provider. keeps local traffic local, lower latency, cheaper for everyone connected. trnog (turkey network operators group) is the community/conference around this stuff in turkey, network engineers getting together to share what theyre running into, same idea as nanog/ripe meetings elsewhere just regional.

## endianness

one more place raw bytes get misread instead of misused: which end of a multi-byte number you start reading from.

endianness is just the order the bytes of a multi byte value get stored/read in. big endian stores the most significant byte first, little endian stores the least significant byte first, same number, different byte order depending on which side youre reading it from.

network protocols use big endian by convention (thats what "network byte order" means, the ip header fields covered in [[encapsulation]] are all in this order), x86/x86_64 cpus store things little endian internally. so anytime raw bytes cross from "how the wire/protocol writes them" to "how the cpu reads them" without an explicit conversion, this kind of thing happens.

this one came up while writing one of the xdp programs. i had a bpf map keyed by source ip, stored as a raw `__u32`, and i'd been pinging google to generate some traffic to test it with:

```
beyza@debian:~/24.08$ ping google.com
PING google.com (142.251.38.238): 56 data bytes
```

dumping the map back out from userspace, one of the keys was `3995532174`. expecting that to just be `142.251.38.238` spelled out as a plain decimal int, i pinged the number directly as a sanity check:

```
beyza@debian:~/24.08$ sudo bpftool map dump id 161
[{ "key": 3995532174, ... }]
beyza@debian:~/24.08$ ping 3995532174
PING 3995532174 (238.38.251.142): 56 data bytes
```

`238.38.251.142` came back, not `142.251.38.238`. not a broken map, not a typo, the exact byte-reversal of the real address.

> the real lesson here: thats endianness, not a bug. ipv4 dotted notation is big endian (most significant octet first, network byte order). `bpftool` read those 4 bytes as a native little-endian int instead, flipping which byte counts as most significant, and ping then made the opposite mistake on a bare numeric argument, unpacking it assuming big endian. two different endian assumptions stacked on the same bytes, so the octets come out reversed. to read a value like that back as an actual ip you have to byte swap it, e.g. `socket.inet_ntoa(struct.pack('<L', 3995532174))` in python.
