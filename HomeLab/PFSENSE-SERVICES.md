# pfSense Services Setup Guide

Everything I run on the pfSense box *besides* routing and firewalling: DNS Resolver, NTP, pfBlockerNG, SNMP, WireGuard, Suricata, and ntopng. `PFSENSE-SETUP.md` covers the VLANs and rules that have to exist before any of this; this guide assumes that's already done and tested.

Two goals here, and they pull in slightly different directions:

1. **Make the firewall earn its keep.** It's a box that's already powered on 24/7 with a view of every packet in the lab. Time sync, DNS for the infrastructure VLANs, and network visibility all belong there rather than on a Proxmox node.
2. **Deliberately load the hardware.** I want to know where a Protectli Vault FW4B actually falls over, not guess. So the last section is a benchmarking method: baseline first, enable one service at a time, measure after each.

Those goals conflict in one specific way, and it's worth saying up front: **the services that stress the box hardest (Suricata, ntopng) are also the ones most likely to make the lab feel broken if I turn them on carelessly.** The order in this guide is deliberately cheapest-to-heaviest for that reason.

## The hardware I'm working with

| Spec | Protectli Vault FW4B |
|---|---|
| CPU | Intel Celeron J3160, 4C/4T, 1.6GHz base / 2.24GHz burst |
| AES-NI | Yes (matters for WireGuard/IPsec throughput) |
| RAM | DDR3L-1600 SODIMM, **8GB maximum** |
| NICs | 4x Intel Gigabit (i211-class) |
| Storage | mSATA SSD |
| Cooling | Fanless |

Two numbers set the ceiling for everything below.

**8GB is the hard RAM cap.** There is no upgrade path. Suricata, pfBlockerNG's IP tables, and ntopng are all memory-hungry, and unlike the M700s in `SERVICES.md` I can't just buy bigger sticks. **If the box currently has 4GB, upgrade to 8GB before starting Suricata or ntopng** — 4GB is fine for base pfSense plus DNS/NTP and nothing else.

**The J3160 routes a full gigabit fine unloaded, but inline IDS/IPS is single-flow CPU-bound and can roughly halve usable throughput.** Real-world reports from people running Suricata + pfBlockerNG on a J3160 with a 1Gbps line describe CPU pegged above 90%, sometimes 99%. That's not automatically a failure — if latency, loss, and throughput are still fine, high CPU is just a well-utilized box — but it means Suricata is a *tuning exercise* on this hardware, not a checkbox. I've written that section accordingly.

## Version note

Written against **pfSense CE 2.8.1** (the current CE release). Menu paths in 2.8.x are stable enough that these should hold on nearby versions, but if a path doesn't match, check the Netgate docs linked at the bottom rather than hunting through the GUI.

## Prerequisites

- `PFSENSE-SETUP.md` Steps 1–6 complete: VLANs 10/20/30/40 up, firewall rules in place, tested.
- A config backup taken **before** starting (`Diagnostics > Backup & Restore`). Keep it out of this repo — see the security note in `PFSENSE-SETUP.md`.
- `System > Update` shows the box on a current release. Install packages *after* updating, not before; a pfSense upgrade reinstalls packages and it's cleaner to not be mid-migration.

---

## Turn off RAM disks before installing any logging package

`System > Advanced > Miscellaneous > RAM Disk Settings`.

pfSense can mount `/tmp` and `/var` as RAM disks to spare the SSD from write wear. That's a reasonable default on a plain firewall. It is **actively harmful** once Suricata, pfBlockerNG, or ntopng are involved: those packages write large amounts of log and database data into `/var`, and on an 8GB box those writes are competing for the same RAM the packages need to run. Worse, RAM disk contents are lost on reboot, so alert history and block statistics vanish every restart.

**Confirm RAM disks are disabled before installing anything below.** If they're currently enabled, disabling them requires a reboot, so do it now rather than discovering it three services later.

The tradeoff is SSD write wear, which is real but overstated for a homelab — an mSATA SSD in a firewall writing logs will outlast the platform's usefulness. If write endurance genuinely worries you, the better fix is remote syslog (`Status > System Logs > Settings > Remote Logging`) pointing at something on VLAN 20, not RAM disks.

---

## Service 1: DNS Resolver (Unbound)

**Load: negligible. Do this first.**

### What this is and isn't

`DNS-HA-SETUP.md` already puts a Keepalived VIP (`10.10.20.20`) fronting two Pi-hole + Unbound instances, and `PFSENSE-SETUP.md` Step 7 points TRUSTED and MEDIA's DHCP at it. **That doesn't change.** Pi-hole stays the client-facing resolver for VLANs 30 and 40.

So what is pfSense's resolver doing?

- **Resolving for pfSense itself.** Package updates, pfBlockerNG feed downloads, NTP pool lookups, and DNS resolution for firewall aliases all go through it. It has to work regardless of whether the Pi-hole pair is up.
- **Serving MGMT (10) and PVE (20).** Per `ADDRESSING.md`, those two VLANs are static-only and use pfSense as their DNS. That's intentional: **the Proxmox nodes must not depend on DNS that runs as guests on themselves.** If both Pi-hole LXCs are down, the nodes hosting them still need working name resolution to be recoverable. Circular dependency avoidance is the whole point.
- **Holding lab DNS records.** Host overrides for `pve1`, `pve2`, `pve3`, `eve`, etc., so the infrastructure VLANs get real names without editing hosts files.

