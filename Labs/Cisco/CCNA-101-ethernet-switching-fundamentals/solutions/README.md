# CCNA-101 — Solutions

Attempt the lab first. This is the answer key.

## Task 1 — Cabling and addressing

Straightforward copper connections, PC1–PC3 to SW1 Fa0/1–Fa0/3, static IPs per the
addressing table. No config needed on SW1 beyond default (VLAN 1, all ports up).

## Task 2 — MAC learning on a bidirectional ping

After `PC1 → PC2` ping:

```
SW1# show mac address-table
Vlan    Mac Address       Type        Ports
----    -----------       ----        -----
   1    0050.0001.0001    DYNAMIC     Fa0/1
   1    0050.0002.0002    DYNAMIC     Fa0/2
```

Both entries appear because a ping is bidirectional: PC1's request teaches SW1 PC1's
MAC on Fa0/1, and PC2's reply teaches SW1 PC2's MAC on Fa0/2 in the same exchange.

## Task 3 — Flood before the table is populated

Immediately after the first echo request leaves PC1 (before PC3 replies), SW1 knows
PC1's MAC (source-learned from the frame that just arrived) but not PC3's — so it
floods the frame out every port except Fa0/1 (unknown-unicast flood), rather than
forwarding it only to Fa0/3.

## Task 4 — Duplex mismatch signature

```
SW1(config)# interface fa0/2
SW1(config-if)# duplex half
```

With PC2's NIC negotiating full and Fa0/2 forced to half, `show interfaces fa0/2`
under load shows:

```
SW1# show interfaces fa0/2
FastEthernet0/2 is up, line protocol is up
  ...
  Half-duplex, 100Mb/s
  ...
     1524 packets input, 198122 bytes
     0 input errors, 0 CRC, 0 frame, 0 overrun
     3402 packets output, 441800 bytes
     412 late collisions, 0 collisions
```

**Late collisions** are the specific signature — they occur when a half-duplex side
detects a collision after it should no longer be possible, which is exactly what
happens when the other side is actually sending full-duplex. Generic congestion
does not produce late collisions; a duplex mismatch does. This is what distinguishes
this fault from "the link is just busy."

## Task 5 — Fix

```
SW1(config)# interface fa0/2
SW1(config-if)# duplex auto
```

Re-run traffic; the late-collision counter stops incrementing (it does not reset to
zero on its own — confirm it stays flat under a fresh traffic burst rather than
expecting the counter itself to clear).

## Configs

See `configs/sw1.txt` — SW1 requires no explicit configuration beyond the VLAN 1
management address and the duplex fault/fix commands used in Tasks 4–5; PCs are
addressed via their IP configuration dialog, not CLI.
