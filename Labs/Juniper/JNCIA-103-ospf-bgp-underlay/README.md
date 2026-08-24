# JNCIA-103 — OSPF and eBGP: Building an IP Fabric Underlay

| | |
|---|---|
| **Covers** | JN0-106 — OSPF, BGP fundamentals, protocol-independent routing. **Bridges to JN0-281 (JNCIA-DC)** — spine-leaf, underlay, eBGP-to-the-leaf |
| **Tier** | T2 Applied |
| **Platform** | EVE-NG, vJunos-router |
| **Time** | 60–70 minutes |
| **Prerequisites** | JNCIA-101 and JNCIA-102 |
| **Status** | built |

> **Platform gate:** four vJunos-router nodes. This is the heaviest lab in the track. If
> three nodes were tight in JNCIA-102, run the OSPF half (two spines, one leaf) and defer
> the eBGP half rather than fighting the memory ceiling mid-lab.

---

## Why this lab exists

This is the lab where the JNCIA-Junos track stops being generic routing practice and starts
being data-centre preparation.

A modern data-centre fabric is a spine-leaf topology with an **underlay** — a simple,
robust routing protocol whose only job is to make every switch's loopback reachable from
every other switch's loopback — and an **overlay** (EVPN-VXLAN) that carries actual tenant
traffic on top. JNCIA-DC tests the underlay concepts directly and expects you to understand
why the industry mostly settled on eBGP for it.

You are going to build the same underlay twice: once with OSPF, once with eBGP, on the same
topology. Building it twice is the point — the comparison is what the exam actually asks about.

## Topology — a two-spine, two-leaf fabric

```
        [ SPINE1 ]                    [ SPINE2 ]
      lo0 10.0.0.1/32               lo0 10.0.0.2/32
        /        \                    /        \
       /          \                  /          \
   .1 /            \ .1          .1 /            \ .1
     /              \              /              \
    /                \            /                \
[ LEAF1 ]          [ LEAF2 ]  (each leaf connects to BOTH spines)
lo0 10.0.0.11/32   lo0 10.0.0.12/32

Fabric links, all /31:
  SPINE1 ge-0/0/0 -- 10.1.0.0/31 -- LEAF1 ge-0/0/0
  SPINE1 ge-0/0/1 -- 10.1.0.2/31 -- LEAF2 ge-0/0/0
  SPINE2 ge-0/0/0 -- 10.1.0.4/31 -- LEAF1 ge-0/0/1
  SPINE2 ge-0/0/1 -- 10.1.0.6/31 -- LEAF2 ge-0/0/1

Note: leaves do NOT connect to each other. Spines do NOT connect to each other.
That is the defining property of the topology, not an omission.
```

**Why /31s?** A point-to-point fabric link needs exactly two addresses. A /30 wastes two
per link, and a fabric has a lot of links. /31 on point-to-point is standard practice in
data-centre designs for exactly this reason.

## Addressing

| Device | Interface | Address |
|---|---|---|
| SPINE1 | lo0.0 | 10.0.0.1/32 |
| SPINE1 | ge-0/0/0.0 | 10.1.0.0/31 |
| SPINE1 | ge-0/0/1.0 | 10.1.0.2/31 |
| SPINE2 | lo0.0 | 10.0.0.2/32 |
| SPINE2 | ge-0/0/0.0 | 10.1.0.4/31 |
| SPINE2 | ge-0/0/1.0 | 10.1.0.6/31 |
| LEAF1 | lo0.0 | 10.0.0.11/32 |
| LEAF1 | ge-0/0/0.0 | 10.1.0.1/31 |
| LEAF1 | ge-0/0/1.0 | 10.1.0.5/31 |
| LEAF2 | lo0.0 | 10.0.0.12/32 |
| LEAF2 | ge-0/0/0.0 | 10.1.0.3/31 |
| LEAF2 | ge-0/0/1.0 | 10.1.0.7/31 |

---

## Part 1 — Build the fabric

**Task 1.** Configure all interfaces and loopbacks. Commit and verify every link pings.

**Task 2.** Before configuring any protocol, state the underlay's job in one sentence in
your notes. If your sentence mentions tenant traffic or VLANs, it is wrong — try again.