### Configuration

`Services > DNS Resolver > General Settings`:

| Setting | Value | Why |
|---|---|---|
| Enable DNS Resolver | ✅ | |
| Network Interfaces | MGMT, PVE, Localhost | **Not "All".** TRUSTED/MEDIA use the Pi-hole VIP; binding here would let them bypass filtering by pointing at the gateway |
| Outgoing Network Interfaces | WAN | |
| DNSSEC | ✅ | Validates the chain of trust from root down |
| DNS Query Forwarding | ❌ | Leave unchecked = full recursion to root servers, same posture as Unbound in `DNS-HA-SETUP.md` |
| DHCP Registration | ❌ | MGMT/PVE have no DHCP pool; nothing to register |
| Static DHCP | ❌ | Same reason |

Then `System > General Setup`: leave the DNS Servers fields **empty** and **uncheck "DNS Server Override"**. Empty + recursion enabled means pfSense resolves from root itself. The override checkbox lets your ISP's DHCP hand pfSense their DNS servers, which silently defeats the whole recursive setup — this is the single most common way this configuration gets quietly broken.

### Host overrides

Bottom of the DNS Resolver page, `Host Overrides > Add`. One per fixed device from `ADDRESSING.md`:

| Host | Domain | IP |
|---|---|---|
| `pve1` | `lab.local` | `10.10.20.11` |
| `pve2` | `lab.local` | `10.10.20.12` |
| `pve3` | `lab.local` | `10.10.20.13` |
| `eve` | `lab.local` | `10.10.20.50` |
| `dns` | `lab.local` | `10.10.20.20` |
| `ha` | `lab.local` | `10.10.20.30` |
| `docker` | `lab.local` | `10.10.20.40` |
| `fw` | `lab.local` | `10.10.10.1` |

Pick a domain that can't collide with anything real. `.local` is technically reserved for mDNS and can confuse Avahi/Bonjour on Apple devices; `.lab` or a subdomain of a domain you actually own (`lab.example.com`) is cleaner if you have one. I'm using `lab.local` here for readability, but if there are Macs or iPhones on TRUSTED, use something else.

Set the same value in `System > General Setup > Domain` so unqualified lookups resolve.

### The loop you must not create

Do **not** set a domain override or forwarder on pfSense pointing at `10.10.20.20`, while Pi-hole forwards upstream to its local Unbound. Two independent recursive resolvers is the correct design here — each resolves from root on its own. Chaining them creates a dependency loop where a Pi-hole restart takes out pfSense's DNS and vice versa, which is precisely what the HA design exists to prevent.

The one legitimate use for a **domain override** would be if the Pi-hole instances held authoritative records for a domain pfSense needed to resolve. They don't — the host overrides above cover the same ground on the pfSense side. Skip it.

### Verify

From the pfSense box (`Diagnostics > Command Prompt`, or SSH):

```
drill -S pve1.lab.local @127.0.0.1
drill -S cloudflare.com @127.0.0.1
```

The second should return `ad` in the flags (DNSSEC-authenticated). From a Proxmox node on VLAN 20:

```
dig pve2.lab.local
dig +dnssec cloudflare.com
```

Then the negative test, which matters more: from a laptop on TRUSTED, `dig @10.10.30.1 google.com` should **fail or time out**. If it answers, the Network Interfaces binding above is wrong and clients can bypass Pi-hole by querying their gateway directly.

---

## Service 2: NTP server

**Load: negligible. Highest value-per-watt on this list.**

### Why this is worth doing

`DEPLOYMENT-GUIDE.md` step 3.5 already flags that Proxmox corosync is clock-sensitive — cluster quorum genuinely misbehaves when node clocks drift apart. Right now every node, LXC, and VM independently reaches out to the public NTP pool. That means clock accuracy for a cluster that cares about it depends on N separate internet paths, and any device that can't reach the internet (or is on a VLAN with tight egress rules) has no time source at all.

Running NTP on pfSense collapses that to one internal, low-latency source that every VLAN can reach. It also makes log correlation across pfSense, Proxmox, Pi-hole, and Home Assistant actually meaningful — chasing an incident across four devices whose clocks disagree by 30 seconds is miserable.

Concretely: pfSense syncs to the public pool over WAN (stratum 2–3), and every device in the lab syncs to pfSense over a sub-millisecond LAN hop. That's a strictly better topology than what's there now.

### Configuration

`Services > NTP > Settings`:

