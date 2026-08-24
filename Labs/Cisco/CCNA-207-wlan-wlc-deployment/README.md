# CCNA-207 — WLAN Deployment: WLC, SSIDs, and the Management/Client Split

| | |
|---|---|
| **Covers** | 2.7 (AP/WLC management access), 2.8 (WLAN creation and security settings), 2.9 (WLAN QoS profile), 1.11 (wireless principles) |
| **Tier** | T2 Applied |
| **Platform** | Packet Tracer 8.2+ (WLC 3504 and lightweight AP) |
| **Time** | 45–55 minutes |
| **Prerequisites** | VLANs and trunking (CCNA-101 or equivalent); DHCP basics help |
| **Status** | built |

---

## Why this lab exists

Wireless is the sub-topic most CCNA candidates skip in labs, because the wired topics feel
more fundamental and wireless "seems like GUI clicking." The exam does not agree — 2.6
through 2.9 plus 1.11 and 5.9 together carry real weight, and the questions are specific:
which interface a WLAN maps to, what a WLC management interface actually is, where a
security policy is set.

The concept that trips people up is that a WLC has **several different interfaces that all
sound like they might be management**, and mapping a WLAN to the wrong one produces a
network that associates fine and passes no traffic. That's the centre of this lab.

## Scenario

Alderway Community College is adding wireless to a single campus building. They want two
SSIDs on the same physical infrastructure:

- **AC-Staff** — staff devices, WPA2-PSK, mapped to the staff VLAN
- **AC-Guest** — visitor devices, open with a captive-portal-style separation, mapped to a
  guest VLAN with no route to staff resources

A lightweight AP is already cabled to an access switch. A WLC sits on the management VLAN.

## Topology

```
      [ R1 ]  10.20.99.1  10.20.10.1  10.20.20.1   (router-on-a-stick, 802.1Q)
         |
      G0/0/0 trunk
         |
      [ SW1 ]  access switch
       /    |          \
  Gi0/1   Gi0/2       Gi0/3
  WLC1     AP1        PC1 (wired staff, 10.20.10.100)

  VLAN 99  Management   10.20.99.0/24   WLC1 = .10
  VLAN 10  Staff        10.20.10.0/24
  VLAN 20  Guest        10.20.20.0/24
```

Topology diagram: not yet drawn — the ASCII diagram above is canonical for now. Runnable
file: [`topology/CCNA-207.pkt`](./topology/) *(add when built)*

## Addressing

Pre-calculated. Full table in [`addressing.md`](./addressing.md).

| Device | Interface / VLAN | Address | Notes |
|---|---|---|---|
| R1 | G0/0/0.99 | 10.20.99.1/24 | Management gateway |
| R1 | G0/0/0.10 | 10.20.10.1/24 | Staff gateway, DHCP pool |
| R1 | G0/0/0.20 | 10.20.20.1/24 | Guest gateway, DHCP pool |
| WLC1 | Management | 10.20.99.10/24 | Gateway 10.20.99.1 |
| AP1 | — | DHCP from VLAN 99 | Joins WLC over the management network |
| PC1 | — | DHCP from VLAN 10 | Wired reference host |

---

## Part 1 — Wired foundation

**Task 1.** Create VLANs 10, 20 and 99 on SW1 with the names above.

**Task 2.** Configure the SW1 uplink to R1 as an 802.1Q trunk carrying all three VLANs.

**Task 3.** Put WLC1's port and AP1's port in the correct VLAN. Think about which VLAN
each belongs to before configuring — they are not the same question.

> **Why they differ:** the AP needs an address so it can *find and join* the controller,
> which happens on the management network. Client traffic never rides the AP's own VLAN —
> it is tunnelled to the controller and dropped onto whichever VLAN the WLAN maps to. So
> the AP's port VLAN says nothing about where wireless clients end up.

**Task 4.** Configure router-on-a-stick sub-interfaces on R1 for all three VLANs, plus a
DHCP pool for VLANs 10, 20 and 99.

**Checkpoint:**