> The underlay's only job is loopback-to-loopback reachability between every device in the
> fabric. Nothing else. Tenant traffic never rides the underlay directly; it rides a VXLAN
> tunnel between two loopbacks, and the underlay's sole responsibility is that those
> loopbacks can find each other. Keeping that boundary clear is most of what makes fabric
> designs comprehensible.

| Check | Command | Expected | Save to |
|---|---|---|---|
| All links up | `show interfaces terse` on all four | Every fabric interface up/up | `output/01-fabric-interfaces.txt` |
| Direct connectivity | ping across each /31 | 4 links, all succeed | `output/02-link-pings.txt` |

---

## Part 2 — Underlay attempt 1: OSPF

**Task 3.** Configure OSPF area 0 on every fabric interface and every loopback:

```
root# set protocols ospf area 0.0.0.0 interface ge-0/0/0.0
root# set protocols ospf area 0.0.0.0 interface ge-0/0/1.0
root# set protocols ospf area 0.0.0.0 interface lo0.0 passive
root# commit
```

**Task 4.** Set the fabric interfaces to `interface-type p2p`. Explain in your notes what
this avoids.

> On a broadcast-type OSPF interface the routers elect a DR and BDR, which exists to reduce
> adjacency count on a shared segment. A /31 point-to-point link has exactly two routers on
> it, so the election is pure overhead and it costs you time on every link event. Declaring
> the interface p2p skips it. In a fabric with many links, that adds up.

**Task 5.** Verify every device sees every other loopback.

**Task 6.** Confirm equal-cost multipath: LEAF1 should reach LEAF2's loopback via *both*
spines, not one.

| Check | Command | Expected | Save to |
|---|---|---|---|
| Adjacencies | `show ospf neighbor` | 2 per leaf, 2 per spine, all Full | `output/03-ospf-neighbors.txt` |
| All loopbacks known | `show route 10.0.0.0/24` on LEAF1 | All four loopbacks | `output/04-ospf-loopbacks.txt` |
| ECMP present | `show route 10.0.0.12/32 detail` on LEAF1 | Two next hops | `output/05-ospf-ecmp.txt` |
| Leaf-to-leaf reachability | `ping 10.0.0.12 source 10.0.0.11` | Success | `output/06-ospf-leaf-to-leaf.txt` |

---

## Part 3 — Underlay attempt 2: eBGP

Now remove OSPF and build the same reachability with eBGP. **Do not delete your OSPF output
files** — the comparison is the deliverable.

**Task 7.** Deactivate OSPF rather than deleting it, so you can bring it back:

```
root# deactivate protocols ospf
root# commit
```

> `deactivate` keeps configuration in the file but marks it inactive. It shows as
> `inactive:` in `show configuration`. There is no clean IOS equivalent, and it is
> genuinely useful — you can turn a protocol off for a test without losing the config or
> having to remember how to type it again.

**Task 8.** Assign a unique private AS to each device — spines and leaves each get their own:

| Device | AS |
|---|---|
| SPINE1 | 65001 |
| SPINE2 | 65002 |
| LEAF1 | 65011 |
| LEAF2 | 65012 |

**Task 9.** Configure eBGP between each leaf and each spine, peering on the **interface
addresses**, not the loopbacks.

```
root# set routing-options autonomous-system 65011
root# set protocols bgp group UNDERLAY type external
root# set protocols bgp group UNDERLAY neighbor 10.1.0.0 peer-as 65001
root# set protocols bgp group UNDERLAY neighbor 10.1.0.4 peer-as 65002
root# commit
```

**Task 10.** Answer before continuing: in JNCIA-102 you peered nothing, and here you are
told to peer on interface addresses rather than loopbacks. Why is that the right choice for
a fabric underlay, when loopback peering is the norm for iBGP?

> Loopback peering requires the loopbacks to already be reachable — which is the very thing
> the underlay is being built to provide. Peering on directly connected interface addresses
> removes the circular dependency: the link is up, so the peering comes up, so the loopbacks
> get advertised. This is why eBGP-to-the-leaf is a fabric convention rather than an
> arbitrary preference.