| Setting | Value | Notes |
|---|---|---|
| Interface(s) | MGMT, PVE, TRUSTED, MEDIA | **Do not select WAN.** An NTP server open to the internet is an amplification-attack reflector. This is the one setting on this page that has real security consequences |
| Time servers | `0.pool.ntp.org` through `3.pool.ntp.org` | Netgate recommends 3–5 sources; ntpd needs multiple to detect a "falseticker" — a source that's wrong but confident. With one source it has no way to know |
| Prefer | leave unchecked on all | Let ntpd pick based on measured quality |
| NoSelect | unchecked | |
| Orphan Mode | `12` | Default. If WAN dies, pfSense keeps serving time from its own clock at stratum 12 rather than refusing to answer. Clocks then drift together instead of apart, which is what corosync actually cares about |
| Enable RRD graphs | ✅ | Free clock-offset/jitter graphs under `Status > Monitoring` |
| Log peer messages | ❌ | Noisy, only useful when debugging |

Set `System > General Setup > Timezone` to your actual local timezone before any of this. Everything downstream inherits it, and fixing it later means re-reading every log you've already collected.

### ACLs

`Services > NTP > ACLs`. Default query policy: **deny**, then allow each lab subnet explicitly:

```
10.10.10.0/24  allow
10.10.20.0/24  allow
10.10.30.0/24  allow
10.10.40.0/24  allow
```

Same narrow-to-broad philosophy as the firewall rules in `PFSENSE-SETUP.md`. This is belt-and-suspenders on top of not binding WAN, but ACLs are free and the failure mode they prevent (being an open reflector) is the kind that gets you an angry email from your ISP.

### Firewall rules

Each VLAN needs to reach its own gateway on UDP/123. MGMT already has `MGMT net → any`, so it's covered. PVE, TRUSTED, and MEDIA all have a broad `→ any` rule that technically covers it too, but that rule is for internet egress — traffic to the gateway itself is worth making explicit so it survives future tightening of those broad rules.

On the **PVE**, **TRUSTED**, and **MEDIA** tabs, above the general allow rule:

- Pass · Source = *(this VLAN)* net · Destination = **This Firewall** · Port = 123 · Protocol = UDP

Using the "This Firewall" destination alias rather than a hardcoded gateway IP means one identical rule works on every tab.

### Hand it out via DHCP

`Services > DHCP Server`, TRUSTED and MEDIA tabs. Expand **Other Options** / "Show Advanced" near the bottom and set **NTP Servers** to that VLAN's gateway — `10.10.30.1` for TRUSTED, `10.10.40.1` for MEDIA. That's DHCP option 42; clients pick it up on next lease renewal with no per-device config.

> **Kea DHCP gotcha, specific to 2.8.** pfSense 2.8 makes **Kea** the default DHCP backend (ISC dhcpd is end-of-life upstream and being removed in a future release). Two things bite here:
> 1. **Kea accepts only IP addresses in the NTP Servers field, not hostnames.** Put `10.10.30.1`, not `ntp.lab.local`.
> 2. There's a reported bug where assigning NTP servers to Kea DHCP clients causes the **DHCP service itself to fail to stay running**. After saving this, check `Status > Services` and confirm the DHCP server is still green, and check `Status > System Logs > System > General` for Kea errors. If DHCP stops staying up, clear the NTP Servers field, restart the service, and hand out NTP via the device side instead — the value of option 42 isn't worth losing DHCP over.
>
> Verify DHCP survived this change *before* moving on. A firewall handing out no leases is a far bigger problem than clients using the public NTP pool.

MGMT and PVE are static-only, so those get configured on the device.

pfSense 2.8 also added **NTP authentication keys** (symmetric key auth between server and client). Overkill for a flat lab where every NTP client is on a VLAN you control, but worth knowing it's there if you ever serve time to something less trusted.

### Point the Proxmox nodes at it

This is the part that actually delivers the corosync benefit, and it's done on the nodes, not pfSense. On **each of the 3 nodes**, edit `/etc/chrony/chrony.conf` (Proxmox VE 8+ ships chrony, not ntpd — older PVE 7 uses `/etc/systemd/timesyncd.conf` instead):

```
# Comment out or remove the existing Debian pool lines, then:
server 10.10.20.1 iburst
```

Then:

```
systemctl restart chronyd
chronyc sources -v
```

The `^*` marker should appear next to `10.10.20.1` within a minute or two, meaning it's the selected synchronization source. `chronyc tracking` shows the actual offset — anything under a few milliseconds is healthy on a LAN hop.

Do these **one node at a time**, confirming `chronyc sources` looks right before moving to the next. Changing time sources on all three cluster members simultaneously is a needless way to disturb quorum.

### Verify

`Status > NTP` on pfSense shows peer status, stratum, offset, and jitter. You want at least one peer marked with `*` (system peer) and offsets in the low milliseconds. From a client: `ntpdate -q 10.10.30.1` or `chronyc sources`.

Give it 10–15 minutes after enabling before judging anything. ntpd deliberately takes its time selecting sources and refuses to be rushed; an unsynchronized reading in the first few minutes is normal, not a fault.

---

## Service 3: SNMP

**Load: negligible.**

Not a service in its own right so much as groundwork. `SERVICES.md`'s Monitoring section lists LibreNMS and Checkmk as the upgrade path once deeper network visibility is worth it — both are SNMP-based, and both need this enabled on pfSense to see anything. It costs nothing to turn on now.

`Services > SNMP`:

