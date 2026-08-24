# HomeLab Network Diagram

The logical topology of my home lab, rendered with Mermaid (GitHub renders this automatically).

```mermaid
flowchart TB
    ISP["Internet / ISP Uplink"]

    subgraph FW["pfSense Firewall"]
        WAN["WAN - DHCP from ISP"]
        LAN["LAN Trunk - 802.1Q Tagged"]
        WAN --- LAN
    end

    ISP --> WAN

    SW["Managed Switch\nTrunk port to pfSense"]
    LAN ---|"Trunk: VLAN 10,20,30,40"| SW

    subgraph V10["VLAN 10 - MGMT - 10.10.10.0/24 - GW .1"]
        MGMTPC["Mgmt Access PC\n10.10.10.10 (static)\njump point"]
    end

    subgraph V20["VLAN 20 - PVE CLUSTER - 10.10.20.0/24 - GW .1"]
        PVE1["Proxmox Node 1 (M910q)\n10.10.20.11 - 32GB RAM\nEVE-NG only"]
        PVE2["Proxmox Node 2 (M700)\n10.10.20.12 - 16GB RAM, i3-6100T\nJellyfin + NAS + Docker + DNS HA-A"]
        PVE3["Proxmox Node 3 (M700)\n10.10.20.13 - 16GB RAM, i3-6100T\nTailscale + Home Assistant + DNS HA-B"]
        EVE["EVE-NG VM\n10.10.20.50\n(exposed to other VLANs)"]
        DNSVIP["DNS VIP (Keepalived)\n10.10.20.20\n(exposed to other VLANs)"]
        PIHOLEA["Pi-hole + Unbound A\n10.10.20.21 (on Node 2, MASTER)"]
        PIHOLEB["Pi-hole + Unbound B\n10.10.20.22 (on Node 3, BACKUP)"]
        NEBULA["Nebula Sync\n(on Node 3, syncs A -> B)"]
        HA["Home Assistant OS VM\n10.10.20.30 (on Node 3)\n(exposed to TRUSTED)"]
        DOCKER["Docker host\n10.10.20.40 (on Node 2)"]
    end

    subgraph V30["VLAN 30 - TRUSTED - 10.10.30.0/24 - GW .1"]
        LAPTOP["Laptop\nDHCP 10.10.30.100-200\nDNS: 10.10.20.20"]
    end

    subgraph V40["VLAN 40 - MEDIA - 10.10.40.0/24 - GW .1"]
        TV["TV\nDHCP 10.10.40.100-200\nDNS: 10.10.20.20"]
        PS5["PS5\nDHCP 10.10.40.100-200\nDNS: 10.10.20.20"]
        FUTSW["(future 2nd switch\nfor more media ports)"]
    end

    SW ---|"Access: VLAN 10"| MGMTPC
    SW ---|"Access: VLAN 20"| PVE1
    SW ---|"Access: VLAN 20"| PVE2
    SW ---|"Access: VLAN 20"| PVE3
    SW ---|"Access: VLAN 20"| EVE
    SW ---|"Access: VLAN 30"| LAPTOP
    SW ---|"Access: VLAN 40"| TV
    SW ---|"Access: VLAN 40"| PS5
    SW ---|"Access: VLAN 40"| FUTSW

    PIHOLEA -.->|"Keepalived VRRP"| DNSVIP
    PIHOLEB -.->|"Keepalived VRRP"| DNSVIP
    NEBULA -.->|"Teleporter sync"| PIHOLEA
    NEBULA -.->|"Teleporter sync"| PIHOLEB
    LAPTOP -.->|"DNS queries"| DNSVIP
    TV -.->|"DNS queries"| DNSVIP
    PS5 -.->|"DNS queries"| DNSVIP
    LAPTOP -.->|"HTTPS :8123"| HA

    style V10 fill:#fde9d9
    style V20 fill:#d9e9fd
    style V30 fill:#d9fde9
    style V40 fill:#f9d9fd
```

## Notes
- All addresses are RFC1918 private space (10.10.x.0/24), not reachable from the internet and safe to publish.
- The EVE-NG VM (10.10.20.50), the DNS VIP (10.10.20.20), and the Home Assistant VM (10.10.20.30, TRUSTED only) are the only hosts in the PVE/cluster VLAN reachable from other VLANs. See `ADDRESSING.md` and `PFSENSE-SETUP.md` for the firewall rules that enforce this.
- Clients only ever talk to the DNS VIP, never directly to Pi-hole instance A or B; Keepalived decides which real instance answers, and Nebula Sync keeps both instances' config identical so it doesn't matter which one is live.
- I update this diagram whenever hardware or VLANs change so it reflects the current build.
