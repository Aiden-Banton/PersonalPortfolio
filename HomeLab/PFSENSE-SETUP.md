# pfSense VLAN Setup Guide

This is the exact configuration process I used to bring up the VLAN scheme in `ADDRESSING.md`, in the order that actually works. Follow it top to bottom the first time through.

## Before you start
- pfSense is configured almost entirely through the **web GUI** (`https://<LAN IP>` from a browser), not the console/terminal.
- The console (keyboard+monitor directly on the firewall, or serial) is **emergency access only**, for setting interface IPs, resetting the web GUI password, or factory reset if you lock yourself out. Don't try to configure VLANs from the console menu; option 8 (Shell) drops to a raw FreeBSD shell, which is not what you want for this.
- Keep a laptop able to plug directly into the console or LAN port for the whole process, in case a rule change locks you out of the web GUI.
- **Before making rule changes**, go to `Diagnostics > Backup & Restore` and download a config backup. See the security note at the bottom before you commit that file anywhere.

## Step 1: Create the VLANs (virtual interfaces)
1. Log into the pfSense web GUI.
2. Go to `Interfaces > Assignments > VLANs` tab.
3. Click **+ Add** and create each of these on your LAN NIC (e.g. `igb1`):
   - VLAN tag `10`, Description: `MGMT`
   - VLAN tag `20`, Description: `PVE`
   - VLAN tag `30`, Description: `TRUSTED`
   - VLAN tag `40`, Description: `MEDIA`

   You're just registering the tags here; nothing is "live" yet.

## Step 2: Assign each VLAN to a logical interface
1. Go to `Interfaces > Assignments` (main tab).
2. In "Available network ports," you'll now see the 4 new VLAN interfaces (e.g. `igb1.10`, `igb1.20`, `igb1.30`, `igb1.40`).
3. Add each one; pfSense will label them OPT1–OPT4.
4. Click into each OPT interface and:
   - Check **Enable interface**
   - Rename it (top field) to `MGMT` / `PVE` / `TRUSTED` / `MEDIA`
   - IPv4 Configuration Type: **Static IPv4**
   - IPv4 Address: the gateway IP for that VLAN (e.g. MGMT = `10.10.10.1/24`, PVE = `10.10.20.1/24`, etc.; see `ADDRESSING.md`)
   - Save, then **Apply Changes** at the top.

> **#1 thing people miss:** a brand-new pfSense interface has no firewall rules and passes nothing, not even between the interface and its own subnet's devices, until you add rules in Step 4. If VLANs "don't work," this is why 90% of the time. It's default-deny behavior, not a bug.

## Step 3: Turn on DHCP where needed
1. Go to `Services > DHCP Server`.
2. Select each VLAN tab that needs DHCP (TRUSTED, MEDIA; skip MGMT and PVE, which are static-only per `ADDRESSING.md`; if you want DHCP on PVE later for flexibility, same steps apply).
3. Check **Enable DHCP server on this interface**.
4. Set the range (e.g. TRUSTED: `10.10.30.100` to `10.10.30.200`).
5. Save.

**Do this only on day one.** Leave the DNS Servers field blank for now (pfSense hands out itself as DNS by default). Come back and fill it in during the "DNS handoff to Pi-hole" step below, once `DNS-HA-SETUP.md` has an actual working DNS VIP to point at; pointing DHCP at a DNS server that doesn't exist yet is how you lock every device on TRUSTED/MEDIA out of the internet.

## Step 4: Firewall rules (this is where "VLANs don't work" gets fixed)
Go to `Firewall > Rules`. There's a tab per interface. Rules are evaluated **top to bottom, first match wins**, so order matters.

**MGMT tab**
- Rule 1: Pass · Source = MGMT net · Destination = any · Port = any
  (your "access everything" jump network)

**PVE tab**
- Rule 1: Pass · Source = PVE net · Destination = any
  (lets Proxmox nodes reach the internet for updates)
- You do *not* need a rule here allowing MGMT in; that's already covered by the MGMT→any rule on the MGMT tab.

**TRUSTED tab**
- Rule 1 (put first): Pass · Source = TRUSTED net · Destination = `10.10.20.50` (EVE-NG VM) · Port = any
- Rule 2: Pass · Source = TRUSTED net · Destination = any (general internet)
- Don't add a rule allowing TRUSTED → PVE net broadly; leaving it out means it's denied by the implicit deny-all.

**MEDIA tab**
- Rule 1 (put first): Pass · Source = MEDIA net · Destination = `10.10.20.50` (EVE-NG VM) · Port = any
- Rule 2: Pass · Source = MEDIA net · Destination = any (internet access)
- Same idea: no rule to the PVE net means it stays blocked.

**Tip:** create a `Firewall > Aliases` entry named `EVE_NG_VM` for `10.10.20.50` before writing these rules. If the VM's IP ever changes, you edit the alias once instead of 4 different rules. Do the same later for `DNS_VIP` (`10.10.20.20`) and `HOME_ASSISTANT` (`10.10.20.30`), see Step 7; same reasoning, one alias beats editing rules on two tabs every time an IP changes.