| Setting | Value |
|---|---|
| Enable | ✅ |
| Polling Port | `161` |
| System Location / Contact | whatever's useful to you |
| Read Community String | **not `public`** — generate a random string, store it in your password manager |
| SNMP Daemon Bind Interface | **MGMT only** |
| Modules | Enable MibII, Host Resources, Interfaces (Netgraph and PF are optional; PF gives firewall state counts, which is genuinely interesting) |

Binding to MGMT only, per `ADDRESSING.md`'s "MGMT → any" jump-network model, is the important part. SNMP v2c community strings are transmitted in cleartext and are effectively a read password for your entire device inventory — that traffic should never traverse TRUSTED or MEDIA, and absolutely never WAN.

Nothing consumes this yet. That's fine; when a monitoring stack goes in, the firewall is already talking.

---

## Service 4: pfBlockerNG-devel (IP/GeoIP only)

**Load: light-to-moderate. ~200–400MB RAM.**

`System > Package Manager > Available Packages` → **pfBlockerNG-devel**. The devel branch is the maintained one; plain `pfBlockerNG` is stale and lacks current DNSBL features.

### The important decision: DNSBL stays OFF

This is the non-obvious call in this whole guide, so here's the reasoning in full.

pfBlockerNG has two halves that are commonly treated as one product:

- **DNSBL** — domain blocking, implemented by hooking pfSense's Unbound resolver.
- **IP/GeoIP blocking** — firewall-level blocking of address ranges, implemented as pf tables and floating rules.

**DNSBL only sees queries that reach pfSense's resolver.** In this lab, TRUSTED and MEDIA clients query `10.10.20.20` (the Pi-hole VIP) — their DNS traffic never touches pfSense's Unbound. DNSBL would therefore filter nothing for the clients that matter, while still consuming RAM, downloading feeds, and adding a large blocklist into Unbound. It's cost with no benefit, and worse, it produces a dashboard full of near-zero block counts that looks like a working defense.

Pi-hole already does domain blocking, does it better for this topology, and has an HA pair behind it. Enabling DNSBL is duplicating a solved problem in the wrong place.

**IP/GeoIP blocking has no such limitation.** It operates on packets in the firewall's data path, entirely independent of which DNS server a client uses. It blocks things Pi-hole structurally cannot — direct-to-IP connections, hardcoded C2 addresses, and inbound scanning against WAN. It's the half of pfBlockerNG that genuinely complements Pi-hole rather than competing with it.

So: **DNSBL disabled, IPv4/GeoIP enabled.** Note that most pfBlockerNG tutorials online are DNSBL-centric because most people don't run a separate Pi-hole; this configuration deliberately diverges from them.

### Configuration

`Firewall > pfBlockerNG > General`:

- Enable pfBlockerNG: ✅
- Keep settings after deinstall: ✅ (saves re-doing all of this after a package reinstall)
- CRON: `Every hour` at a random minute offset, or `Every 4 hours` if you'd rather minimize churn. Feed downloads are the only time this package does meaningful work
- Inbound Firewall Rules: **WAN**
- Outbound Firewall Rules: **MGMT, PVE, TRUSTED, MEDIA**

`Firewall > pfBlockerNG > DNSBL`:

- Enable DNSBL: ❌ — per the reasoning above

`Firewall > pfBlockerNG > IPv4` — start deliberately small:

| Feed | Action | Purpose |
|---|---|---|
| PRI1 (Emerging Threats, Spamhaus DROP, etc.) | Deny Both | Known-hostile addresses. Low false-positive rate, high value |
| Abuse.ch feeds | Deny Both | Malware C2 tracking |

`Firewall > pfBlockerNG > IPv6`: leave off unless you've deployed IPv6 in the lab. You haven't per `ADDRESSING.md`.

**GeoIP** requires a free MaxMind license key (`Firewall > pfBlockerNG > IP > MaxMind License Key`; sign up at maxmind.com). Once configured, the useful setting is inbound-only continent blocking on WAN. Do **not** block continents outbound — that breaks CDNs, package mirrors, and streaming services in ways that surface days later as "the internet is weirdly broken" and are miserable to trace back to this.

### The failure mode to watch for

"Deny Both" on outbound means a false positive in a feed silently blocks legitimate traffic. When something inexplicably stops working after enabling this, `Firewall > pfBlockerNG > Reports > Alerts` is the first place to look — it shows exactly which rule and which feed dropped a packet. Adding the address to a custom allowlist takes one line.

Start with PRI1 only. Add feeds one at a time, a few days apart. Resist the urge to enable everything on day one — you'll have no way to attribute a breakage to a specific feed, and that's how people end up disabling the whole package in frustration.

---

## Service 5: WireGuard

**Load: light. AES-NI on the J3160 handles the crypto; expect a few hundred Mbps of tunnel throughput, well past any realistic remote-access need.**

### How this relates to the planned Tailscale subnet router

`SERVICES.md` lists a Tailscale subnet router on Node 3 advertising `10.10.20.0/24`. These overlap, but not completely, and running both is defensible:

