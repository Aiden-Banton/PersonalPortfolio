# CCNA-108 — IPv6 Addressing and Connectivity: Derive It, Configure It, Prove It

| | |
|---|---|
| **Covers** | 1.8 (IPv6 addressing and prefix), 1.9 (IPv6 address types), 1.6 (subnetting concepts, applied) |
| **Tier** | T2 Applied — **you fill in the addressing table yourself** |
| **Platform** | Packet Tracer 8.2+ |
| **Time** | 55–65 minutes |
| **Prerequisites** | None beyond IPv4 basics |
| **Status** | built |

---

## Why this lab exists

IPv6 is the topic candidates most often decide to "come back to," and then don't. It sits
in the highest-weight exam domain, and the questions are not conceptual — they ask you to
identify an address type on sight, compress a prefix correctly, and know which address a
host actually uses to talk to its gateway.

Unlike the other labs in this collection, **the addressing table here is blank on purpose.**
The learning objective *is* the addressing. If the table were filled in you would be
practising interface configuration, which you can already do.

## Scenario

Fenwick Marine is adding IPv6 alongside its existing IPv4 network — dual-stack, not a
migration. Head office has allocated the documentation prefix `2001:db8:acad::/48` for the
lab build, and wants each site LAN on its own /64.

Three routers, three site LANs, dual-stacked throughout.

## Topology

```
         [ R1 ]---- 2001:db8:acad:12::/64 ----[ R2 ]
            |  \                                 |
            |   \                                |
            |    2001:db8:acad:13::/64           |
            |          \                         |
   LAN A    |           \                        | LAN B
   PC1      |          [ R3 ]                   PC2
            |            |
            |          LAN C
            |          PC3

   Link networks:  R1-R2, R1-R3  (R2-R3 not connected in this lab)
   Site LANs:      LAN A off R1, LAN B off R2, LAN C off R3
```

Topology diagram: not yet drawn — the ASCII diagram above is canonical for now. Runnable
file: [`topology/CCNA-108.pkt`](./topology/) *(add when built)*

---

## Part 1 — Derive the plan

You have `2001:db8:acad::/48`. Subnet it into /64s and assign them as follows, using the
**lowest available subnet IDs in order**: LAN A, LAN B, LAN C, then the two link networks.

**Task 1.** Fill in this table before configuring anything. Write it out by hand first —
the point is the derivation, not the typing.

| Segment | Prefix (/64) | Router | Interface address | Notes |
|---|---|---|---|---|
| LAN A | `2001:db8:acad:____::/64` | R1 | `____::1/64` | |
| LAN B | `2001:db8:acad:____::/64` | R2 | `____::1/64` | |
| LAN C | `2001:db8:acad:____::/64` | R3 | `____::1/64` | |
| Link R1–R2 | `2001:db8:acad:____::/64` | R1 `____::1`, R2 `____::2` | | |
| Link R1–R3 | `2001:db8:acad:____::/64` | R1 `____::1`, R3 `____::2` | | |

**Task 2.** Answer these before you configure. Write the answers down — several are exam
questions almost verbatim:

1. How many /64 subnets does a /48 yield? Show the arithmetic, don't recall the number.
2. Write `2001:0db8:0acad:0001:0000:0000:0000:0001` in fully compressed form.
3. Is `2001:db8:acad:1::/64` a global unicast, unique local, or link-local prefix? How do
   you know from the address alone?
4. Which address will PC1 use as its default gateway — R1's global unicast address on that
   LAN, or R1's link-local address? Why?

> Question 4 is the one that catches people. Hosts learn their default gateway from Router
> Advertisements, and the source of an RA is the router's **link-local** address. So the
> gateway a host installs is a `fe80::` address even though every packet it forwards is
> addressed globally. This is a genuine structural difference from IPv4, not a trivia detail.

---

## Part 2 — Configure

**Task 3.** Enable IPv6 unicast routing on all three routers. Note what does *not* work
before you do this, and why the symptom looks like an addressing problem.

**Task 4.** Configure the global unicast addresses from your table on every router interface.

