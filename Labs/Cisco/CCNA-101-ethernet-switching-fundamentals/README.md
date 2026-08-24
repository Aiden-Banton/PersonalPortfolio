# CCNA-101 — Ethernet Switching Fundamentals

| | |
|---|---|
| Domain | 1.0 Network Fundamentals |
| Covers | 1.3, 1.4, 1.13 |
| Tier | T1 Foundational |
| Platform | Packet Tracer |
| Duration | 30 min |
| Prerequisites | none |
| Status | built |

## What you'll build

A single switch with three connected hosts. You'll watch the MAC address table
populate as frames move through the switch, then diagnose an interface that's been
deliberately set to a duplex mismatch — a fault that produces a specific, recognisable
signature in `show interfaces` output.

## Objectives

- Predict whether a given frame gets flooded or forwarded, based on MAC table state
- Read `show mac address-table` and explain how each entry got there
- Identify a duplex/speed mismatch from interface counters, not from a symptom alone
- Distinguish a collision domain from a broadcast domain on a switched segment

## Topology

```
        +--------+
        |  SW1   |
        +--------+
        /   |    \
     Fa0/1 Fa0/2  Fa0/3
      /      |      \
   PC1     PC2      PC3
```

*(topology.svg pending — see note in `topology/`)*

| A device | A interface | B device | B interface | Link type |
|---|---|---|---|---|
| SW1 | Fa0/1 | PC1 | NIC | Copper, access |
| SW1 | Fa0/2 | PC2 | NIC | Copper, access |
| SW1 | Fa0/3 | PC3 | NIC | Copper, access |

## Addressing

| Device | Interface | IPv4 Address | Mask / Prefix | Default Gateway | Notes |
|--------|-----------|--------------|----------------|------------------|-------|
| SW1    | VLAN 1    | 10.1.99.1    | /24            | —                | Management only |
| PC1    | NIC       | 10.1.99.11   | /24            | 10.1.99.1        | |
| PC2    | NIC       | 10.1.99.12   | /24            | 10.1.99.1        | |
| PC3    | NIC       | 10.1.99.13   | /24            | 10.1.99.1        | |

## Before you start

You need Packet Tracer 8.2+ with one 2960-series switch and three end devices. No
`.pkt` file is bundled yet — build the topology above from scratch; it's three cable
runs and three IP assignments. Estimated time: 30 minutes including the fault task.

## Tasks

1. **Cable and address the topology.** Connect PC1–PC3 to SW1 as shown, assign the
   addresses above, and confirm all three hosts are on the same subnet.

2. **Clear and observe the MAC address table.** Before generating any traffic, run
   `show mac address-table` on SW1. It should be empty (aside from any default
   entries). Ping PC1 → PC2, then immediately re-check the table.

   - What entries appeared, on which interfaces, and why did the switch learn them
     from that specific ping rather than needing traffic in both directions?

3. **Force a flood, not a forward.** Clear the MAC table again (`clear mac
   address-table dynamic`). This time ping PC1 → PC3, but check the table
   *immediately* after the first ICMP echo request leaves PC1, before PC3 has replied.

   - At that instant, does SW1 know PC3's MAC address? What does it do with the
     frame as a result?

4. **Diagnose an injected fault.** Set Fa0/2 on SW1 to `duplex half` while leaving
   PC2's NIC on auto (which will negotiate full). Generate sustained traffic between
   PC2 and the other two hosts (a continuous ping is enough in Packet Tracer's
   simulation, or just several manual pings back to back).

   - Run `show interfaces fa0/2` and identify which counters move in a way that's
     specific to a duplex mismatch, as opposed to a general "something's wrong" signal.

5. **Fix it and confirm.** Set Fa0/2 back to `duplex auto`, re-run the traffic test,
   and confirm the counters from Task 4 stop incrementing.

## Verification

- `show mac address-table` — after Task 2, SW1 should show exactly two dynamic
  entries (PC1 and PC2's MACs) on Fa0/1 and Fa0/2 respectively, with VLAN 1.
- `show interfaces fa0/1`, `fa0/2`, `fa0/3` — all should show `line protocol is up`,
  `full-duplex` (before Task 4), and 100Mb/s.
- Three reachability tests: PC1 → PC2, PC1 → PC3, PC2 → PC3 — all succeed with 0%
  loss once cabling and addressing are correct.
- **Prove-it check:** after Task 4's fault injection, `show interfaces fa0/2` must
  show late collisions and/or a runts/CRC-adjacent counter incrementing under load —
  not just "high traffic." A duplex mismatch has a specific signature; if you can't
  point to the specific counter, the diagnosis isn't proven yet.

## What breaks

- **Pinging before the switch has any MAC entries and reading the first attempt as a
  failure.** The first frame to an unknown destination floods; it doesn't fail. Give
  ARP and the initial flood a moment before concluding something's wrong.
- **Confusing a collision domain with a broadcast domain on a switched topology.**
  Every switch port here is its own collision domain (full-duplex, point-to-point);
  all three hosts still share one broadcast domain (VLAN 1). Mixing these up is a
  common multiple-choice trap.
- **Setting only one side to half-duplex and expecting an obvious link failure.** A
  duplex mismatch doesn't take the link down — it degrades it under load. That's
  what makes it a diagnostic exercise rather than a "the cable's unplugged" one.
- **Reading `show mac address-table` right after `clear` and assuming the switch
  forgot the port exists.** An empty table is a table with no learned addresses, not
  a broken switch — the port is still up.

## How this maps to the exam

MAC learning/aging/flooding shows up as straightforward multiple choice ("what does
the switch do when it receives a frame for an unknown destination MAC"). Duplex
mismatch diagnosis appears as an interface-counter-reading item, sometimes paired
with a `show interfaces` output to interpret rather than a live device.

## Solutions

Full configuration steps, expected output for every verification command, and the
fault-injection commands are in [`solutions/`](./solutions/). Work the lab first.
