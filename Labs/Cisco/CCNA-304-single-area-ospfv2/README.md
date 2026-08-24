# CCNA-304 — Single-Area OSPFv2: Adjacency, Cost, and Why Neighbours Get Stuck

| | |
|---|---|
| **Covers** | 3.4 (single-area OSPFv2), 3.1–3.2 (routing table, forwarding decision) |
| **Tier** | T2 Applied — objectives given, you find the commands |
| **Platform** | Packet Tracer 8.2+ |
| **Time** | 50–60 minutes |
| **Prerequisites** | CCNA-301 (static routes, AD, longest match) helps but is not required |
| **Status** | built |

---

## Why this lab exists

Most OSPF labs stop when `show ip ospf neighbor` prints `FULL`. That is the least
interesting moment in OSPF. The exam — and real troubleshooting — lives in the states
*before* FULL, and in the question of which path OSPF actually picks once it gets there.

This lab spends its second half deliberately breaking adjacencies in the three ways OSPF
actually breaks, and asks you to identify each one from the symptom before you fix it.

## Scenario

Bramford Logistics runs three sites — a distribution hub and two depots — connected in a
triangle so that either depot can still reach the hub if one link fails. Each site has a
single user LAN. The network team has decided on OSPF single-area (area 0) because the
network is small and the routing needs to converge without manual route maintenance.

You are configuring OSPF from scratch on all three routers, then diagnosing three faults
that the previous contractor left behind.

## Topology

```
                        [ R1 - Hub ]
                     Lo0 1.1.1.1/32
                    /                \
       192.168.12.0/30              192.168.13.0/30
       G0/0/0 .1 --- .2 G0/0/0      G0/0/1 .1 --- .2 G0/0/0
              /                              \
    [ R2 - Depot A ]                   [ R3 - Depot B ]
    Lo0 2.2.2.2/32                     Lo0 3.3.3.3/32
              \                              /
               \      192.168.23.0/30       /
        G0/0/1 .1 ------------------ .2 G0/0/1

   R1 G0/0/2 -- 10.1.10.0/24 -- PC1
   R2 G0/0/2 -- 10.2.10.0/24 -- PC2
   R3 G0/0/2 -- 10.3.10.0/24 -- PC3
```

Topology diagram: not yet drawn — the ASCII diagram above is canonical for now. Runnable
file: [`topology/CCNA-304.pkt`](./topology/) *(add when built)*

## Addressing

Pre-calculated — this lab is about the protocol, not the arithmetic. Full table in
[`addressing.md`](./addressing.md).

| Device | Interface | Address | Mask |
|---|---|---|---|
| R1 | Loopback0 | 1.1.1.1 | /32 |
| R1 | G0/0/0 (to R2) | 192.168.12.1 | /30 |
| R1 | G0/0/1 (to R3) | 192.168.13.1 | /30 |
| R1 | G0/0/2 (LAN) | 10.1.10.1 | /24 |
| R2 | Loopback0 | 2.2.2.2 | /32 |
| R2 | G0/0/0 (to R1) | 192.168.12.2 | /30 |
| R2 | G0/0/1 (to R3) | 192.168.23.1 | /30 |
| R2 | G0/0/2 (LAN) | 10.2.10.1 | /24 |
| R3 | Loopback0 | 3.3.3.3 | /32 |
| R3 | G0/0/0 (to R1) | 192.168.13.2 | /30 |
| R3 | G0/0/1 (to R2) | 192.168.23.2 | /30 |
| R3 | G0/0/2 (LAN) | 10.3.10.1 | /24 |

Hosts use `.100` in their LAN with the router as gateway.

---

## Part 1 — Build the adjacencies

**Task 1.** Configure interface addressing on all three routers per the table above.
Bring every interface up. Do not configure OSPF yet.

**Task 2.** Verify Layer 3 connectivity across each point-to-point link before touching
OSPF. If R1 cannot ping 192.168.12.2, OSPF will not form and you will waste time looking
at the wrong layer.

> Diagnosing bottom-up is not a stylistic preference here. Roughly half of "OSPF isn't
> working" turns out to be an interface, a mask, or a shutdown — none of which OSPF will
> tell you about directly.

**Task 3.** Enable OSPFv2 process 1 on all three routers, all interfaces in **area 0**.

Set the router ID explicitly on each router to its loopback address. Do not rely on the
automatic selection.

> **Why explicitly?** OSPF picks its RID from the highest loopback, or the highest active
> physical interface if no loopback exists — but only at process start. Add a loopback
> later and the RID doesn't change until the process is cleared, so the RID you see and
> the RID you'd predict stop matching. Setting it removes that whole class of confusion.

**Task 4.** Advertise the LAN interfaces into OSPF, but configure them so they do not
attempt to form adjacencies. There are no OSPF routers on the user LANs, and sending
hellos there is wasted traffic and a small attack surface.

**Task 5.** Verify full adjacency between all three routers.

**Checkpoint** — before continuing, confirm:

| Check | Command | Expected | Save to |
|---|---|---|---|
| Three neighbours, all FULL | `show ip ospf neighbor` | 2 neighbours per router, state FULL | `output/01-ospf-neighbor-baseline.txt` |
| RIDs are the loopbacks | `show ip protocols` | Router ID 1.1.1.1 / 2.2.2.2 / 3.3.3.3 | `output/02-show-ip-protocols.txt` |
| LAN interfaces passive | `show ip ospf interface brief` | LAN interfaces listed, no neighbours on them | `output/03-ospf-interface-brief.txt` |
| All LANs reachable | `ping` PC1 → PC2, PC1 → PC3 | Success | `output/04-end-to-end-ping.txt` |