| Check | Command | Expected | Save to |
|---|---|---|---|
| VLANs exist and ports assigned | `show vlan brief` | 10, 20, 99 with correct ports | `output/01-show-vlan-brief.txt` |
| Trunk up, all VLANs allowed | `show interfaces trunk` | Trunk forwarding 10, 20, 99 | `output/02-show-trunk.txt` |
| Sub-interfaces up | `show ip interface brief` on R1 | Three sub-interfaces up/up | `output/03-r1-interfaces.txt` |
| Wired host gets an address | `ipconfig` on PC1 | 10.20.10.x, gateway .1 | `output/04-pc1-dhcp.txt` |

---

## Part 2 — Controller and AP join

**Task 5.** Configure WLC1's management interface with the address from the table and
confirm you can reach its management address from PC1.

**Task 6.** Bring AP1 up and confirm it registers with the controller. Record how long it
takes and what the AP does while it is joining.

> If the AP never joins, the fault is almost always one of three things: it has no address,
> it has an address but no route to the controller, or it cannot discover the controller's
> address. Check them in that order — each is cheap to test and rules out the next.

| Check | Where | Expected | Save to |
|---|---|---|---|
| Management reachable | ping WLC1 from PC1 | Success | `output/05-ping-wlc-mgmt.txt` |
| AP registered | WLC → Wireless → access points | AP1 listed, status registered | `output/06-ap-joined.txt` |

---

## Part 3 — The two WLANs

**Task 7.** Create WLAN **AC-Staff**:

- SSID `AC-Staff`, broadcast enabled
- Mapped to the **staff** VLAN interface
- Security: WPA2-PSK. Choose a passphrase and record it in your own notes, not in any file
  you would commit.

**Task 8.** Create WLAN **AC-Guest**:

- SSID `AC-Guest`, broadcast enabled
- Mapped to the **guest** VLAN interface
- Open authentication

**Task 9.** Before you associate any client, answer this in your notes: a WLAN mapped to
the wrong dynamic interface will still let a client associate and authenticate. What
exactly fails, and at what point does the user notice?

> This is the whole point of the lab. Association and authentication are radio-layer and
> security-layer events; VLAN mapping determines where the client's frames are *placed*
> after that succeeds. Get it wrong and the client shows "connected" with no address, or
> an address from the wrong scope and no path to anything.

**Task 10.** Associate a wireless client with each SSID and verify it lands in the correct
subnet.

| Check | Where | Expected | Save to |
|---|---|---|---|
| Both WLANs enabled | WLC → WLANs | AC-Staff and AC-Guest enabled, correct interfaces | `output/07-wlan-list.txt` |
| Staff client subnet | client `ipconfig` | 10.20.10.x | `output/08-staff-client-ip.txt` |
| Guest client subnet | client `ipconfig` | 10.20.20.x | `output/09-guest-client-ip.txt` |
| Guest isolation | guest client → PC1 | Should fail once ACL applied (Task 11) | `output/10-guest-isolation.txt` |

---

## Part 4 — Guest separation and QoS

**Task 11.** Apply an ACL on R1 so that the guest VLAN can reach the internet path but not
the staff VLAN. Verify from a guest client that staff resources are unreachable and that
DNS/DHCP still work.

> Order matters here in the same way it does on any firewall: a permit that is too broad
> placed above your deny makes the deny unreachable, and the ACL will look correct in the
> config while doing nothing.

**Task 12.** Set the QoS profile on AC-Guest to **Bronze** and on AC-Staff to **Silver** or
better. Note in your own words what the profile actually changes.

| Check | Where | Expected | Save to |
|---|---|---|---|
| ACL applied and matching | `show access-lists` on R1 | Non-zero match counters after testing | `output/11-acl-matches.txt` |
| Guest cannot reach staff | ping guest → PC1 | Fails | `output/12-guest-blocked.txt` |
| Guest still has service | guest → gateway, DNS | Success | `output/13-guest-services-ok.txt` |
| QoS profiles set | WLC → WLANs → QoS | Bronze / Silver | `output/14-qos-profiles.txt` |

---

## What to write down

In `output/notes.md`:

1. Name every WLC interface you touched and what each one is for. If you can't distinguish
   the management interface from a dynamic interface without looking it up, do this part again.
2. What is the actual difference between an AP's VLAN and a WLAN's VLAN?
3. You configured WPA2-PSK. In one sentence each: what would change with WPA2-Enterprise,
   and what does 802.1X add that PSK cannot provide?

---

**Answer key:** [`solutions/README.md`](./solutions/). Look after, not before.
