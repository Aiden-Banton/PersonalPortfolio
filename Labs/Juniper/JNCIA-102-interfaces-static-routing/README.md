# JNCIA-102 — Routing Tables, Route Preference, and Routing Instances

| | |
|---|---|
| **Covers** | JN0-106 — routing fundamentals, routing tables, route preference, static routing |
| **Tier** | T2 Applied |
| **Platform** | EVE-NG, vJunos-router |
| **Time** | 45–55 minutes |
| **Prerequisites** | JNCIA-101 (CLI, commit/rollback) |
| **Status** | built |

> **Platform gate:** requires three vJunos-router nodes booting concurrently. If JNCIA-101's
> boot test passed with one node, check RAM headroom before deploying three — vJunos-router
> is not a light image and three nodes plus the EVE-NG overhead is where a nested setup
> tends to run out of memory rather than fail to boot.

---

## Why this lab exists

Junos separates two things IOS blends together: the **routing table** (everything the
routing protocols know) and the **forwarding table** (what the hardware actually uses).
It also uses **route preference** where IOS uses administrative distance — same idea,
different numbers, and the numbers are exam material.

The third concept here, **routing instances**, has no IOS-CCNA equivalent at all. It is
Junos's VRF, and it is the foundation that every data-centre tenant-separation design is
built on. That makes it the bridge from this track into JNCIA-DC.

## Scenario

A regional ISP has three POP routers in a triangle. Two customers connect at R3 and must
be kept in completely separate routing domains — neither should be able to reach the other
even though they share the same physical router.

## Topology

```
                    [ R1 ]
                 lo0 1.1.1.1/32
                /                \
     10.0.12.0/30                10.0.13.0/30
       ge-0/0/0                    ge-0/0/1
              /                        \
        [ R2 ]                        [ R3 ]
     lo0 2.2.2.2/32                lo0 3.3.3.3/32
              \                        /
               \   10.0.23.0/30       /
        ge-0/0/1 ---------------- ge-0/0/0

     R3 ge-0/0/2 -> CUST-A instance -> 172.16.10.0/24
     R3 ge-0/0/3 -> CUST-B instance -> 172.16.20.0/24
```

## Addressing

| Device | Interface | Address |
|---|---|---|
| R1 | lo0.0 | 1.1.1.1/32 |
| R1 | ge-0/0/0.0 | 10.0.12.1/30 |
| R1 | ge-0/0/1.0 | 10.0.13.1/30 |
| R2 | lo0.0 | 2.2.2.2/32 |
| R2 | ge-0/0/0.0 | 10.0.12.2/30 |
| R2 | ge-0/0/1.0 | 10.0.23.1/30 |
| R3 | lo0.0 | 3.3.3.3/32 |
| R3 | ge-0/0/0.0 | 10.0.23.2/30 |
| R3 | ge-0/0/1.0 | 10.0.13.2/30 |
| R3 | ge-0/0/2.0 | 172.16.10.1/24 (instance CUST-A) |
| R3 | ge-0/0/3.0 | 172.16.20.1/24 (instance CUST-B) |

---

## Part 1 — Base connectivity and reading the routing table

**Task 1.** Configure all interfaces and loopbacks from the table. Commit on each device.

**Task 2.** Examine the routing table properly. Run all four and note what each adds:

```
root> show route
root> show route terse
root> show route 2.2.2.2 detail
root> show route forwarding-table
```

**Task 3.** In your notes, answer: `show route` and `show route forwarding-table` show
different things. What is the difference, and when would they disagree?

> The routing table (RIB) holds every route from every source, including ones that lost.
> The forwarding table (FIB) holds only the winners, pushed down to the packet-forwarding
> engine. They disagree whenever a better route exists that isn't installed — a route with
> an unreachable next hop, for instance, stays in the RIB and never reaches the FIB. IOS
> has the same split (`show ip route` vs `show ip cef`), but CCNA rarely makes you look at it.

| Check | Command | Expected | Save to |
|---|---|---|---|
| Interfaces up | `show interfaces terse` | All configured interfaces up/up | `output/01-interfaces-terse.txt` |
| RIB | `show route` | Direct, local and loopback routes | `output/02-show-route.txt` |
| FIB | `show route forwarding-table` | Fewer entries than the RIB | `output/03-forwarding-table.txt` |

---

