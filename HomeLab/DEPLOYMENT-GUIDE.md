# HomeLab Deployment Guide

This is the start-to-finish build order I used for my lab: 3x Lenovo Tiny nodes (1x M910q, 2x M700), a pfSense firewall, and a managed switch, now also running a highly-available DNS/DHCP stack, Home Assistant, and Docker. Follow it in order; each stage assumes the previous one is working before you move on. Swap in your own hardware and IPs as you go.

## Minimum hardware requirements
What this build actually needs, and what I run:

| Component | Minimum | What I run |
|---|---|---|
| Firewall | Any x86 box with 2+ NICs (or 1 NIC + managed switch for VLANs), 4GB RAM | pfSense on a dedicated appliance |
| Switch | Managed, supports 802.1Q VLAN tagging | TP-Link SG108E-class managed switch |
| Primary compute node (EVE-NG) | 16GB+ RAM, CPU with VT-x/VT-d, 4+ cores | Lenovo M910q, 32GB RAM |
| Secondary compute nodes | 16GB+ RAM recommended once running the DNS HA pair + Jellyfin/Home Assistant/Docker, CPU with VT-x/VT-d | 2x Lenovo M700, 16GB RAM each, Intel Core i3-6100T (2C/4T, Quick Sync-capable iGPU) |
| Storage | At least one dedicated data drive separate from any node's OS drive | 1TB (M910q, EVE-NG only) + 1 data drive on an M700 (media/NAS) |
| Backup target | External NAS with an NFS or CIFS export, sized to retention needs | See `BACKUP-SETUP.md` |

See `SERVICES.md` for the per-service RAM/vCPU breakdown and media disk sizing guidance.

## 0. Downloads you'll want on hand before you start
Grab these first so you're not hunting for installers mid-build:

- **Proxmox VE ISO**: https://www.proxmox.com/en/downloads
- **pfSense CE installer**: https://www.pfsense.org/download/
- **EVE-NG Community Edition**: https://www.eve-ng.net/index.php/download/ (free account required)
- **Tailscale**: https://tailscale.com/download
- **Jellyfin**: https://jellyfin.org/downloads/ (or use the Proxmox Community Scripts LXC below)
- **OpenMediaVault**: https://www.openmediavault.org/ (or plain Samba, see step 6)
- **Proxmox VE Community Scripts** (one-line LXC installers for Jellyfin, OMV, Pi-hole, Docker, etc.): https://community-scripts.github.io/ProxmoxVE/
- **Pi-hole**: https://pi-hole.net/ (v6.x; installed inside each DNS LXC, see `DNS-HA-SETUP.md`)
- **Unbound**: `apt install unbound` (Debian package, no separate download)
- **Keepalived**: `apt install keepalived` (Debian package, no separate download)
- **Nebula Sync**: https://github.com/lovelaze/nebula-sync (Docker image or standalone binary)
- **Home Assistant OS image** (for the VM route, not Supervised, see `HOME-ASSISTANT-SETUP.md`): https://www.home-assistant.io/installation/alternative
- **Docker Engine**: https://docs.docker.com/engine/install/debian/
- **Rufus** (flash installer ISOs to USB, Windows): https://rufus.ie/, or **balenaEtcher** (cross-platform): https://etcher.balena.io/

## 1. BIOS setup: do this on all 3 nodes before installing anything
Lenovo Tiny BIOS (F1 at boot):

- Enable **Intel VT-x** and **VT-d** (virtualization + IOMMU), required for Proxmox, and required for EVE-NG's nested virtualization.
- Set **AC Recovery / Power On after power loss = ON**. Without this, a power blip means driving over to physically press the power button on 3 headless boxes.
- Confirm boot order has the USB installer first (temporarily) for the install step.

This is the single most common "why won't my VM start" cause on Tiny-form-factor homelabs, so don't skip it.

## 2. Network first: pfSense + switch VLANs
Do this before touching Proxmox. The nodes need VLAN 20 to exist before they have anywhere to plug into.

Follow **`PFSENSE-SETUP.md`** in this folder for the full walkthrough (VLAN creation, interface assignment, DHCP, firewall rules, switch trunk/access ports). Once that's done and tested per its Step 6 checklist, come back here.

