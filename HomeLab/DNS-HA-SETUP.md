# DNS/DHCP High-Availability Setup

Two independent Pi-hole + Unbound instances (one per M700), a Keepalived VRRP pair providing a single floating DNS address, and Nebula Sync keeping both instances' config identical. The goal: losing either M700 node doesn't take DNS down for the rest of the lab.

## Architecture at a glance
- **Unbound** (both nodes): recursive resolver, queries the DNS root directly instead of forwarding to a third party. Listens on `127.0.0.1:5335`, local to each instance.
- **Pi-hole** (both nodes): DNS filtering + local DNS records, forwards upstream to its local Unbound instead of a public resolver.
- **Keepalived** (both nodes): VRRP daemon, holds the floating VIP `10.10.20.20`. Node 2's instance is `MASTER` (priority 150), Node 3's is `BACKUP` (priority 100). Health-checks the *actual DNS path*, not just "is the process running."
- **Nebula Sync** (Node 3): polls Node 2's Pi-hole (instance A, `10.10.20.21`, treated as the source of truth) via the Teleporter API and pushes the same config/blocklists to Node 3's Pi-hole (instance B, `10.10.20.22`).

Clients never talk to `.21` or `.22` directly, only to the VIP. See `network-diagram.md` for how this fits into the rest of VLAN 20, and `ADDRESSING.md` for the IP table.

## Before you start
- Node 2 and Node 3 are already joined to the cluster and reachable.
- Static IPs `10.10.20.21` (Node 2) and `10.10.20.22` (Node 3) are reserved per `ADDRESSING.md`, not yet handed out anywhere else.
- **Don't touch pfSense's DHCP DNS setting or add the firewall rules yet.** That's Step 7 in `PFSENSE-SETUP.md`, done only after this whole stack is verified working end to end. Pointing DHCP at a VIP that isn't answering yet locks every TRUSTED/MEDIA device out of DNS.

## Step 1: Create the two LXCs
On each of Node 2 and Node 3:
1. Create a Debian 12 LXC: 1–2 vCPU, 1GB RAM (matches the per-instance floor in `SERVICES.md`), 4–8GB disk.
2. Static IP on VLAN 20: `10.10.20.21/24` (Node 2), gateway `10.10.20.1`; `10.10.20.22/24` (Node 3), same gateway.
3. `apt update && apt full-upgrade`, then `apt install curl dnsutils` on both before continuing.

## Step 2: Install Unbound (both nodes, identical config)
1. `apt install unbound`
2. Create `/etc/unbound/unbound.conf.d/pi-hole.conf`:
   ```
   server:
       verbosity: 0
       interface: 127.0.0.1
       port: 5335
       do-ip4: yes
       do-udp: yes
       do-tcp: yes
       do-ip6: no
       prefer-ip6: no
       harden-glue: yes
       harden-dnssec-stripped: yes
       use-caps-for-id: no
       edns-buffer-size: 1232
       prefetch: yes
       num-threads: 1
       so-rcvbuf: 1m
       private-address: 10.10.10.0/24
       private-address: 10.10.20.0/24
       private-address: 10.10.30.0/24
       private-address: 10.10.40.0/24
       root-hints: "/var/lib/unbound/root.hints"
   ```
   The `private-address` lines are DNS-rebinding protection: they stop an external domain from resolving to an address inside this lab's own VLANs.
3. Fetch current root hints: `curl -o /var/lib/unbound/root.hints https://www.internic.net/domain/named.root`
4. `systemctl restart unbound && systemctl enable unbound`
5. Test recursion works before moving on: `dig @127.0.0.1 -p 5335 pi-hole.net` should return a real answer, not an error.

## Step 3: Install Pi-hole (both nodes)
1. `curl -sSL https://install.pi-hole.net | bash` — worth reading the script first (`curl -sSL https://install.pi-hole.net -o pihole-install.sh`, review, then run) since it's a curl-pipe-bash install running as root.
2. During setup: pick **Custom** upstream DNS, `127.0.0.1#5335` (Unbound, from Step 2). Don't use a public upstream here, that defeats the point of running Unbound.
3. Skip Pi-hole's own DHCP server option entirely. pfSense stays the one DHCP authority in this lab (`ADDRESSING.md`); running two DHCP servers on the same VLAN causes clients to get conflicting leases.
4. After install, set an app password for the API (`Settings > All Settings > Webserver and API`, or `pihole setpassword`) — Nebula Sync needs this to authenticate. Do this on **both** instances.
5. On instance B (Node 3, the replica) only, enable `webserver.api.app_sudo` (`Settings > All Settings`, toggle "Modified settings / All settings" to show all, find it under Webserver and API section, or `pihole-FTL --config webserver.api.app_sudo true`). Nebula Sync's replica writes fail authentication without this.

## Step 4: Keepalived (both nodes)
1. `apt install keepalived`
2. Health-check script, identical on both nodes, `/usr/local/bin/check_pihole.sh`:
   ```bash
   #!/bin/bash
   dig @127.0.0.1 pi.hole +short +timeout=1 >/dev/null 2>&1 || exit 1
   ```
   `chmod +x /usr/local/bin/check_pihole.sh`. This checks the whole local resolution path (Pi-hole → Unbound → answer), not just whether the `pihole-FTL` process exists, so a hung resolver still triggers failover.
