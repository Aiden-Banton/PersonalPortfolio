# Lab 05: Junos first-boot and CLI fundamentals

| | |
|---|---|
| **Platform** | EVE-NG |
| **Cert objective** | JNCIA-Junos (JN0-106) — Junos OS fundamentals, user interfaces, configuration basics |
| **Device images** | Juniper vSRX or vMX — **not yet in [`../../EVE-NG-Images/`](../../EVE-NG-Images/README.md)**, add before starting |
| **Time to build** | |
| **Status** | planned |
| **Ties into** | JNCIA-Junos track (private, not yet published) |

## Goal
Get comfortable in Junos by rebuilding something already understood in IOS, so the only new variable is the OS. Interfaces, a couple of static routes, and a firewall filter — then compare the workflow to Lab 01 and Lab 02 line for line.

## The IOS-to-Junos gaps worth documenting
This is the real value of the lab; Junos differs structurally from IOS, not just in syntax:

| Concept | IOS | Junos |
|---|---|---|
| Config model | Changes apply immediately | Candidate config, applied on `commit` |
| Undo | `no <command>`, or reload without saving | `rollback`, with 50 stored rollbacks |
| Modes | user EXEC / privileged EXEC / config | operational mode / configuration mode |
| Viewing config | `show running-config` | `show configuration`, or `| display set` |
| ACL equivalent | `access-list` / `ip access-group` | firewall filter, applied to an interface |
| Interface naming | `GigabitEthernet0/0` | `ge-0/0/0` |

The `commit` / `rollback` model is the one that actually changes how you work — a bad filter locks you out in IOS, but `commit confirmed` in Junos rolls itself back if you don't confirm. Worth testing deliberately.

## Steps
1.
2.

## Verification
| Check | Command | Expected result | Output file |
|---|---|---|---|
| Interfaces up | `show interfaces terse` | ge- interfaces up/up with expected addresses | |
| Config committed cleanly | `commit check` then `commit` | No errors | |
| Routes present | `show route` | Static routes in inet.0 | |
| Rollback works | `rollback 1` then `show | compare` | Previous config restored | |
| `commit confirmed` self-rescues | `commit confirmed 2`, then wait | Config reverts without intervention | |

## What broke

## Takeaway