**Task 11.** Advertise the loopbacks into BGP using an export policy. Junos will not
advertise anything from BGP without one — this is a hard difference from IOS and a
frequent exam point.

```
root# set policy-options policy-statement EXPORT-LOOPBACK term 1 from protocol direct
root# set policy-options policy-statement EXPORT-LOOPBACK term 1 from route-filter 10.0.0.11/32 exact
root# set policy-options policy-statement EXPORT-LOOPBACK term 1 then accept
root# set protocols bgp group UNDERLAY export EXPORT-LOOPBACK
root# commit
```

> **This is the thing to remember from this lab.** IOS advertises what `network` statements
> tell it to. Junos BGP advertises *only* what an export policy accepts, and the default
> policy for BGP is to advertise nothing but received BGP routes. Configure a perfect BGP
> session with no export policy and you get a working peering that carries zero prefixes —
> which looks like a broken session and isn't.

**Task 12.** Enable multipath so the leaf uses both spines, then verify.

```
root# set protocols bgp group UNDERLAY multipath
```

| Check | Command | Expected | Save to |
|---|---|---|---|
| BGP sessions established | `show bgp summary` | 2 peers per device, state Established | `output/07-bgp-summary.txt` |
| Loopbacks learned via BGP | `show route protocol bgp` on LEAF1 | Remote loopbacks present | `output/08-bgp-routes.txt` |
| Multipath active | `show route 10.0.0.12/32 detail` | Two next hops | `output/09-bgp-ecmp.txt` |
| Advertised prefixes | `show route advertising-protocol bgp 10.1.0.0` | The loopback | `output/10-bgp-advertised.txt` |
| Leaf-to-leaf works | `ping 10.0.0.12 source 10.0.0.11` | Success | `output/11-bgp-leaf-to-leaf.txt` |

---

## Part 4 — The comparison

**Task 13.** Fail one spine (`set interfaces ge-0/0/0 disable` on SPINE1's leaf-facing
links, or shut the whole node) and measure how long leaf-to-leaf connectivity takes to
recover under eBGP. Then reactivate OSPF, deactivate BGP, and repeat.

**Task 14.** Complete this table in `output/notes.md` from your own observations:

| | OSPF underlay | eBGP underlay |
|---|---|---|
| Config lines per leaf | | |
| Adjacency/session count | | |
| Advertisement requires a policy? | | |
| Convergence after spine failure | | |
| What happens when you add a 3rd spine | | |
| Which scales better to 100 leaves, and why | | |

**Task 15.** Write two or three sentences answering: **why did the data-centre industry
largely converge on eBGP for fabric underlays rather than OSPF?** Answer from what you
observed, not from what you have read.

> Things worth having noticed: OSPF floods LSAs fabric-wide, so every device processes every
> topology change whether or not it is affected; BGP only tells its direct peers. OSPF's
> single area 0 across a large fabric means one flooding domain. BGP's per-device AS makes
> the topology explicit in the config and loop prevention automatic via AS-path. Against
> that, OSPF is far less configuration and needs no policy to advertise anything — which is
> exactly why it is the easier choice at small scale and the harder one at large scale.

---

## Where this goes next

You have now built a fabric underlay. That is roughly a third of the JNCIA-DC (JN0-281)
blueprint. The remaining pieces:

| Concept | What it adds | Where it builds on this lab |
|---|---|---|
| VXLAN | Tunnels tenant traffic loopback-to-loopback | Your underlay is what makes those loopbacks reachable |
| EVPN | The control plane telling VXLAN where MACs live | Runs as an address family over BGP sessions like the ones you just built |
| Tenant separation | Per-tenant isolation across the fabric | The routing instances from JNCIA-102, extended fabric-wide |
| IRB interfaces | Routing between VLANs inside the fabric | New concept |

Every one of those assumes the underlay works. That is why it is worth building twice.

---

## Final verification

| Check | Expected |
|---|---|
| Every loopback reachable from every device | Success under both protocols |
| ECMP across both spines | Two next hops in both cases |
| Spine failure survivable | Connectivity holds, path shifts |
| Comparison table complete | Filled from your own output |

Save to `output/12-final-state.txt`.