3. `/etc/keepalived/keepalived.conf` on **Node 2** (MASTER):
   ```
   vrrp_script chk_pihole {
       script "/usr/local/bin/check_pihole.sh"
       interval 2
       weight -20
   }

   vrrp_instance VI_DNS {
       state MASTER
       interface eth0
       virtual_router_id 51
       priority 150
       advert_int 1
       authentication {
           auth_type PASS
           auth_pass <shared-secret>
       }
       virtual_ipaddress {
           10.10.20.20/24
       }
       track_script {
           chk_pihole
       }
   }
   ```
4. Same file on **Node 3** (BACKUP), only `state` and `priority` differ:
   ```
   vrrp_instance VI_DNS {
       state BACKUP
       interface eth0
       virtual_router_id 51
       priority 100
       ...same as above...
   }
   ```
   `virtual_router_id` and `auth_pass` must match on both nodes; `priority` is what decides which one holds the VIP under normal conditions.
5. `systemctl enable --now keepalived` on both. Confirm the VIP is live on Node 2: `ip a show eth0` should list `10.10.20.20`. It should **not** appear on Node 3 while Node 2 is healthy.

## Step 5: Nebula Sync (Node 3)
Runs as a small standalone LXC on Node 3, next to replica B, using the prebuilt Linux binary directly rather than nesting Docker just for one process; this also means the DNS HA stack doesn't depend on the separate Docker host (`DOCKER-SETUP.md`) being up.

1. Create a lightweight Debian 12 LXC (512MB RAM is plenty), static IP within VLAN 20, no public exposure needed.
2. Install the binary: grab the latest release from https://github.com/lovelaze/nebula-sync/releases/latest, or `go install github.com/lovelaze/nebula-sync@latest` if Go is already on hand.
3. Create `/etc/nebula-sync.env`:
   ```
   PRIMARY=http://10.10.20.21|<instance-A-app-password>
   REPLICAS=http://10.10.20.22|<instance-B-app-password>
   FULL_SYNC=true
   RUN_GRAVITY=true
   CRON=*/30 * * * *
   TZ=<your timezone>
   ```
   `PRIMARY` is instance A (Node 2), matching Keepalived's MASTER; config always flows A → B. `FULL_SYNC=true` does a complete Teleporter export/import each run rather than picking individual settings, simplest option for two instances that should just always match. `chmod 600` this file, it holds Pi-hole app passwords.
4. Systemd unit, `/etc/systemd/system/nebula-sync.service`:
   ```
   [Unit]
   Description=nebula-sync
   After=network-online.target

   [Service]
   EnvironmentFile=/etc/nebula-sync.env
   ExecStart=/usr/local/bin/nebula-sync run
   Restart=on-failure

   [Install]
   WantedBy=multi-user.target
   ```
5. `systemctl enable --now nebula-sync`. `nebula-sync` handles its own cron scheduling internally (the `CRON` env var), the systemd unit just keeps the process alive between runs.

## Step 6: Verify end to end
1. `dig @10.10.20.20 pi-hole.net` from any host on VLAN 20 resolves normally.
2. Add a test domain to instance A's blocklist, wait one Nebula Sync cycle (or trigger a run manually), confirm it's also blocked when queried against instance B directly (`dig @10.10.20.22 <test-domain>`).
3. **Failover test**: `systemctl stop keepalived` on Node 2. Confirm the VIP appears on Node 3 within a few seconds (`ip a show eth0`), and that `dig @10.10.20.20 ...` keeps resolving throughout. `systemctl start keepalived` on Node 2 and confirm the VIP moves back (higher priority wins once it's healthy again).

## Step 7: Point the rest of the network at it
Once Steps 1–6 are verified working, follow **Step 7 in `PFSENSE-SETUP.md`** to add the firewall aliases/rules and switch TRUSTED/MEDIA's DHCP-assigned DNS server over to `10.10.20.20`. Not duplicated here to keep pfSense-side changes in one place.

## Notes / gotchas
- **Sync is one-directional, A → B.** If Node 2 is down and you make changes on instance B (because it's the only one you can reach), those changes get overwritten on the next sync once Node 2 comes back and Nebula Sync runs again. Treat instance A as the only place to make lasting changes; B is a hot spare, not a second source of truth.
- Pi-hole's own DHCP server stays **off** on both instances. This stack solves DNS availability; DHCP stays with pfSense, which is already documented and working.
- Give the Keepalived priorities real separation (150 vs 100, not 101 vs 100) so a brief blip doesn't cause flapping between the two.
- Pi-hole app passwords, the Keepalived `auth_pass`, and `/etc/nebula-sync.env` stay out of this repo — password manager, not here. See the security note in `PFSENSE-SETUP.md`.

Sources: [Pi-hole's own Unbound recipe](https://docs.pi-hole.net/guides/dns/unbound/), [lovelaze/nebula-sync](https://github.com/lovelaze/nebula-sync), [High Availability DNS/DHCP with Pi-hole 6](https://homelab.casaursus.net/high-availability-pi-hole-6/), [Building a Pi-hole failover cluster with Keepalived and Unbound](https://vipinpg.com/blog/building-a-pi-hole-failover-cluster-with-keepalived-and-unbound-recursive-dns-for-zero-downtime-ad-blocking).
