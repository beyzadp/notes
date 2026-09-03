---
title: "Encapsulation"
---

an ipv4 address is just one field, and that field sits inside a header, which itself sits inside more headers. before any of that addressing actually means anything on the wire, a packet has to get built in the first place: layers, wrapped in layers.

when data goes out it has to pass through several of them (app -> transport -> network -> link), and each layer needs to add its own addressing/control info without caring whats inside. thats what encapsulation and decapsulation are, the mechanism for wrapping and unwrapping that at each layer. zoomed all the way out, one computer sending to another looks like this, headers piling on going down, peeling back off going up:

![[encapsulation-tcpip-diagram.png]]

now the same idea, step by step:

```
 app data
    |
    v  wrap with tcp/udp header
+-------------------+------------------+
| L4 header (tcp/udp)|     data        |   <- segment
+-------------------+------------------+
    |
    v  wrap with ip header
+-----------+-------------------+------------------+
| L3 header |   L4 header       |     data          |   <- packet
+-----------+-------------------+------------------+
    |
    v  wrap with eth header (+ trailer)
+-----------+-----------+-------------------+------------------+------+
| L2 header | L3 header |   L4 header       |     data         | trlr |   <- frame
+-----------+-----------+-------------------+------------------+------+
```

same idea with the actual protocol names filled in instead of L2/L3/L4:

![[encapsulation-boxes.png]]

> that `trlr` at the end of the frame row is the FCS, a crc checksum appended after the payload so the receiving nic can tell if the frame got corrupted on the wire.

each arrow going down is an encapsulation step (add a header), going back up on the receiving side is decapsulation (strip a header, one per layer, in reverse order).

**encapsulation**: each layer wraps whatever it got from the layer above with its own header (sometimes a trailer too) on the way down the stack. app data becomes a tcp/udp segment once a transport header goes on, that becomes an ip packet once a network header goes on, that becomes an ethernet frame once a link header/trailer goes on, then it hits the wire. each layer only knows about its own header, everything above it is just opaque payload to it.

**decapsulation**: same thing in reverse on the receiving end. as the frame comes up the stack, each layer strips off the header meant for it, reads a field to figure out what protocol is next (eth header's ethertype says "ip", ip header's protocol field says "icmp/tcp/udp"), and hands the rest of the payload up to that protocol.

> those aren't arbitrary strings under the hood, ethertype `0x0800` means ipv4 and `0x0806` means arp, ip's protocol field is `6` for tcp, `17` for udp, `1` for icmp.

with the actual field names inside each header instead of just "header":

![[frame-packet-segment.png]]

> the real lesson: no layer ever needs to understand the layer above it, it just needs to know its own header format and one field telling it what comes next. thats the whole trick that lets ethernet, ip, and tcp get designed, implemented, and updated independently of each other.

the ip header itself follows the same layered idea, fixed fields up front, options after if it needs them. its 20 bytes minimum (up to 60 with options), and the fields that matter most day to day:

| field | what it does |
|---|---|
| ttl | hop limit, decremented per router |
| protocol | says what's next, icmp/tcp/udp |
| header checksum | covers just the ip header, recalculated at every hop since ttl changes each time |
| saddr / daddr | source / destination address, 32 bits each |
| ihl | header length in 32 bit words, needed because options make the header variable length |

the full 20-byte layout, with the fields that dont come up as often (version, type of service, identification, flags, fragment offset) alongside the ones above:

![[ipv4-header-fields.png]]

> `identification`, `flags`, and `fragment offset` up there exist for one reason, fragmentation. if a packet is bigger than the outgoing link's mtu it gets split into fragments that all share the same identification value, so the receiving end knows which pieces belong together and where each one goes using the offset. its also where the ~1460 byte number tcp connections often settle on comes from: a 1500 byte ethernet mtu minus a 20 byte ip header minus a 20 byte tcp header.