---

## Part 2 — Cost and path selection

**Task 6.** From R2, determine which path traffic to R3's LAN (10.3.10.0/24) currently
takes. Record the path and the metric.

**Task 7.** All three links are the same speed, so OSPF sees three equal-cost options
around the triangle. Change the cost so that R2's traffic to R3's LAN prefers the direct
R2–R3 link rather than transiting the hub, **without** changing the bandwidth statement
on any interface.

> **Why not `bandwidth`?** Because `bandwidth` is also used by QoS and by other routing
> protocols, and by interface statistics. Changing it to influence OSPF has side effects
> somewhere else. `ip ospf cost` changes exactly one thing.

**Task 8.** Verify the path changed, and explain in one sentence in your notes why the
routing table now shows what it does.

| Check | Command | Expected | Save to |
|---|---|---|---|
| Path before change | `traceroute` from R2 to 10.3.10.100 | Records the transit path | `output/05-traceroute-before.txt` |
| Cost applied | `show ip ospf interface G0/0/1` | Cost reflects your change | `output/06-ospf-interface-cost.txt` |
| Path after change | `traceroute` from R2 to 10.3.10.100 | Direct path, one hop fewer | `output/07-traceroute-after.txt` |

---

## Part 3 — Three faults, three symptoms

Load the fault version of the topology (`topology/CCNA-304-faults.pkt`) or apply the
fault list in [`solutions/faults.md`](./solutions/) to your working config.

For each fault: **identify the state first, predict the cause, then verify.** Write your
prediction down before you run the confirming command. This is the part that transfers to
the exam.

### Fault A — R1 and R2 stuck in EXSTART/EXCHANGE

**Symptom:** `show ip ospf neighbor` on R1 shows R2 in `EXSTART` or `EXCHANGE` and it
never progresses. Hellos are clearly getting through, because the neighbour is listed at all.

**Your task:** name the mismatch before checking. Then confirm it, fix it, and capture the
adjacency reaching FULL.

> The state itself narrows this a long way. Reaching EXSTART means hello parameters already
> matched — area, timers, authentication and subnet all agreed, or you would still be in
> INIT or have no neighbour at all. What EXSTART begins is database exchange over DD
> packets, and the one thing that breaks *there* and nowhere earlier is the interface MTU.

| Check | Command | Save to |
|---|---|---|
| Stuck state | `show ip ospf neighbor` | `output/08-faultA-stuck-exstart.txt` |
| MTU comparison | `show interface G0/0/0` on both ends | `output/09-faultA-mtu-mismatch.txt` |
| Resolved | `show ip ospf neighbor` | `output/10-faultA-resolved.txt` |

### Fault B — R1 and R3 never see each other at all

**Symptom:** no neighbour entry on either side. Interfaces are up/up and the two routers
can ping each other.

**Your task:** there are three plausible causes that all produce exactly this symptom.
List all three, then work out which one applies here.

> Ping working proves Layer 3 is fine, which eliminates a whole category and is the reason
> the task says to ping first. What remains are the parameters OSPF checks *before* it will
> even create a neighbour entry: mismatched hello/dead timers, mismatched area ID, or one
> side not running OSPF on that interface at all.

| Check | Command | Save to |
|---|---|---|
| No neighbour | `show ip ospf neighbor` | `output/11-faultB-no-neighbor.txt` |
| Timers and area | `show ip ospf interface G0/0/1` both ends | `output/12-faultB-parameter-mismatch.txt` |
| Resolved | `show ip ospf neighbor` | `output/13-faultB-resolved.txt` |

### Fault C — adjacency is FULL but one LAN is unreachable

**Symptom:** all neighbours FULL, but PC1 cannot reach 10.3.10.100. R3 can ping its own LAN.

**Your task:** the adjacency is healthy, so the problem is not adjacency. Where in the
OSPF configuration can a network be missing while everything else looks correct?

| Check | Command | Save to |
|---|---|---|
| Route absent | `show ip route ospf` on R1 | `output/14-faultC-missing-route.txt` |
| What R3 advertises | `show ip ospf database` / `show ip protocols` on R3 | `output/15-faultC-advertisement.txt` |
| Resolved | `show ip route ospf` on R1 | `output/16-faultC-resolved.txt` |

---

## Final verification

| Check | Command | Expected |
|---|---|---|
| All adjacencies FULL | `show ip ospf neighbor` on all three | 2 FULL neighbours each |
| All LANs in every table | `show ip route ospf` | Three /24 LAN routes on each router |
| Preferred path holds | `traceroute` R2 → R3 LAN | Direct link |
| Full reachability | ping between all three PCs | Success |

Save the final state to `output/17-final-state.txt`.

## What to write down

Before you close the lab, answer these in `output/notes.md`. They are the difference
between having done a lab and being able to talk about it:

1. Which OSPF state told you the most, and why did the state itself narrow the cause?
2. What would you check first next time — and would that have been faster?
3. Fault B had three candidate causes. What single command distinguishes them fastest?

---

**Answer key:** [`solutions/README.md`](./solutions/) — full configs, fault list, and
expected output. Look after, not before.
