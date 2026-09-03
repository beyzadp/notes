---
title: "Addressing Authorities"
---

zoom out from any single header field, like the `saddr`/`daddr` in [[encapsulation|an ip header]], and theres a whole hierarchy behind where that address actually came from.

iana sits at the top of the chain. it hands out big address blocks to five regional registries (RIRs), each covering a chunk of the world: ripe (europe), arin (north america), apnic (asia pacific), lacnic (latin america), afrinic (africa). the RIRs then allocate smaller blocks down to isps and large orgs in their region, who allocate even smaller ones to end customers. same hierarchy as domain names basically, just for ip space instead of names.

```
                        IANA
              (global pool, hands out /8s)
                          |
      +---------+---------+---------+---------+
      |         |         |         |         |
    RIPE      ARIN     APNIC     LACNIC    AFRINIC
    (EU)      (NA)     (APAC)    (LATAM)   (AFRICA)
      |
      v
  ISPs / large orgs (regional allocation)
      |
      v
  end customers (you)
```

ietf isnt an addressing authority though, its the standards body, publishes rfcs that define the protocols themselves. the ipv4/ipv6 header formats, cidr, all of it traces back to an rfc somewhere.

> iana and ietf split the job cleanly, iana runs the actual registries (who owns which address block, which protocol number means what), ietf writes the specs that reference those registries. iana isnt just handing out ip blocks either, its also the registry for a bunch of other protocol parameters rfcs point to, port numbers, and the ip protocol numbers behind the [[encapsulation|protocol field]] covered earlier.

dn42 is a community run network that mimics real internet routing/addressing (bgp, its own rir-style registry) but sits entirely outside iana, worth a look if you want to actually poke at this stuff hands on: [en.wikipedia.org/wiki/Dn42](https://en.wikipedia.org/wiki/Dn42).