## Step 5: Configure the managed switch
1. Create VLANs 10, 20, 30, 40 on the switch itself (matching tags).
2. The port going to pfSense = **trunk/tagged** port, tagged for VLANs 10, 20, 30, 40 (all four).
3. Each device port = **access/untagged** port, member of exactly one VLAN:
   - Mgmt Access PC → VLAN 10, untagged
   - Proxmox nodes → VLAN 20, untagged
   - Laptop → VLAN 30, untagged
   - TV, PS5 → VLAN 40, untagged
   - Future 2nd switch → VLAN 40, untagged (or trunk if that switch is also managed and you want more VLANs behind it later)
4. Leave VLAN 1 (default) unused/unassigned to any device port as a light hardening step.

## Step 6: Test incrementally, don't flip everything on at once
1. **MGMT only**: plug the Mgmt Access PC into its port, confirm it gets/holds `10.10.10.x`, and confirm it can ping `10.10.10.1` and reach the pfSense GUI.
2. **Bring up PVE**: plug in one Proxmox node, confirm it gets its static IP and reaches the internet (e.g. `apt update`). Confirm you *can* reach it from the Mgmt Access PC, and confirm you *cannot* reach it from the Laptop on VLAN 30 yet.
3. **Bring up TRUSTED**: confirm the Laptop gets internet + can reach the EVE-NG VM once it exists, but nothing else in VLAN 20.
4. **Bring up MEDIA last**: same checks (internet + EVE-NG only).

## Step 7: DNS handoff to Pi-hole (after `DNS-HA-SETUP.md` is deployed)
Do this once the Pi-hole + Unbound + Keepalived pair from `DNS-HA-SETUP.md` is up and the VIP is answering queries. Don't do it before, see the warning in Step 3.

1. **Create two new aliases** (`Firewall > Aliases`): `DNS_VIP` = `10.10.20.20`, `HOME_ASSISTANT` = `10.10.20.30`.
2. **Add firewall rules** on the **TRUSTED** and **MEDIA** tabs, above the general "allow to any" rule (narrow-to-broad ordering, same reasoning as the EVE-NG rule):
   - TRUSTED: Pass · Source = TRUSTED net · Destination = `DNS_VIP` · Port = 53 (TCP+UDP)
   - TRUSTED: Pass · Source = TRUSTED net · Destination = `HOME_ASSISTANT` · Port = 8123 (TCP)
   - MEDIA: Pass · Source = MEDIA net · Destination = `DNS_VIP` · Port = 53 (TCP+UDP)
   - MEDIA gets no Home Assistant rule; nothing on that VLAN (TV, PS5) needs it, and leaving it out means it's denied by the implicit deny-all, same as PVE net access already is.
3. **Point DHCP at the VIP**: `Services > DHCP Server`, TRUSTED tab and MEDIA tab, set **DNS Servers** to `10.10.20.20`. Save, then have a device on each VLAN release/renew its lease (or just wait out the lease) and confirm `nslookup` resolves through the VIP, not pfSense itself.
4. **Verify failover before trusting it**: from a TRUSTED client, run continuous pings/`dig` queries against `10.10.20.20` while power-cycling Node 2, then Node 3. You should see at most a few seconds of dropped queries during the VRRP failover window, then resolution continues from whichever instance is still up.

## Step 8: Firewall services (after the VLANs are stable)
Once Steps 1–7 are done and the network has been solid for a few days, the firewall has plenty of spare capacity to run services of its own: a DNS Resolver for the MGMT/PVE VLANs, an internal NTP server for the whole lab (which the Proxmox cluster benefits from directly, corosync being clock-sensitive), pfBlockerNG for IP-level blocking, WireGuard as a break-glass remote path, and Suricata/ntopng as a deliberate load test of the Protectli Vault.

Full walkthrough, including how pfSense's resolver is scoped so it doesn't conflict with the Pi-hole VIP from Step 7, is in **`PFSENSE-SERVICES.md`**.

## Best practices / gotchas
- Never delete or disable the rule that lets you reach the pfSense GUI until you've confirmed a replacement rule works; test in a *new* tab/window before closing your existing session.
- Order rules narrow-to-broad: specific rules (like the EVE-NG allow) go *above* general rules (like "allow to any"). pfSense stops at the first match, so ordering mistakes are the most common cause of "my rule isn't working."
- The default deny-all at the bottom of every interface is implicit. You won't see it in the GUI, but it's always there catching anything you didn't explicitly allow.
- Click **Apply Changes** (green banner at the top) after adding/editing rules; they sit pending until you do.
- Once this is stable, it's a great candidate to write up as a CCNA-style ACL exercise: you're hand-building the same allow/deny logic as a standard/extended ACL, just through pfSense's GUI instead of raw IOS `access-list` syntax. That writeup would reimplement this exact rule set in IOS and compare the two side by side — planned as a Cisco ACL lab, not yet built. The VLAN/trunk work in Steps 1–5 would get the same treatment in a planned VLAN/inter-VLAN routing lab, also not yet built.

## ⚠️ Security note before pushing anything else here
These docs are written to be publishable, so a few things should **never** be committed, even accidentally:
- pfSense config exports (`config.xml` from Backup & Restore); these can contain hashed credentials, VPN keys, and your WAN/public IP.
- SSH keys, API tokens, `.env` files, or Proxmox/VM credentials.
- Screenshots that show your public IP, ISP account info, or WAN-side details.

`.gitignore` in this repo already excludes common secret/config-backup file types; see the note in `.gitignore` itself. If you ever export a pfSense backup for your own records, keep it outside this repo (or in a private, encrypted location), not in `HomeLab/`.
