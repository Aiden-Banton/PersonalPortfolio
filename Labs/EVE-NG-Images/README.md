# EVE-NG Device Images

Which node images the EVE-NG VM on Node 1 (`10.10.20.50`) has available, and which labs in [`../`](../README.md) need them. The image files themselves are **not** in this repo — they're excluded by `.gitignore` (too large, and usually licensed). This folder tracks what's installed and what still needs sourcing.

## Structure
```
EVE-NG-Images/
    Routers/      IOSv, CSR1000v, vMX
    Switches/     IOSvL2, vIOS-L2
    Firewalls/    pfSense, OPNsense, FortiGate VM, vSRX
```

Each subfolder is a drop-in slot: put the `.qcow2`/`.bin`/`.iso` in the matching one locally and it stays out of git automatically. Record it in the tables below so the repo still shows what the lab can actually run.

**How to actually import one** — the folder-prefix and disk-filename rules EVE-NG enforces, the `fixpermissions` step, and the differences between QEMU, IOL, and Dynamips images — is in [`../../HomeLab/EVE-NG-SETUP.md`](../../HomeLab/EVE-NG-SETUP.md). This file only tracks *which* images exist.

## Status legend
`installed` — imported into EVE-NG and boots · `sourced` — file downloaded, not imported yet · `needed` — a lab depends on it, don't have it · `optional` — nice to have, nothing blocked on it

## Router images
| Image | Version | Status | Needed for | Notes |
|---|---|---|---|---|
| Cisco IOSv | | needed | VLAN/inter-VLAN routing, ACLs, and DHCP/DNS labs (planned, not yet built) plus [CCNA-304](../Cisco/CCNA-304-single-area-ospfv2/README.md) (OSPF routing) | The workhorse image — most CCNA labs need it. ~512MB–1GB RAM per running node |
| Cisco CSR1000v | | optional | — | Heavier than IOSv; only worth it for features IOSv lacks |
| Juniper vMX | | optional | JNCIA routing labs (not yet created) | Alternative to vSRX for Junos practice; heavier |

## Switch images
| Image | Version | Status | Needed for | Notes |
|---|---|---|---|---|
| Cisco IOSvL2 | | needed | VLAN/inter-VLAN routing and ACLs labs (planned, not yet built) | Layer 2 switching, VLANs, trunking |
| Cisco vIOS-L2 | | optional | — | Older alternative to IOSvL2; same role, pick one |

## Firewall images
| Image | Version | Status | Needed for | Notes |
|---|---|---|---|---|
| Juniper vSRX | | needed | [Lab 05](../Juniper/01-junos-fundamentals/) | Blocks the whole JNCIA-Junos track (private, not yet published). Requires a Juniper account |
| pfSense | | optional | — | Lets me test firewall rule changes against a virtual copy before touching the real Protectli Vault ([`../PFSENSE-SETUP.md`](../../HomeLab/PFSENSE-SETUP.md)) |
| OPNsense | | optional | — | pfSense fork; comparison only |
| FortiGate VM | | optional | — | Third-vendor firewall exposure |

Fill in the version column as images are imported, flip status to `installed`, and add rows for anything else that goes in.

## RAM budgeting

EVE-NG runs on the M910q's 32GB specifically so several nodes can run at once — but each node still costs real memory, roughly 512MB–1GB for IOSv/IOSvL2 and more for vSRX/CSR1000v. A six-node CCNA topology is comfortable; a topology mixing Juniper and Cisco images is worth checking against the per-service budget in [`../SERVICES.md`](../../HomeLab/SERVICES.md) before it'll boot.

## Sourcing note

These images are Cisco and Juniper intellectual property. Cisco's come through a CML/VIRL subscription or Cisco Learning Network entitlements; Juniper's through a Juniper account. Whichever route, the files stay local — never in this repo, never in a public one.
