# HomeLab Addressing Scheme

The VLAN/subnet layout I run, enforced on a pfSense firewall and an 802.1Q trunked managed switch.

| VLAN ID | Name | Subnet | Gateway | Purpose |
|---|---|---|---|---|
| 10 | MGMT | 10.10.10.0/24 | 10.10.10.1 | Jump network, full access to everything, static-only |
| 20 | PVE / CLUSTER | 10.10.20.0/24 | 10.10.20.1 | Proxmox cluster, isolated from Trusted/Media |
| 30 | TRUSTED | 10.10.30.0/24 | 10.10.30.1 | General trusted clients (laptop, etc.) |
| 40 | MEDIA | 10.10.40.0/24 | 10.10.40.1 | Internet + EVE-NG only, no device management |

## Static assignments
| Device | VLAN | IP | Notes |
|---|---|---|---|
| Mgmt Access PC | 10 | 10.10.10.10 | static, jump point |
| Proxmox Node 1 | 20 | 10.10.20.11 | static |
| Proxmox Node 2 | 20 | 10.10.20.12 | static |
| Proxmox Node 3 | 20 | 10.10.20.13 | static |
| EVE-NG VM | 20 | 10.10.20.50 | static (exposed to other VLANs) |
| DNS VIP (Keepalived, Pi-hole HA pair) | 20 | 10.10.20.20 | static, floating; the address every other VLAN actually queries for DNS, exposed to other VLANs |
| Pi-hole + Unbound, instance A (Node 2) | 20 | 10.10.20.21 | static, HA member (MASTER); not directly queried by clients, VIP only |
| Pi-hole + Unbound, instance B (Node 3) | 20 | 10.10.20.22 | static, HA member (BACKUP); not directly queried by clients, VIP only |
| Home Assistant OS VM (Node 3) | 20 | 10.10.20.30 | static, exposed to TRUSTED only |
| Docker host (Node 2) | 20 | 10.10.20.40 | static, per-app firewall rules added as needed, same pattern as EVE-NG |
| External NAS (backup target) | 20 or its own segment | see `BACKUP-SETUP.md` | Whatever address the NAS already has; treated as a fixed device, not a cluster node |

## DHCP pools
| VLAN | Pool | DNS server handed out | Notes |
|---|---|---|---|
| 10 (MGMT) | none | pfSense (default) | static-only by design |
| 20 (PVE) | none (default) | pfSense (default) | static-only; can add a pool later for flexibility |
| 30 (TRUSTED) | 10.10.30.100 – 10.10.30.200 | 10.10.20.20 (Pi-hole VIP) | changed from pfSense's own resolver once `DNS-HA-SETUP.md` is deployed |
| 40 (MEDIA) | 10.10.40.100 – 10.10.40.200 | 10.10.20.20 (Pi-hole VIP) | changed from pfSense's own resolver once `DNS-HA-SETUP.md` is deployed |

### Reserved (not yet deployed)
| Range | Purpose |
|---|---|
| 10.10.50.0/24 | WireGuard tunnel clients (pfSense side `10.10.50.1`), deliberately outside VLANs 10–40 so tunnel traffic is distinguishable in firewall rules and logs. See `PFSENSE-SERVICES.md` |

### DNS and NTP responsibilities
- **TRUSTED (30) and MEDIA (40)** use the Pi-hole VIP `10.10.20.20` for DNS (per the DHCP table above).
- **MGMT (10) and PVE (20)** use pfSense's own DNS Resolver at their gateway. This is intentional, not an oversight: the Proxmox nodes must not depend on DNS that runs as guests on themselves. If both Pi-hole LXCs are down, the nodes hosting them still need working name resolution to be recoverable.
- **Every VLAN** uses its own gateway (`10.10.x0.1`) as its NTP server, with pfSense syncing upstream to the public pool. Setup for both in `PFSENSE-SERVICES.md`.

WAN: DHCP from my ISP by default. Check your modem/ONT documentation if a static IP or PPPoE is required on your connection instead.

## Inter-VLAN access rules (summary; full detail in `PFSENSE-SETUP.md`)
- **MGMT (10)** → any (full access, jump network).
- **PVE (20)** → internet only (outbound updates). No inbound rule needed from MGMT; that's covered by the MGMT→any rule.
- **TRUSTED (30)** → EVE-NG VM (10.10.20.50), DNS VIP (10.10.20.20, port 53 tcp/udp), Home Assistant (10.10.20.30, port 8123) within VLAN 20, plus general internet access.
- **MEDIA (40)** → EVE-NG VM (10.10.20.50), DNS VIP (10.10.20.20, port 53 tcp/udp) within VLAN 20, plus general internet access. Not Home Assistant; nothing on MEDIA (TV, PS5) needs it.
- Everything not explicitly allowed is denied by pfSense's implicit deny-all on every interface.

I keep pfSense aliases (`EVE_NG_VM` = 10.10.20.50, `DNS_VIP` = 10.10.20.20, `HOME_ASSISTANT` = 10.10.20.30) so I only need to update one place if any of those IPs ever change, rather than editing every firewall rule that references them. The two Pi-hole instance IPs (`.21`/`.22`) intentionally don't get their own firewall rules or aliases; clients only ever talk to the VIP, and Keepalived decides which real instance answers.
