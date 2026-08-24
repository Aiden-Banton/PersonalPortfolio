# CCNA-304 — Addressing

Source of truth for the README and the printable PDF. If these disagree, this file wins.


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
