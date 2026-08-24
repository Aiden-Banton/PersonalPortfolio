# Proxmox Cluster: Services

My 3-node Proxmox cluster runs on VLAN 20 (see `ADDRESSING.md`). One heavy node handles EVE-NG, two lightweight nodes handle everything else, now including a DNS/DHCP high-availability pair, Home Assistant, and a Docker host. Full build order and links are in `DEPLOYMENT-GUIDE.md`.

## Cluster nodes
| Node | Model | CPU | RAM installed | RAM max (stock) | OS drive | Data drive | IP (VLAN 20) | Role |
|---|---|---|---|---|---|---|---|---|
| Proxmox Node 1 | Lenovo M910q | (not swapped, stock) | 32 GB | 32 GB (2x16GB DDR4-2133 SODIMM) | M.2 SATA, 128–256GB | 1TB (existing, EVE-NG VM disks only) | 10.10.20.11 | Primary, runs EVE-NG only (heavy resource demand, kept isolated from everything below) |
| Proxmox Node 2 | Lenovo M700 | Intel Core i3-6100T (2C/4T, 3.2GHz) | 16 GB | 32 GB (2x16GB DDR4-2133 SODIMM) | M.2 SATA, 128–256GB | 2.5" SATA SSD/HDD, sized to media library | 10.10.20.12 | Jellyfin + NAS/file share + Docker host + DNS/DHCP HA member A (MASTER) |
| Proxmox Node 3 | Lenovo M700 | Intel Core i3-6100T (2C/4T, 3.2GHz) | 16 GB | 32 GB (2x16GB DDR4-2133 SODIMM) | M.2 SATA, 128–256GB | none needed | 10.10.20.13 | Tailscale subnet router + Home Assistant + DNS/DHCP HA member B (BACKUP) |

Both M700s were upgraded from 8GB to 16GB (still within the documented 32GB/2-slot max) to make room for the services below. The i3-6100T's HD Graphics 530 iGPU is also what makes the Jellyfin hardware-acceleration guide (`HARDWARE-ACCELERATION.md`) possible: it supports Intel Quick Sync (QSV) for H.264/HEVC encode and decode.

**Storage standard:** each node's M.2 SATA slot is reserved for the Proxmox OS install only; I never mix OS and data on the same drive. Node 2's 2.5" bay holds the only bulk data drive in the cluster; the M910q's 1TB stays dedicated to EVE-NG so file-serving I/O never competes with lab traffic. Node 3 needs no data drive; Home Assistant and the DNS containers are all config-sized, not media-sized.

- Storage backend: local-lvm (default). Ceph isn't worth the RAM overhead on 16GB M700s for a cluster this size.
- Backup target: an external NAS over NFS/CIFS, not Proxmox Backup Server. See `BACKUP-SETUP.md` for the full setup and reasoning.

## VMs / LXCs
| Service | Purpose | VM/LXC | Node | IP | Status |
|---|---|---|---|---|---|
| EVE-NG | Network emulation (CCNA labs, topology testing) | VM | M910q | 10.10.20.50 | running |
| Tailscale subnet router | Remote access into VLAN 20 without installing Tailscale on every host | LXC | M700 (Node 3) | | planned |
| Jellyfin | Media server, hardware-accelerated transcoding via QSV | LXC | M700 (Node 2) | | planned |
| NAS / file share (Samba or OpenMediaVault) | Central storage | LXC | M700 (Node 2) | | planned |
| Pi-hole + Unbound, instance A | DNS filtering + recursive resolver, HA member (MASTER) | LXC | M700 (Node 2) | 10.10.20.21 | planned |
| Pi-hole + Unbound, instance B | DNS filtering + recursive resolver, HA member (BACKUP) | LXC | M700 (Node 3) | 10.10.20.22 | planned |
| Keepalived (runs alongside both Pi-hole instances) | VRRP floating VIP so clients always query one working DNS server | N/A (co-located with Pi-hole LXCs) | M700 (Node 2 & 3) | 10.10.20.20 (VIP) | planned |
| Nebula Sync | Keeps both Pi-hole instances' blocklists/config identical | LXC | M700 (Node 3) | | planned |
| Home Assistant OS | Home automation hub + add-ons | VM | M700 (Node 3) | 10.10.20.30 | planned |
| Docker host | General-purpose container host for self-hosted apps | LXC (VM if resources get tight; see `DOCKER-SETUP.md`) | M700 (Node 2) | 10.10.20.40 | planned |

See `DNS-HA-SETUP.md`, `HOME-ASSISTANT-SETUP.md`, and `DOCKER-SETUP.md` for why each service landed on the node it did and the full build steps.

