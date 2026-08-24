# HomeLab

## Overview
My home lab runs on real hardware: a pfSense firewall, a VLAN-trunked managed switch, and a 3-node Proxmox cluster. I use it to self-host services and to practice the infrastructure from my BIT program — VLANs, firewall rules, and virtualization — on real gear instead of a simulator.

## Documentation

### Infrastructure
- [**Deployment Guide**](./DEPLOYMENT-GUIDE.md): start here, full build order, hardware requirements, and download links
- [Network Diagram](./network-diagram.md): logical topology (Mermaid)
- [Addressing Scheme](./ADDRESSING.md): VLANs, subnets, static assignments
- [pfSense VLAN Setup Guide](./PFSENSE-SETUP.md): step-by-step firewall/switch configuration
- [pfSense Services Setup Guide](./PFSENSE-SERVICES.md): DNS Resolver, NTP, pfBlockerNG, SNMP, WireGuard, Suricata, ntopng, plus a benchmarking method for the Protectli Vault
- [Services](./SERVICES.md): what's running on the Proxmox cluster, node/storage layout, and minimum hardware requirements
- [Home Assistant Setup](./HOME-ASSISTANT-SETUP.md): Home Assistant OS on a Proxmox VM, plus add-ons
- [DNS/DHCP High Availability Setup](./DNS-HA-SETUP.md): Pi-hole + Unbound + Keepalived + Nebula Sync, so DNS survives a node failure
- [Backup Setup](./BACKUP-SETUP.md): Proxmox backups to an external NAS over NFS/CIFS
- [Hardware Acceleration](./HARDWARE-ACCELERATION.md): Intel Quick Sync passthrough for Jellyfin transcoding
- [Docker Setup](./DOCKER-SETUP.md): running Docker as an LXC or a VM, and how to decide which
- [EVE-NG Setup](./EVE-NG-SETUP.md): EVE-NG CE as a Proxmox VM, nested virtualization, and the folder/filename rules for importing QEMU, IOL, and Dynamips images

### Labs and study
- [**Labs**](../Labs/README.md): every hands-on lab, one folder each, with drop-in slots for Packet Tracer `.pkt` files, EVE-NG `.unl` topologies, instruction PDFs, device configs, and verification output
- **Certifications** (private, not yet published): CCNA, JNCIA-Junos, and Cisco AITECH tracks, with exam objectives mapped to the labs and infrastructure docs that cover them
- [EVE-NG Device Images](../Labs/EVE-NG-Images/README.md): which router/switch/firewall images are installed, and which labs are blocked waiting on one

## Hardware
- Firewall: pfSense CE on a Protectli Vault FW4B (Intel Celeron J3160, 4C/4T, 8GB max RAM, 4x Intel GbE, fanless)
- Switch: managed, 802.1Q trunked
- Compute: 3-node Proxmox cluster: 1x Lenovo M910q (32GB RAM, runs EVE-NG), 2x Lenovo M700 (16GB RAM each, Intel Core i3-6100T) running media/storage, DNS/DHCP HA, Home Assistant, and Docker

## Written to be publishable
Every IP documented here is private RFC1918 space (10.10.x.0/24), not reachable from the internet, and safe to publish. Secrets, credentials, VPN configs, and firewall config backups never belong in these docs regardless — see the security note at the bottom of `PFSENSE-SETUP.md` and the repo's `.gitignore` for what's excluded.

## Repo layout
`HomeLab/` is one project folder in the portfolio repo root, alongside `Labs/` (hands-on
practice labs) and, privately, `Certifications/` (exam tracks). It doesn't nest those —
they're siblings, not children:
```
HomeLab/
    *.md                 infrastructure guides for the running lab (this folder's own files)
Labs/
    EVE-NG-Images/        which node images are installed; the files themselves stay local
```

## Notes / Lessons Learned
A running log of what worked and what I'd do differently next time. Per-lab versions of this live in each lab's "What broke" section; this table is for the physical build.

| Date | What happened | What I'd do differently |
|---|---|---|
| | | |