## 3. Install Proxmox VE on each node
1. Flash the Proxmox ISO to USB (Rufus/Etcher, see step 0).
2. Install on the M910q first, then each M700.
3. During install, set a static IP on VLAN 10 (MGMT) for each node's web UI; see `ADDRESSING.md` for the addresses to use. Don't rely on DHCP for these; MGMT is static-only by design.
4. First-login housekeeping on **every** node (via `Datacenter > Node > Updates > Repositories` or CLI):
   - Disable the `enterprise` repo (needs a paid subscription) and add the `pve-no-subscription` repo, or you'll get update errors/nag screens.
   - Run a full `apt update && apt full-upgrade` before doing anything else.
5. Confirm NTP/time sync is working on all 3 (`timedatectl`). Proxmox clustering (corosync) is latency- and clock-sensitive.

## 4. Form the cluster
1. On the M910q (primary): `Datacenter > Cluster > Create Cluster`.
2. On each M700: `Datacenter > Cluster > Join Cluster`, using the join info from the M910q.
3. Confirm all 3 nodes show green/online in the Datacenter view. A 3-node cluster keeps quorum as long as 2 of 3 are up, which is why 3 (not 2) is the right minimum.

## 5. Storage
Set up per the drive plan in `SERVICES.md`:
- M910q: 1TB drive dedicated to EVE-NG VM disks only.
- Node 2 (M700): add its 2.5" data drive, format/mount it, present it to the NAS/Jellyfin LXCs as a bind-mount or passthrough disk. Don't just use the small OS drive for media storage.
- Node 3 (M700): no data drive needed.

## 6. Deploy services, in this order
1. **EVE-NG** on the M910q: create the VM, enable nested virtualization (CPU type = `host`, not the default `kvm64`) so EVE-NG's virtual routers/switches can run inside it. This is the step people miss and then wonder why nodes inside EVE-NG won't boot. Full walkthrough — VM sizing, the multi-reboot install, first-boot config, and the strict folder/filename rules for importing device images — in [`EVE-NG-SETUP.md`](./EVE-NG-SETUP.md).
2. **NAS share (Node 2)**: Samba LXC or OpenMediaVault, pointed at the data drive from step 5. Set up shares before Jellyfin so it has somewhere to read from.
3. **Jellyfin (Node 2)**: LXC via the Community Scripts one-liner (see step 0) is the fastest path; point its library at the NAS share. Once it's running, follow `HARDWARE-ACCELERATION.md` to pass the iGPU through for Quick Sync transcoding rather than leaving it on software transcoding.
4. **Tailscale (Node 3)**: install as a subnet router, not per-VM:
   ```
   tailscale up --advertise-routes=10.10.20.0/24 --accept-routes
   ```
   Then approve the advertised route in the Tailscale admin console. This gives remote access to the whole VLAN 20 segment through one node instead of installing Tailscale everywhere.
5. **DNS/DHCP HA pair (Node 2 + Node 3)**: two Pi-hole + Unbound LXCs (one per node), Keepalived on both for the floating VIP, Nebula Sync to keep their config identical. Deploy this before Home Assistant and Docker so both nodes already have working local DNS resolution for whatever hostnames those services need. Full steps, including the pfSense-side DHCP/firewall changes, in `DNS-HA-SETUP.md`.
6. **Home Assistant OS (Node 3)**: VM, not the deprecated Supervised install method. Full steps in `HOME-ASSISTANT-SETUP.md`.
7. **Docker host (Node 2)**: unprivileged LXC by default, VM if a workload needs it. Full steps and the decision criteria in `DOCKER-SETUP.md`.

8. **pfSense services (back on the firewall)**: with the cluster's DNS VIP live and `PFSENSE-SETUP.md` Step 7 done, go back to the firewall and bring up its own services — DNS Resolver scoped to MGMT/PVE, an internal NTP server for the whole lab, pfBlockerNG, WireGuard, and Suricata/ntopng as a deliberate load test of the Protectli Vault. Full walkthrough and a benchmarking method in `PFSENSE-SERVICES.md`. Do this **after** the DNS HA pair, not before: pfSense's resolver is scoped specifically around the Pi-hole VIP already existing.
9. **Repoint the Proxmox nodes at the internal NTP server**, one node at a time, once it's verified. Worth doing explicitly — corosync is clock-sensitive (see step 3.5), and a sub-millisecond LAN time source beats three nodes independently reaching the public pool over WAN. Steps in `PFSENSE-SERVICES.md`.