| | Tailscale (Node 3 LXC) | WireGuard (pfSense) |
|---|---|---|
| Reaches | VLAN 20 only | Any VLAN you write a rule for |
| Depends on | Node 3 + Proxmox + Tailscale's coordination servers | The firewall being up |
| NAT traversal | Automatic | Needs a port forward or a routable WAN IP |
| Setup effort | Very low | Moderate |
| Survives a Proxmox outage | ❌ | ✅ |

That last row is the argument. If the cluster is down — which is exactly when remote access matters most — the Tailscale route is down with it. WireGuard on the firewall is a lower-level, fewer-dependencies path in, and it's the one that lets you reach MGMT (VLAN 10) to fix things.

If you'd rather keep one remote-access mechanism, keep Tailscale for convenience and treat this section as documentation of the break-glass alternative. My preference is both, with WireGuard's firewall rules kept deliberately tight.

### Setup

`System > Package Manager` → install **WireGuard**.

**1. Tunnel** — `VPN > WireGuard > Tunnels > Add Tunnel`:

| Setting | Value |
|---|---|
| Description | `WG_REMOTE` |
| Listen Port | `51820` |
| Interface Keys | Click **Generate** — copy the **public** key, you'll need it for clients |
| Interface Addresses | `10.10.50.1/24` |

`10.10.50.0/24` is a new subnet reserved for the tunnel, deliberately outside VLANs 10–40 so it's distinguishable in firewall rules and logs. Add it to `ADDRESSING.md` when this goes live.

**2. Peer** — `VPN > WireGuard > Peers > Add Peer`, one per client device:

| Setting | Value |
|---|---|
| Tunnel | `tun_wg0` |
| Description | e.g. `laptop-remote` |
| Dynamic Endpoint | ✅ (roaming clients) |
| Public Key | the client's public key |
| Allowed IPs | `10.10.50.2/32` — one /32 per peer, never a broader range |

Generate client keys on the client (`wg genkey | tee private.key | wg pubkey`), not on pfSense. The private key should never exist on two machines.

**3. Assign the interface** — `Interfaces > Assignments`. `tun_wg0` appears as an available port. Assign it, enable it, name it `WIREGUARD`, IPv4 Configuration Type = **None** (the tunnel already carries its address).

Assigning it is what makes WireGuard traffic subject to normal per-interface firewall rules and NAT. Skip this and you get a tunnel that connects but can't reach anything, which is the most common WireGuard-on-pfSense complaint.

**4. Firewall rules:**

*WAN tab* — one rule, or the tunnel never establishes:
- Pass · Protocol UDP · Destination = WAN address · Port = `51820`

*WIREGUARD tab* — this is where you decide how much access a remote client gets. Narrow-to-broad, same as everywhere else:
- Pass · Source = `10.10.50.0/24` · Destination = `MGMT net` · Port = any
- Pass · Source = `10.10.50.0/24` · Destination = `PVE net` · Port = any

Deliberately no `→ any` rule. This is an emergency-administration path, not a general-purpose VPN, and it shouldn't become one by accident. Everything not listed is caught by the implicit deny.

**5. Client config:**

```ini
[Interface]
PrivateKey = <client private key>
Address = 10.10.50.2/32
DNS = 10.10.20.1

[Peer]
PublicKey = <pfSense tunnel public key>
Endpoint = <your WAN IP or dynamic DNS hostname>:51820
AllowedIPs = 10.10.10.0/24, 10.10.20.0/24
PersistentKeepalive = 25
```

`DNS = 10.10.20.1` points remote clients at pfSense's resolver (which the DNS Resolver section bound to MGMT/PVE), so `pve1.lab.local` resolves over the tunnel. `AllowedIPs` lists only lab subnets, so this is a split tunnel — regular browsing doesn't route through your home connection.

**If your WAN IP is dynamic** (likely — `ADDRESSING.md` notes WAN is ISP DHCP), set up `Services > Dynamic DNS` first and use the hostname as the endpoint. A WireGuard config with a hardcoded IP breaks silently the next time your ISP renews your lease, typically while you're away and need it.

**Client config files contain private keys. They never go in this repo.**

---

## Service 6: Suricata

**Load: heavy. This is the actual stress test.**

Everything above is essentially free on this hardware. Suricata is not, and on a J3160 it's the difference between an idle firewall and a busy one.

### Read this before installing

Reported experience on this class of CPU with a 1Gbps WAN: Suricata plus pfBlockerNG drives CPU to 90%+, occasionally 99%. Inline IPS mode is single-flow CPU-bound and can roughly halve usable throughput.

That's the honest expectation. It does not mean don't do it — a fanless 4-core box running a full IDS at high utilization is a genuinely interesting result to document, and high CPU with acceptable latency and throughput is a well-utilized machine, not a broken one. It means:

- **IDS (alert-only) mode first, not IPS (blocking).** Learn what it flags before letting it drop packets.
- **One interface at a time.** WAN first.
- **Measure before and after.** Baseline section below; capture it before installing.
- **Expect tuning to be the actual work.** Installing Suricata takes 10 minutes. Getting it to a state where the alerts are meaningful and the box isn't melting takes weeks of iteration. That's the skill this exercises.

### Install and configure

`System > Package Manager` → **suricata**.

