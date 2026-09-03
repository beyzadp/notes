---
title: "NAT"
---

[[ipv4-basics|ipv4 exhaustion]] mentioned nat as one of the things that kept the internet running past 4.3 billion addresses. this is the actual mechanism: what nat does to a packet, the four variants and why they matter for peer-to-peer, and hole punching as the trick for getting two nat'd hosts talking directly without a relay in the middle.

## the parts

1. **[[01-introduction|How It Works]]**: what nat actually does to a packet, the translation table, and the four nat variants (full cone, restricted cone, port restricted cone, symmetric) and what each means for how hard p2p hole punching is.
2. **[[02-traversal|Traversal]]**: hole punching, the general technique for getting two nat'd hosts talking directly, how stun tells you your own public mapping, why symmetric nat breaks that (and when port prediction can still save it), turn as the reliable fallback, and ice as the real protocol name for tying it all together.