**Task 5.** Configure a **static, memorable link-local address** on each router interface
rather than accepting the EUI-64-derived one — use `fe80::1` on R1, `fe80::2` on R2,
`fe80::3` on R3.

> **Why override it?** The auto-generated link-local address is derived from the MAC and is
> unreadable in output — `fe80::2d0:97ff:fe5b:1a01` tells you nothing at a glance. Setting
> it makes every `show ipv6 route` and neighbour table instantly readable, and it is
> standard practice on production gear for exactly that reason. It also makes the next
> task's point visible.

**Task 6.** Configure the hosts. Use SLAAC on PC1 and PC3, and a static address on PC2.

**Checkpoint:**

| Check | Command | Expected | Save to |
|---|---|---|---|
| Addresses applied | `show ipv6 interface brief` on each router | GUA and `fe80::x` per interface | `output/01-ipv6-interface-brief.txt` |
| Routing enabled | `show running-config \| include ipv6 unicast` | Present on all three | `output/02-ipv6-unicast-routing.txt` |
| SLAAC worked | `ipconfig` on PC1 | GUA in LAN A prefix, gateway is `fe80::1` | `output/03-pc1-slaac.txt` |
| Neighbour discovery | `show ipv6 neighbors` on R1 | PC1 and R2/R3 present | `output/04-ipv6-neighbors.txt` |

---

## Part 3 — Make the LANs reach each other

**Task 7.** With addressing complete, PC1 still cannot reach PC2. Explain why before
fixing it — the answer is one sentence and it is the same reason it would be true in IPv4.

**Task 8.** Configure IPv6 static routes so all three LANs reach each other. Use the
**link-local next-hop with an exit interface** form, not the global address.

> This form (`ipv6 route <prefix> <interface> <link-local-next-hop>`) is the one you will
> see in production and in exam output, and it is why Task 5 mattered — a static route
> pointing at `fe80::2 GigabitEthernet0/0/0` is readable. Pointing at an EUI-64 link-local
> address is not, and mistyping one character in a 39-character address is a debugging
> session you don't need.

**Task 9.** Verify end-to-end connectivity between all three LANs.

| Check | Command | Expected | Save to |
|---|---|---|---|
| Routes installed | `show ipv6 route static` on all three | Remote LAN prefixes present | `output/05-ipv6-static-routes.txt` |
| PC1 → PC2 | `ping` | Success | `output/06-ping-lana-lanb.txt` |
| PC1 → PC3 | `ping` | Success | `output/07-ping-lana-lanc.txt` |
| Path is as expected | `tracert` from PC1 to PC2 | Via R1 → R2 | `output/08-traceroute-v6.txt` |

---

## Part 4 — Identify what you're looking at

**Task 10.** Capture the output of `show ipv6 interface G0/0/0` on R1 and, in
`output/notes.md`, label every address it lists by type: global unicast, link-local,
solicited-node multicast, all-nodes multicast, all-routers multicast.

Most candidates can define these in isolation and cannot spot them in real output. That
gap is exactly what the exam tests.

**Task 11.** From that output, answer:

1. Why does the interface have a solicited-node multicast address it was never configured with?
2. What is the IPv6 equivalent of IPv4 ARP, and which multicast address does it use?
3. If you shut down IPv6 routing on R1, which of these addresses disappears?

Save to `output/09-ipv6-interface-detail.txt` and `output/notes.md`.

---

## Final verification

| Check | Command | Expected |
|---|---|---|
| All LANs reachable both ways | ping between all three PCs | Success |
| Gateway is link-local | `ipconfig` on PC1 and PC3 | Default gateway `fe80::1` |
| Static address host works | PC2 reaches both other LANs | Success |
| Dual-stack intact | IPv4 pings still succeed | Success |

Save the final state to `output/10-final-state.txt`.

## What to write down

1. Your subnetting work for Task 1, including the arithmetic — not just the answer.
2. Which of the four Part 1 questions you got wrong, and what you'd misunderstood.
3. One sentence: what genuinely differs between IPv4 and IPv6 in how a host finds its gateway?

---

**Answer key:** [`solutions/README.md`](./solutions/) — completed addressing table, configs,
and answers to all questions. Look after you've attempted the derivation, not before.