`Services > Suricata > Global Settings`:

| Setting | Value | Why |
|---|---|---|
| Install ETOpen Emerging Threats rules | ✅ | Free, well-maintained, the sensible default ruleset |
| Install Snort rules | ❌ | Requires an Oinkcode; ETOpen alone is plenty to start |
| Install Feodo Tracker / ABUSE.ch rules | ✅ | Small, high-signal |
| Update Interval | `12 hours` | |
| Update Start Time | `00:30` | Off-peak; rule updates are CPU-spiky |
| Remove Blocked Hosts Interval | `1 hour` | |
| Log to System Log | ❌ | Keep Suricata's noise out of the main system log |

`Services > Suricata > Interfaces > Add`, select **WAN**:

| Setting | Value | Why |
|---|---|---|
| Enable | ✅ | |
| Interface | WAN | |
| Description | `WAN_IDS` | |
| **Block Offenders** | ❌ | **IDS mode. Leave this off for at least two weeks.** Turning it on immediately means false positives silently break things with no baseline to compare against |
| Pattern Matcher Algorithm | `Hyperscan` | Meaningfully faster than the alternatives on Intel; if the interface refuses to start, fall back to `Aho-Corasick` |
| Detect-Engine Profile | **`Low`** | Sets the size of internal detection structures. The default `Medium` assumes far more RAM than 8GB total. Start Low, raise only if alerts are being dropped |
| Max Pending Packets | `1024` | Default. Raising it raises RAM use |
| Stream Memcap / Flow Memcap | leave default initially | Tune from `Services > Suricata > Interfaces > (WAN) > Logs View` if you see memcap drops |

**Categories tab:** do not enable all rule categories. The ETOpen set is enormous and most of it is irrelevant to a home network — SCADA rules, enterprise VoIP, obscure server software you don't run. Every enabled rule costs RAM and per-packet CPU on a box that has little to spare.

Start with:

- `emerging-malware`
- `emerging-trojan`
- `emerging-exploit`
- `emerging-worm`
- `emerging-scan` (noisy on WAN — expect a lot of internet background radiation)
- `emerging-dns`
- Feodo Tracker / abuse.ch

Skip `emerging-policy`, `emerging-games`, `emerging-p2p`, `emerging-chat` unless you specifically want that visibility; they generate high alert volume with low actionability.

**Flow/Stream tab:** if `Status > System Logs > Suricata` shows memcap drops, raise the relevant memcap *slightly* — but on 8GB, a memcap drop is more often a signal to cut rule categories than to allocate more memory.

### WAN or LAN?

WAN sees everything from the internet, including the constant background scanning that hits any public IP. That's where the alerts are, and it's the right first interface.

The catch: with NAT, WAN-side alerts show your public IP as the internal endpoint, so you can't tell *which* lab device was involved. If you later want per-device attribution, run Suricata on a LAN-side interface (TRUSTED is the most interesting — it has the general-purpose clients) instead of, not in addition to. **Do not run Suricata on both WAN and multiple VLAN interfaces on this hardware.** Each interface instance is a separate process with its own full copy of the rule set in memory. Two instances on an 8GB box running everything else in this guide will exhaust RAM.

### After two weeks in IDS mode

Review `Services > Suricata > Alerts`. For every rule that fired on legitimate traffic, suppress it: click the ⊕ next to the alert to add a suppression entry (`Services > Suricata > Pass Lists` / `Suppress`). Add your own lab subnets and any known-good external services to a Pass List.

**Only after the alert log is quiet enough that new entries actually mean something** should you consider enabling Block Offenders. If you skip this and enable blocking on day one, the first false positive will block something you depend on, and because Suricata blocks by adding hosts to a table rather than logging a firewall deny, it'll be non-obvious why. That's the experience that makes people uninstall Suricata and conclude IDS isn't worth it.

If you do enable blocking: set **Which IP to Block** = `SRC` (not `BOTH`), and add a Pass List containing your VLANs, the DNS VIP, and the gateway addresses. Blocking your own infrastructure is an easy and very confusing mistake.

---

## Service 7: ntopng (optional — heaviest RAM consumer)

**Load: heavy, primarily RAM. Only attempt with 8GB installed, and consider it mutually exclusive with Suricata on this box.**

ntopng gives per-host bandwidth accounting, flow history, and traffic breakdown by application — genuinely useful visibility, and it ties into `SERVICES.md`'s Monitoring section as the "what is actually using my bandwidth" answer that Uptime Kuma and Netdata don't provide.

It's also a Redis-backed application storing flow records, and its memory use grows with the number of distinct hosts and flows it tracks. On an 8GB box already running Suricata plus everything above, it's a real risk of memory exhaustion.

`System > Package Manager` → **ntopng**. `Diagnostics > ntopng Settings`:

| Setting | Value |
|---|---|
| Enable | ✅ |
| Interface | **one interface only** — TRUSTED is the most interesting for per-device visibility |
| DNS Mode | `Decode DNS responses and resolve local numeric IPs only` |
| Local Networks | `10.10.10.0/24`, `10.10.20.0/24`, `10.10.30.0/24`, `10.10.40.0/24` |
| Redis / History | keep retention short |