## Minimum hardware requirements (per service)
Rough floors, not hard limits, enough to run smoothly at small-to-medium scale on this hardware:

| Service | Min RAM | Min vCPU | Notes |
|---|---|---|---|
| Proxmox VE (host overhead, per node) | ~2 GB | 1 | Reserved before any VM/LXC allocation |
| EVE-NG | 8–16 GB+ | 4+ | Scales with topology size: each running router/switch node typically costs 128MB–1GB+ depending on image. The M910q's 32GB gives headroom for multiple concurrent labs |
| Tailscale subnet router | 512 MB | 1 | Negligible footprint, no meaningful storage needs |
| Jellyfin | 2 GB | 2 | Comfortable with hardware transcoding (Intel Quick Sync via the M700's HD Graphics 530 iGPU, see `HARDWARE-ACCELERATION.md`); software transcoding is far more RAM/CPU hungry and I avoid relying on it |
| NAS / Samba / OpenMediaVault | 1–2 GB | 1–2 | RAM mostly benefits file-cache performance, not a hard requirement |
| Pi-hole (per instance) | 512 MB–1 GB | 1 | Scales with blocklist size and query volume; negligible at homelab scale |
| Unbound (per instance) | 512 MB | 1 | Recursive resolver cache benefits from more RAM but doesn't require it |
| Keepalived (per instance) | ~64 MB | <1 | VRRP daemon, essentially free to run |
| Nebula Sync | ~128 MB | <1 | Small Go binary, runs on a cron schedule, no meaningful idle footprint |
| Home Assistant OS | 2–4 GB | 2 | Baseline HAOS is light; each add-on (Node-RED, ESPHome, Zigbee2MQTT, InfluxDB+Grafana, etc.) adds its own overhead, budget more if running several at once |
| Docker host (LXC) | 1–2 GB + per-container needs | 1–2 | Base LXC overhead is small; real usage depends entirely on what's deployed inside it |
| Docker host (VM, alternative) | 2–4 GB + per-container needs | 2 | Full kernel isolation costs more baseline RAM/CPU than the LXC route; see `DOCKER-SETUP.md` for the tradeoff |

**Node 2 budget (Jellyfin + NAS + Docker host + Pi-hole/Unbound/Keepalived instance A):** roughly 2 (host) + 2 (Jellyfin) + 1.5 (NAS) + 2 (Docker LXC baseline) + 1 (Pi-hole) + 0.5 (Unbound) + 0.1 (Keepalived) ≈ 9GB of the 16GB installed, leaving headroom for Docker workloads and transcoding bursts. QSV hardware transcoding matters here specifically because it keeps Jellyfin's *CPU* usage low, which is what leaves the i3's 2 cores free for whatever's running in Docker.

**Node 3 budget (Tailscale + Home Assistant + Pi-hole/Unbound/Keepalived instance B + Nebula Sync):** roughly 2 (host) + 0.5 (Tailscale) + 4 (Home Assistant OS VM) + 1 (Pi-hole) + 0.5 (Unbound) + 0.1 (Keepalived) + 0.1 (Nebula Sync) ≈ 8GB of the 16GB installed, before counting Home Assistant add-ons.

Both nodes have real headroom at 16GB, though less than the raw number suggests now that each M700 carries five services rather than two. If either node feels tight in practice, the fixes I'd try, in order: move Nebula Sync or the Docker host to whichever node has more headroom at the time, then a RAM upgrade (still under the documented 32GB max), then a 4th node if the lab keeps growing.

## Media disk sizing
Rough sizing guide for the data drive on Node 2, based on typical file sizes:

| Content | Approx. size | Library of 50 | Library of 200 |
|---|---|---|---|
| 1080p movie (H.264/H.265) | 4–8 GB | 200–400 GB | 800 GB–1.6 TB |
| 4K movie (remux/high bitrate) | 20–50 GB | 1–2.5 TB | 4–10 TB |
| TV series (1080p, per season) | 5–15 GB | n/a | n/a |

Starting point: a 1–2TB 2.5" drive covers a modest 1080p library comfortably and leaves room to grow. If 4K content is part of the plan from the start, size up to 4TB+; the 2.5" form factor tops out around there for HDDs, and SSDs at that capacity get expensive fast. Since the M700 only has one data bay, there's no in-place expansion once it's full, so plan capacity for a year or two of growth rather than exactly what's needed today.

## Storage expansion (beyond the cluster)
If the Node 2 data drive fills up, or I want more storage without adding another x86 Tiny to the cluster, the option I'd reach for is a **Raspberry Pi 5 + Radxa Penta SATA HAT**, running as an independent NAS box rather than a Proxmox node.

**Important distinction:** Proxmox VE is x86_64-only, so a Pi 5 (ARM) can't join this cluster as a node. It sits on the network as its own storage appliance, and any Proxmox VM/LXC (Jellyfin, the existing NAS share, etc.) mounts it over NFS or SMB, the same way you'd treat a commercial NAS. This keeps expansion simple: add drives to the Pi, not to the cluster's own storage config.

| Component | Spec | Notes |
|---|---|---|
| Board | Raspberry Pi 5 (4GB or 8GB) | 8GB gives more headroom if running ZFS/Samba/NFS simultaneously |
| Expansion | Radxa Penta SATA HAT | Up to 4 SATA drives + 1 eSATA port, connects via the Pi 5's PCIe FFC connector through a JMB585 SATA-to-PCIe bridge |
| Bandwidth | Shared PCIe Gen2 x1 (~400MB/s aggregate) | Fine for a media/backup NAS, not meant to compete with local NVMe/SATA on the actual cluster nodes. Read-heavy workloads are the sweet spot |
| Power | 12V barrel or ATX Molex on the HAT; it back-feeds 5V to the Pi | No separate Pi power supply needed |
| OS | Raspberry Pi OS / Debian (arm64) with Samba or NFS, or OpenMediaVault (has arm64 builds) | Plain Samba/NFS is the simplest, most reliable option on a Pi |
| Typical power draw | ~6–16W (idle to load) | Cheap to run 24/7 compared to adding another x86 node just for storage |

How it plugs in:
1. Assign it a static IP in VLAN 20 per `ADDRESSING.md` (treat it like any other fixed device, not part of the Proxmox cluster's node range).
2. Set up the drives (RAID/pool of your choice: mdadm, ZFS, or just independent mounts depending on how much redundancy matters) and export shares over NFS or SMB.
3. Mount that share from whichever LXC/VM needs the extra space, most naturally the Jellyfin or NAS LXC on Node 2, so the media library can span both the local 2.5" drive and the Pi's pool without restructuring anything on the cluster side.
4. Add it to the pfSense rules the same way as the EVE-NG VM if it needs to be reachable from TRUSTED/MEDIA. A dedicated alias makes this a one-line change later if the IP moves.

This is a good one to log in `DEPLOYMENT-GUIDE.md`'s Additional Nodes section too, since it's the clearest path to "I'm out of storage" that doesn't involve buying another Tiny PC.

## DNS/DHCP high availability
pfSense stays the DHCP server for TRUSTED/MEDIA (no change there), but the DNS server it hands out is now a **Keepalived VIP** (`10.10.20.20`) fronting two independent Pi-hole + Unbound instances, one per M700. If either node goes down for a reboot, an update, or a hardware problem, the other keeps answering DNS queries within the VRRP failover window (low single-digit seconds), no manual intervention. Nebula Sync keeps both Pi-hole instances' blocklists and settings identical so they behave as one logical service, not two configs that drift apart.

Full build steps, the Keepalived config for both nodes, and the pfSense-side alias/rule/DHCP changes needed to expose the VIP to TRUSTED/MEDIA are in `DNS-HA-SETUP.md`.

## Home Assistant
Runs as **Home Assistant OS in a VM** on Node 3, not the "Supervised" install method. Supervised was deprecated by the Home Assistant project starting with the 2025.6 release and unsupported as of 2025.12; HAOS in a VM is their official replacement path and preserves full add-on support (Node-RED, ESPHome, Zigbee2MQTT, etc.), so it's a straight swap for what Supervised would have given me anyway, just on a currently-maintained install method. See `HOME-ASSISTANT-SETUP.md`.

## Docker host
A general-purpose Docker host on Node 2 for self-hosted apps that don't need their own dedicated VM. Deployed as an **unprivileged LXC** by default, since the i3-6100T is only 2 cores/4 threads and Node 2 is already carrying Jellyfin, the NAS share, and half the DNS HA pair; an LXC's near-zero baseline overhead matters more here than it would on a beefier node. If a workload needs a container image or kernel feature that doesn't play well with LXC nesting, the fallback is a small dedicated VM instead. Full reasoning and setup for both paths in `DOCKER-SETUP.md`.

## Monitoring
Nothing in the cluster is monitored yet; it's worth adding once the core services above are stable. Two directions to consider:

**PRTG (free tier)**: Paessler's PRTG is a turnkey commercial option with a permanent free edition capped at 100 sensors (a "sensor" is roughly one monitored metric, e.g. one interface's traffic, one disk's free space, one ping check). For a lab this size, 100 sensors comfortably covers all 3 Proxmox nodes, the pfSense box, the switch, and the VMs/LXCs above with room to spare. Upside: full GUI, auto-discovery, and alerting out of the box with almost no setup. Downside: it's closed-source and the free tier is a hard wall if the lab grows past it.

**Open-source equivalents**, no sensor cap, self-hosted, better fit long-term for a lab that's meant to keep growing:

| Tool | Best for | Notes |
|---|---|---|
| Uptime Kuma | Simple up/down checks + status page | Lightest to run, easiest to set up, least detail. Small enough to fit wherever has headroom at the time, doesn't need a dedicated node |
| Netdata | Per-node real-time resource metrics | Very low overhead per agent, minimal config, best for "what is this node doing right now" rather than long-term trends |
| LibreNMS | SNMP-based network monitoring (switch, pfSense, interfaces) | Purpose-built for network gear, more setup than Uptime Kuma but still lighter than Zabbix |
| Checkmk (raw edition) | Balanced infra + network monitoring | Auto-discovery keeps setup fast, low CPU footprint, single pane of glass for hosts + services |
| Zabbix | Full-featured, highly customizable monitoring | Most powerful, but the heaviest to configure and run; more than this lab needs at 3 nodes |

**Already on the firewall**: SNMP is enabled on pfSense (bound to MGMT only) specifically so LibreNMS or Checkmk can poll it whenever one of them goes in, and ntopng is documented there as the per-host bandwidth-accounting option. See `PFSENSE-SERVICES.md`. Note that ntopng on the FW4B competes for RAM with Suricata, so it's an either/or on that hardware rather than an addition.

**Recommendation for this cluster**: given the M700s are still RAM-constrained relative to what they're now running (16GB split across five services each), the lightest-touch path is Uptime Kuma as an LXC for up/down + status page, paired with Netdata agents on all 3 nodes for resource metrics. That covers the essentials without competing meaningfully for RAM anywhere in the cluster. LibreNMS or Checkmk are the natural upgrade if deeper switch/pfSense-level (SNMP) visibility becomes worth the extra overhead later. PRTG's free tier is a reasonable alternative if a polished GUI with zero configuration matters more than staying fully open-source. Worth keeping an eye on: with Keepalived already doing VRRP health checks on the DNS pair, Uptime Kuma's monitors are redundant for that one service but still worth having for everything else (Jellyfin, Home Assistant, Docker host, EVE-NG).

## Notes
- Tailscale is deployed **once**, as a subnet router advertising the VLAN 20 range, not installed per-service. One `tailscale up --advertise-routes=10.10.20.0/24` on Node 3 gives remote access to the whole segment; simpler to manage and secure than per-VM installs. It's paired with a WireGuard tunnel on pfSense itself (`PFSENSE-SERVICES.md`) as the break-glass path: Tailscale is easier day to day, but it depends on Node 3 and the Proxmox cluster being up, which is exactly what's not true when remote access matters most.
- Jellyfin and the NAS share live on the *same* node (Node 2) so Jellyfin reads media over local disk, not a network hop, for better transcoding performance on RAM-constrained hardware.
- Skipping TrueNAS/ZFS on Node 2: its RAM budget is better spent on Jellyfin transcoding than ZFS's ARC cache. Samba or OpenMediaVault is the right fit here.
- The DNS/DHCP HA pair is deliberately split one instance per M700 (not both on one node): the entire point of Keepalived + Nebula Sync is that losing either node still leaves working DNS. Stacking both Pi-hole instances on the same box would defeat the purpose.
- Home Assistant and the Docker host are split across Node 2 and Node 3 rather than both landing on one node, so no single M700 is carrying two heavy new services on top of its existing load.
- M910q stays EVE-NG-only; none of the six services added here run on it, even though it has the most spare RAM, to keep EVE-NG's resource-heavy lab runs from competing with always-on infrastructure.
- Update "Status" to `running` once deployed, and fill in real IPs as assigned per `ADDRESSING.md`.
- Keep this list free of credentials, tokens, or connection strings; link to where credentials are stored (password manager, vault), don't paste them here. These docs are written to be publishable.

Sources on M700 hardware specs: [Lenovo ThinkCentre M700 Tiny: Specs and upgrade options](https://www.hardware-corner.net/desktop-models/Lenovo-ThinkCentre-M700-Tiny/), [ThinkCentre M700 Tiny Platform Specifications (Lenovo)](https://psref.lenovo.com/syspool/Sys/PDF/ThinkCentre/ThinkCentre_M700_Tiny/ThinkCentre_M700_Tiny_Spec.PDF). Home Assistant Supervised deprecation: [Deprecating Core and Supervised installation methods, and 32-bit systems](https://www.home-assistant.io/blog/2025/05/22/deprecating-core-and-supervised-installation-methods-and-32-bit-systems/). Nebula Sync: [lovelaze/nebula-sync](https://github.com/lovelaze/nebula-sync).
