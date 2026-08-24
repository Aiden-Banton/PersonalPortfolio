# JNCIA-101 — Junos CLI, the Candidate Configuration, and Getting Yourself Out of Trouble

| | |
|---|---|
| **Covers** | JN0-106 — Junos OS fundamentals, user interfaces, configuration basics |
| **Tier** | T1 Foundational — commands shown |
| **Platform** | EVE-NG, vJunos-router |
| **Time** | 40–50 minutes |
| **Prerequisites** | Comfortable in a Cisco IOS CLI. This lab is built around that contrast. |
| **Status** | built |

> ### Platform gate — read this before starting
>
> **This is the lab that tells you whether the rest of the Juniper track is viable.**
>
> vJunos-switch is explicitly not supported on EVE-NG because of nested-virtualization
> constraints. vJunos-router *is* documented for EVE-NG, but if your EVE-NG instance is
> itself running as a VM, you are nesting one layer deeper than that documentation assumes.
>
> So Part 0 is a boot test, and it comes first deliberately. If a single vJunos-router node
> will not boot, stop — do not work through three labs' worth of configuration against a
> platform that can't run them. The fallback options are in
> [`../LAB-INDEX.md`](../LAB-INDEX.md#platform-fallbacks).

---

## Why this lab exists

Junos is not IOS with different syntax. The difference that matters is structural: IOS
applies configuration the instant you press enter, and Junos does not. Junos edits a
*candidate* configuration that has no effect until you `commit` it.

That single difference changes how you work. It means you can make a change that would
lock you out, look at it before it takes effect, and have the device undo it for you if
you lose access. Nothing in IOS does that.

This lab makes you use that safety net deliberately — including locking yourself out on
purpose and letting the router rescue you.

## Topology

Two vJunos-router nodes, one link. That is all this lab needs.

```
   [ R1 ]  ge-0/0/0 ---- 10.0.12.0/30 ---- ge-0/0/0  [ R2 ]
   lo0: 1.1.1.1/32                          lo0: 2.2.2.2/32
```

## Addressing

| Device | Interface | Address |
|---|---|---|
| R1 | lo0.0 | 1.1.1.1/32 |
| R1 | ge-0/0/0.0 | 10.0.12.1/30 |
| R2 | lo0.0 | 2.2.2.2/32 |
| R2 | ge-0/0/0.0 | 10.0.12.2/30 |

---

## Part 0 — Boot test (do this first)

**Task 0.** Import vJunos-router into EVE-NG per the vendor's EVE-NG guide. Deploy a single
node. Boot it. Record what happens.

| Result | What it means | Do this |
|---|---|---|
| Boots to a login prompt | Nested virtualization is surviving. Continue. | Deploy the second node, go to Part 1 |
| Boots then hangs or panics | Likely the extra nesting layer | Check the host CPU flags are exposed to the EVE-NG VM, then retry once |
| Never boots | Nesting too deep for this image | Stop. See platform fallbacks in the index. |

Capture the console output either way — `output/00-boot-test.txt`. A failed boot test is
still a documented result, and knowing the platform's limits is worth writing down.

---

## Part 1 — Get oriented

**Task 1.** Log in and identify which mode you are in. Move to configuration mode and back.

```
root> configure          # operational -> configuration
[edit]
root# exit               # back to operational
```

> IOS has user EXEC, privileged EXEC, and global config. Junos has **two** modes:
> operational and configuration. There is no privileged-mode equivalent — authorisation is
> handled by the user's login class, not by a second password.

**Task 2.** View the configuration three ways and note how each differs:

```
root# show                       # hierarchical, the native format
root# show | display set         # as flat 'set' commands
root# show | compare             # candidate vs committed — empty right now
```

> `| display set` is the one to remember. It converts the hierarchy into the exact commands
> that would recreate it, which is how you copy configuration between devices and how you
> read someone else's config quickly if you think in IOS lines.

**Task 3.** Set the hostname on both routers and commit.

```
root# set system host-name R1
root# show | compare             # now shows your pending change
root# commit
```

**Stop and notice:** you ran `show | compare` *before* committing and it showed you exactly
what was about to change. There is no equivalent moment in IOS — by the time you can see
the change, it has already happened.

| Check | Command | Save to |
|---|---|---|
| Mode and version | `show version` | `output/01-show-version.txt` |
| Pending diff before commit | `show \| compare` | `output/02-compare-before-commit.txt` |
| Hostname applied | `show configuration system host-name` | `output/03-hostname.txt` |

---

## Part 2 — Interfaces and the unit concept

**Task 4.** Configure the addresses from the table:

```
root# set interfaces ge-0/0/0 unit 0 family inet address 10.0.12.1/30
root# set interfaces lo0 unit 0 family inet address 1.1.1.1/32
root# commit
```

**Task 5.** Answer in your notes before moving on: what is `unit 0`, and what is the IOS
equivalent?

> A unit is a logical interface. `ge-0/0/0` is physical; `ge-0/0/0.0` is a logical interface
> on it. The closest IOS analogue is a sub-interface — but in IOS sub-interfaces are optional
> and in Junos the unit is mandatory, so every Junos interface configuration has one even
> when there is only ever going to be one. `family inet` then says this unit carries IPv4;
> `family inet6` would carry IPv6, and a unit can carry both.

**Task 6.** Verify and confirm reachability between R1 and R2.

| Check | Command | Expected | Save to |
|---|---|---|---|
| Interfaces up | `show interfaces terse` | ge-0/0/0.0 and lo0.0 up/up with addresses | `output/04-interfaces-terse.txt` |
| Link works | `ping 10.0.12.2` from R1 | Success | `output/05-ping-p2p.txt` |
| Route table | `show route` | Direct and local routes in inet.0 | `output/06-show-route.txt` |

> `show route` output shows both a `/30` direct route and a `/32` local route per interface.
> IOS shows one connected route. The `/32` is the router's own address as a distinct entry —
> once you notice it, `show route` output stops looking cluttered.

---

## Part 3 — Rollback, and rescuing yourself on purpose

This is the part worth doing carefully. It is also the part that gives you something real
to say in an interview.

**Task 7.** Make a change, commit it, then undo it with rollback:

```
root# set system host-name WRONG-NAME
root# commit
root# rollback 1                 # load the previous committed config as candidate
root# show | compare             # confirm it will revert
root# commit
```

**Task 8.** Check how many rollbacks are available and view an older one:

```
root> show system rollback 3
root> show system commit          # commit history with timestamps and users
```

> Junos stores 50 previous committed configurations. Not a backup you configured — this is
> default behaviour. `rollback 49` is a valid command on a device you have just met.

**Task 9 — lock yourself out deliberately.** Apply a firewall filter to `ge-0/0/0` that
drops everything, using `commit confirmed`:

```
root# set firewall family inet filter LOCKOUT-TEST term deny-all then discard
root# set interfaces ge-0/0/0 unit 0 family inet filter input LOCKOUT-TEST
root# commit confirmed 2
```

Now try to ping R2 from R1. It will fail. **Do nothing for two minutes.**

The router will roll the change back by itself and connectivity will return.

> This is the mechanism the whole candidate-config model exists to enable. `commit confirmed`
> applies the change and starts a timer; if you don't type `commit` again to confirm, it
> reverts automatically. The scenario it is built for is changing the management interface
> or a filter on the link you are connected through — in IOS that is a reload-scheduling
> exercise, in Junos it is one keyword.

**Task 10.** Do it again, but this time confirm the change within the window, then remove
the filter properly with `delete`.

| Check | Command | Expected | Save to |
|---|---|---|---|
| Rollback reverted the name | `show configuration system host-name` | R1 | `output/07-rollback-result.txt` |
| Commit history | `show system commit` | Multiple entries with timestamps | `output/08-commit-history.txt` |
| Filter blocked traffic | `ping 10.0.12.2` during the window | Fails | `output/09-lockout-active.txt` |
| Auto-revert restored it | `ping 10.0.12.2` after 2 min | Succeeds | `output/10-commit-confirmed-rescue.txt` |

---

## The IOS-to-Junos gap sheet

Fill this in yourself in `output/notes.md` as you go. Do not copy it from anywhere — the
value is in writing down the ones that surprised you.

| Concept | IOS | Junos | Which surprised you? |
|---|---|---|---|
| When config applies | Immediately | On `commit` | |
| Undo a change | `no <command>` | `delete`, or `rollback n` | |
| View config | `show running-config` | `show configuration`, `\| display set` | |
| See a change before it applies | not possible | `show \| compare` | |
| Interface naming | `GigabitEthernet0/0` | `ge-0/0/0.0` | |
| Save config | `copy run start` | `commit` (no separate save) | |
| Self-rescuing change | schedule a reload | `commit confirmed` | |

## What to write down

1. What actually happened during the `commit confirmed` window — how long until connectivity
   returned, and did anything else recover with it?
2. Which single Junos behaviour would have saved you time in an IOS task you have done before?
3. Did vJunos-router boot on your platform? Record it plainly either way — the next two labs
   depend on the answer.

---

**Next:** [JNCIA-102 — Interfaces and Static Routing](../JNCIA-102-interfaces-static-routing/)
