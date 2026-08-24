# Addressing — CCNA-101 Ethernet Switching Fundamentals

This lab is about switching behaviour, not addressing, so every value below is
pre-calculated. Type it in and move on.

| Device | Interface | IPv4 Address | Mask / Prefix | Default Gateway | Notes |
|--------|-----------|--------------|----------------|------------------|-------|
| SW1    | VLAN 1    | 10.1.99.1    | /24            | —                | Management only, not used for switching tasks |
| PC1    | NIC       | 10.1.99.11   | /24            | 10.1.99.1        | |
| PC2    | NIC       | 10.1.99.12   | /24            | 10.1.99.1        | |
| PC3    | NIC       | 10.1.99.13   | /24            | 10.1.99.1        | |

All hosts sit on the same broadcast domain — VLAN 1, no trunking or inter-VLAN routing
in this lab. That's deliberate: it isolates switching concepts from VLAN configuration,
which belongs to a later lab.