Access it at `https://<pfSense IP>:3000`.

**My recommendation on the FW4B specifically:** treat Suricata and ntopng as an either/or. Run Suricata for a few weeks, capture the numbers, then disable it and run ntopng for a few weeks, and capture those. Comparing the two loads is a better use of this hardware — and a better writeup — than running both badly at once and learning only that 8GB isn't enough.

---

## Benchmarking: actually measuring what the Vault can do

The reason for enabling all this is to find the ceiling. That requires measuring deliberately rather than noticing things feel slow.

### Capture a baseline first

**Before installing any package**, record:

| Metric | Where |
|---|---|
| Idle CPU % | `Status > Dashboard` (add the System Information widget) |
| RAM used (MB and %) | Dashboard |
| MBUF usage | Dashboard — watch this; exhaustion causes packet loss that looks like a NIC fault |
| States / max states | Dashboard |
| WAN throughput, down/up | speedtest from a TRUSTED client |
| LAN-to-LAN throughput | `iperf3` between two hosts on different VLANs (forces routing through pfSense) |
| Latency to gateway | `ping 10.10.30.1`, note avg and jitter |
| Temperature | Thermal Sensors dashboard widget, or `sysctl dev.cpu.0.temperature` — the FW4B is fanless, so thermals are a real ceiling |

**CPU temperature isn't readable out of the box.** By default pfSense only reads ACPI motherboard sensors. For the J3160's on-die sensors, set `System > Advanced > Miscellaneous > Cryptographic & Thermal Hardware > Thermal Sensors` to **Intel Core** (loads the `coretemp` driver), then add the **Thermal Sensors** widget to the dashboard. Do this before capturing the baseline — thermal headroom is one of the more interesting numbers on a fanless box, and it's useless without a pre-load reading to compare against.

For the routed-throughput test, run iperf3 server on a Proxmox node (VLAN 20) and client on the laptop (VLAN 30). That path crosses the firewall, so it measures pfSense's routing performance rather than switch performance.

```
# server, on a VLAN 20 host
iperf3 -s
# client, on VLAN 30
iperf3 -c 10.10.20.11 -t 60
iperf3 -c 10.10.20.11 -t 60 -P 8    # 8 parallel streams
iperf3 -c 10.10.20.11 -t 60 -R      # reverse direction
```

The single-stream vs. 8-stream comparison matters specifically because Suricata is single-flow CPU-bound — a box that holds up fine on 8 parallel streams can still collapse on one big transfer once IDS is inline.

### Measure after each service

Re-run the same battery after enabling **each** service, one at a time, giving each at least 24 hours of normal use before measuring. Record it in a table:

| Stage | Idle CPU | Peak CPU | RAM | MBUF | LAN→LAN (1 stream) | LAN→LAN (8 streams) | WAN down | Temp |
|---|---|---|---|---|---|---|---|---|
| Baseline (routing only) | | | | | | | | |
| + DNS Resolver | | | | | | | | |
| + NTP | | | | | | | | |
| + SNMP | | | | | | | | |
| + pfBlockerNG (IP only) | | | | | | | | |
| + WireGuard | | | | | | | | |
| + Suricata (WAN, IDS) | | | | | | | | |
| + Suricata (WAN, IPS) | | | | | | | | |
| + ntopng | | | | | | | | |

One service per row, one change at a time. The moment you enable two things between measurements, the data stops being able to attribute a regression to either. This is the discipline that turns "I installed some packages" into an actual result worth putting in a portfolio.

### Where to look while it's running

- `Status > Monitoring` — historical graphs for CPU, memory, traffic, states, and (with the RRD option enabled above) NTP offset
- `Diagnostics > System Activity` — live `top`, the fastest way to see *which* process is eating the CPU
- `top -aSH` from a shell — per-thread view; Suricata's worker threads show up individually here, which tells you whether it's actually parallelizing across the J3160's 4 cores or bottlenecked on one
- `Diagnostics > Halt System`… is not what you want, but it's alarmingly close to `Diagnostics > Reboot` in the menu. Worth knowing before you're clicking quickly

### Thresholds worth reacting to

| Signal | Threshold | What it means |
|---|---|---|
| CPU sustained | >85% at idle-ish traffic | Not enough headroom for a traffic burst; cut Suricata rule categories |
| RAM used | >85% | Approaching swap; on an mSATA SSD, swapping is a severe performance cliff |
| MBUF usage | >80% | Raise `kern.ipc.nmbclusters` in `System > Advanced > System Tunables`, or reduce load |
| States | >70% of max | Raise Firewall Maximum States, but check RAM first — states cost memory |
| CPU temp | >80°C | Fanless box in an enclosed space. Improve airflow before adding load |
| Latency to gateway | >5ms, or rising jitter | The firewall is CPU-starved and delaying its own packet processing. This is the symptom users actually notice |

That last one is the one that matters most in practice. Throughput degradation is easy to spot in a benchmark; latency and jitter creep is what makes the network *feel* bad while every number still looks acceptable.

---

## Order of operations

Recommended sequence, each stage verified before starting the next:

