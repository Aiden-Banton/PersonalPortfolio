# CCNA-108 — Addressing

Source of truth for the README and the printable PDF. If these disagree, this file wins.

> **This table is intentionally blank.** Deriving it is the learning objective of this lab.
> Per the addressing standard, labs tagged 1.6–1.9 give the student blank cells; every other
> lab gives the addressing pre-calculated. The completed version is in `solutions/README.md`.

**Allocation:** `2001:db8:acad::/48`

Subnet into /64s. Assign the **lowest available subnet IDs in order**: LAN A, LAN B, LAN C,
then the two link networks.

## Router interfaces

| Segment | Prefix (/64) | Device | Interface | Global unicast address | Link-local |
|---|---|---|---|---|---|
| LAN A | `2001:db8:acad:____::/64` | R1 | G0/0/2 | `____::1/64` | `fe80::1` |
| LAN B | `2001:db8:acad:____::/64` | R2 | G0/0/2 | `____::1/64` | `fe80::2` |
| LAN C | `2001:db8:acad:____::/64` | R3 | G0/0/2 | `____::1/64` | `fe80::3` |
| Link R1–R2 | `2001:db8:acad:____::/64` | R1 | G0/0/0 | `____::1/64` | `fe80::1` |
| Link R1–R2 | (same prefix) | R2 | G0/0/0 | `____::2/64` | `fe80::2` |
| Link R1–R3 | `2001:db8:acad:____::/64` | R1 | G0/0/1 | `____::1/64` | `fe80::1` |
| Link R1–R3 | (same prefix) | R3 | G0/0/0 | `____::2/64` | `fe80::3` |

## Hosts

| Host | LAN | Method | Address | Default gateway |
|---|---|---|---|---|
| PC1 | LAN A | SLAAC | *(auto)* | `____` |
| PC2 | LAN B | Static | `____::100/64` | `____` |
| PC3 | LAN C | SLAAC | *(auto)* | `____` |

> The gateway column is worth pausing on. It is not the router's global unicast address —
> work out why before you fill it in.

## Show your working

Record the arithmetic for "how many /64 subnets does a /48 yield," not just the number.
The exam asks for the reasoning as often as the result.