## 7. Backups (don't skip this once things are running)
- Point Proxmox at an external NAS over NFS or CIFS (`Datacenter > Storage > Add`) rather than standing up Proxmox Backup Server; simpler for a lab this size, and it keeps backup storage physically off the nodes it's protecting. Full steps, retention settings, and the NFS root-squash gotcha in `BACKUP-SETUP.md`.
- Schedule backups for every VM/LXC that would hurt to lose: EVE-NG, NAS/Jellyfin, both Pi-hole instances (or at minimum, keep Nebula Sync as the recovery path for those since they replicate each other), Home Assistant, and the Docker host.

## 8. Additional nodes: scaling the cluster
Proxmox clusters aren't limited to 3 nodes. Adding one is straightforward: physically wire it into the VLAN 20 switch port, run through BIOS setup (Section 1) and the Proxmox install (Section 3) on it, then `Datacenter > Cluster > Join Cluster` exactly as I did for the two M700s in Section 4. The only real requirements are that the new node runs a Proxmox VE version compatible with the rest of the cluster and has VT-x/VT-d enabled.

Extra nodes are also where the lab becomes a place to practice new skills: each one is a low-risk way to try something without touching what's already stable. A few example nodes and what they'd realistically need:

| Example node purpose | Min RAM | Min storage | Why it's a good next step |
|---|---|---|---|
| Second EVE-NG node | 16–32GB | 250GB+ | For running bigger topologies in parallel with the primary M910q instead of queueing labs one at a time |
| Kubernetes sandbox (k3s) | 8–16GB | 50GB+ | Docker's already covered on Node 2 (`DOCKER-SETUP.md`); this is the next step up if container orchestration is worth practicing without risking anything the rest of the lab depends on |
| Dedicated Proxmox Backup Server | 4–8GB | Sized to retention needs | Upgrade path over the current NFS/CIFS `vzdump` setup in `BACKUP-SETUP.md`: adds deduplication and incremental backups if the external NAS's raw storage use starts to matter |
| Raspberry Pi 5 + SATA HAT storage expansion | 4–8GB (Pi) | As many drives as the HAT supports (4–5 bays) | Not a cluster node, since Proxmox VE doesn't run on ARM. It's an independent NAS box the cluster mounts over NFS/SMB, and a candidate backup target for `BACKUP-SETUP.md`. See "Storage expansion" in `SERVICES.md` for the full writeup |
| Third M700/Tiny node | 8–16GB | Matches whatever it's dedicated to | Once Node 2 and Node 3 are both carrying multiple services each, a 4th node is the cleanest way to give the DNS HA pair, Home Assistant, or Docker more dedicated headroom instead of packing more onto two boxes |

Whatever gets added, I document it the same way as the rest of this lab: an entry in `SERVICES.md`'s node table, an IP from `ADDRESSING.md`'s VLAN 20 range, and a note in this guide if the deployment order matters.

## 9. Using the lab once it's built
The infrastructure above is the platform, not the point. Once EVE-NG is up on Node 1, the practice work lives in two folders:

- **`Labs/`** — one folder per hands-on lab, each with drop-in slots for the Packet Tracer `.pkt` or EVE-NG `.unl` topology, the instructions PDF, device configs, and verification output. Copy `Labs/_TEMPLATE/` to start a new one. See [`Labs/README.md`](../Labs/README.md).
- **`Certifications/`** — CCNA, JNCIA-Junos, and Cisco AITECH tracks, each mapping exam objectives to the labs and to the infrastructure docs in this repo that already cover them (private, not yet published).

Which device images EVE-NG needs for a given lab (and which are still missing) is tracked in [`EVE-NG-Images/README.md`](../Labs/EVE-NG-Images/README.md). Note that Lab 05 and the entire JNCIA track are blocked until a Juniper vSRX image is imported.

## 10. Ongoing
- I update this guide and `SERVICES.md` as the build diverges from the plan; both change as it does.
- I review `ADDRESSING.md` and `PFSENSE-SETUP.md`'s security note before adding new services. Nothing with credentials, tokens, or config exports belongs in this repo.