## Part 2 — Static routes and preference

**Task 4.** Configure static routes on R1 so it can reach R2's and R3's loopbacks, and on
R2 and R3 so all three loopbacks are mutually reachable.

```
root# set routing-options static route 2.2.2.2/32 next-hop 10.0.12.2
```

**Task 5.** On R1, add a **second, less-preferred** route to `3.3.3.3/32` via R2, so that if
the direct R1–R3 link fails, traffic reroutes. Give it a preference worse than the default.

```
root# set routing-options static route 3.3.3.3/32 qualified-next-hop 10.0.12.2 preference 10
```

**Task 6.** Record the default preference values for direct, local, static and OSPF routes
from your own device output — do not look them up. `show route detail` prints the
preference for every route.

> Junos calls it **preference**; IOS calls it **administrative distance**. Same concept,
> lower-is-better in both, but the numbers differ — Junos static is 5, IOS static is 1.
> Assuming they match is a reliable way to get an exam question wrong.

**Task 7.** Test the failover. Disable R1's `ge-0/0/1` and confirm traffic to `3.3.3.3`
reroutes through R2, then re-enable it and confirm it reverts.

```
root# set interfaces ge-0/0/1 disable
root# commit
```

| Check | Command | Expected | Save to |
|---|---|---|---|
| Both routes present | `show route 3.3.3.3 detail` | Active route + backup with preference 10 | `output/04-qualified-next-hop.txt` |
| Preferences observed | `show route detail` | Direct/local/static values recorded | `output/05-route-preferences.txt` |
| Failover works | `traceroute 3.3.3.3` with link down | Path via R2 | `output/06-failover-path.txt` |
| Reverts on restore | `show route 3.3.3.3` | Back to direct | `output/07-failback.txt` |

---

## Part 3 — Routing instances (the DC bridge)

**Task 8.** On R3, create two `virtual-router` routing instances and place the customer
interfaces in them:

```
root# set routing-instances CUST-A instance-type virtual-router
root# set routing-instances CUST-A interface ge-0/0/2.0
root# set routing-instances CUST-B instance-type virtual-router
root# set routing-instances CUST-B interface ge-0/0/3.0
root# commit
```

**Task 9.** Verify each instance has its own routing table, separate from the main one:

```
root> show route table CUST-A.inet.0
root> show route table CUST-B.inet.0
root> show route table inet.0
```

**Task 10.** Prove the separation is real. From a host in CUST-A, attempt to reach the
CUST-B subnet. It must fail — and it must fail *even though both interfaces are on the same
physical router*.

**Task 11.** Answer in your notes: why is this the foundation of multi-tenant data centre
design, and what would you need to add to let CUST-A and CUST-B reach a shared service
without being able to reach each other?

> This is where the track starts pointing at JNCIA-DC. A virtual router is one routing table
> in one box; a data-centre fabric is many of them across many boxes, stitched together with
> EVPN-VXLAN so a tenant's separation holds across the whole fabric. The isolation you just
> demonstrated on one router is the unit that design is made of. The shared-service question
> is answered by route leaking between instances — which is the next concept up.

| Check | Command | Expected | Save to |
|---|---|---|---|
| Instances exist | `show route instance summary` | CUST-A and CUST-B, type virtual-router | `output/08-route-instances.txt` |
| Separate tables | `show route table CUST-A.inet.0` | Only CUST-A's routes | `output/09-instance-tables.txt` |
| Isolation holds | ping CUST-A host → CUST-B subnet | Fails | `output/10-tenant-isolation.txt` |
| Main table unaffected | `show route table inet.0` | No customer routes | `output/11-main-table-clean.txt` |

---

## Final verification

| Check | Expected |
|---|---|
| All three loopbacks mutually reachable | Success from each router |
| Backup path takes over on link failure | Traceroute shows the alternate path |
| Two routing instances, isolated | Cross-instance ping fails |
| Main routing table clean | No customer prefixes leaked |

Save to `output/12-final-state.txt`.

## What to write down

1. The preference values you observed, in a table, from your own output.
2. What `show route` showed that `show route forwarding-table` did not, and why.
3. One sentence on why routing instances matter for a data centre, in your own words.

---

**Next:** [JNCIA-103 — OSPF and BGP: the IP Fabric Underlay](../JNCIA-103-ospf-bgp-underlay/)