1. Disable RAM disks (requires reboot)
2. Capture baseline benchmarks
3. DNS Resolver + host overrides → verify from MGMT/PVE, verify TRUSTED *can't* use it
4. NTP → verify, then repoint Proxmox nodes one at a time
5. SNMP → enable, bind to MGMT, done
6. pfBlockerNG-devel, IP feeds only, PRI1 first → benchmark
7. WireGuard → test from off-network before relying on it
8. Suricata, WAN, IDS mode, minimal categories → benchmark, then two weeks of tuning
9. ntopng, or Suricata IPS mode — pick one, benchmark, don't stack them

Stages 1–5 are safe on a 4GB box. Stage 6 onward is where 8GB starts mattering, and stages 8–9 are where the hardware ceiling actually shows up.

## Gotchas

- **`System > Advanced > Miscellaneous > RAM Disks` must be off** before Suricata/pfBlockerNG/ntopng. Covered above, repeated here because it's the one that costs the most time when missed.
- **Don't enable a package and walk away.** Every service here has a way to silently break something a day later — pfBlockerNG false positives, Suricata blocking a legitimate host, a DNS Resolver interface binding that lets clients bypass Pi-hole. Watch the logs for a day after each change.
- **Take a config backup between each stage**, not just at the start. Rolling back one service is trivial; rebuilding six is not.
- **Package installs and pfSense upgrades interact badly.** Don't install packages immediately before a firmware upgrade — upgrade first, then reinstall packages.
- **pfSense upgrades reinstall packages from scratch.** Keep "Keep settings after deinstall" checked in pfBlockerNG, and screenshot or note your Suricata category selections somewhere outside the box.
- **Every service here is a new listening port.** After each install, check `Firewall > Rules > WAN` and confirm you haven't exposed anything unintentionally. NTP bound to WAN and SNMP reachable from anywhere are the two classic mistakes, and both are covered above — but verify rather than trust.
- **CCNA tie-in, same as `PFSENSE-SETUP.md`'s ACL note:** the NTP ACLs and pfBlockerNG's IP tables are the same allow/deny logic as an IOS `ntp access-group` and a prefix list. That comparison would belong alongside the firewall-rule translation in a planned Cisco ACL lab (not yet built) — it's the kind of thing that shows the concept transferred rather than just the GUI clicks. The DNS and NTP service design here maps to CCNA domain 4.0; see `Certifications/CCNA/` (private, not yet published) for what's covered and what still isn't.

## ⚠️ Security note

Same rules as `PFSENSE-SETUP.md`. Specific to this guide, **never commit**:

- WireGuard private keys or client `.conf` files (they contain a private key and your WAN endpoint)
- SNMP community strings
- MaxMind license keys
- Suricata alert exports (they contain your public IP and internal topology)
- pfBlockerNG reports and ntopng exports (same reason)
- Any screenshot of `Status > Dashboard` or `Status > Monitoring` that shows your WAN address

Benchmark *numbers* are fine and are the interesting part anyway — throughput, CPU, RAM, temperature. Just crop or redact anything showing a public IP.

---

Sources: [pfSense CE 2.8.1 release notes](https://docs.netgate.com/pfsense/en/latest/releases/2-8-1.html), [pfSense CE 2.8.0 new features and changes (Kea default, NTP auth keys)](https://docs.netgate.com/pfsense/en/latest/releases/2-8-0.html), [Bug #15012 — NTP assigned to Kea DHCP clients causes service to fail](https://redmine.pfsense.org/issues/15012), [Hardware Temperature Monitoring (Netgate)](https://docs.netgate.com/pfsense/en/latest/monitoring/status/hardware.html), [pfSense CE configuration recommendations (Protectli KB)](https://kb.protectli.com/kb/pfsense-configuration-recommendations/), [DNS Resolver Configuration (Netgate)](https://docs.netgate.com/pfsense/en/latest/services/dns/resolver-config.html), [Domain Overrides (Netgate)](https://docs.netgate.com/pfsense/en/latest/services/dns/resolver-domain-overrides.html), [NTP Server Configuration (Netgate)](https://docs.netgate.com/pfsense/en/latest/services/ntpd/server.html), [NTPD overview (Netgate)](https://docs.netgate.com/pfsense/en/latest/services/ntpd/index.html), [Protectli Vault FW4B datasheet](https://protectli.com/wp-content/uploads/2025/02/FW4B-Datasheet-20250128.pdf), [Protectli FW4B product page](https://protectli.com/product/fw4b/), [Celeron J3160 with Suricata + pfBlockerNG, real-world CPU load (Netgate Forum)](https://forum.netgate.com/topic/144404/celeron-j3160-enough-or-step-up-to-i3-or-i5), [Best hardware for pfSense 2026 — IDS/IPS throughput impact](https://pfsenselab.com/posts/best-hardware-for-pfsense-2026/), [pfBlockerNG project](https://github.com/pfBlockerNG/pfBlockerNG), [pfBlockerNG setup guide](https://pfsenselab.com/posts/pfsense-pfblockerng-setup/), [WireGuard on pfSense](https://www.wundertech.net/how-to-set-up-wireguard-on-pfsense/).
