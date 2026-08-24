# CCNA-207 — Addressing

Source of truth for the README and the printable PDF. If these disagree, this file wins.


Pre-calculated. Full table in [`addressing.md`](./addressing.md).

| Device | Interface / VLAN | Address | Notes |
|---|---|---|---|
| R1 | G0/0/0.99 | 10.20.99.1/24 | Management gateway |
| R1 | G0/0/0.10 | 10.20.10.1/24 | Staff gateway, DHCP pool |
| R1 | G0/0/0.20 | 10.20.20.1/24 | Guest gateway, DHCP pool |
| WLC1 | Management | 10.20.99.10/24 | Gateway 10.20.99.1 |
| AP1 | — | DHCP from VLAN 99 | Joins WLC over the management network |
| PC1 | — | DHCP from VLAN 10 | Wired reference host |
