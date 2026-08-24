# JNCIA Lab Index

The Juniper track. Started 2026-08-04, running alongside CCNA rather than after it —
see [`../Cisco/LAB-PLAN.md`](../Cisco/LAB-PLAN.md) for the policy change and why.

**Track sequence:** JNCIA-Junos (JN0-106) fundamentals first, sequenced so the third lab
lands on the IP fabric underlay that JNCIA-DC (JN0-281) builds on. The DC exam covers
spine-leaf architecture, underlay and overlay, EVPN-VXLAN fundamentals, L2 switching, and
OSPF/IS-IS/BGP — JNCIA-103 is the deliberate hand-off point between the two.

---

## Labs

| # | Lab | Covers | Tier | Nodes | Time | Status |
|---|-----|--------|------|-------|------|--------|
| JNCIA-101 | [Junos CLI, Candidate Config, and Rollback](./JNCIA-101-junos-cli-commit-rollback/) | CLI, config model, commit/rollback, `commit confirmed` | T1 | 2 | 40–50 min | built |
| JNCIA-102 | [Routing Tables, Preference, and Routing Instances](./JNCIA-102-interfaces-static-routing/) | RIB vs FIB, route preference, static routing, virtual routers | T2 | 3 | 45–55 min | built |
| JNCIA-103 | [OSPF and eBGP: the IP Fabric Underlay](./JNCIA-103-ospf-bgp-underlay/) | OSPF, eBGP, ECMP, export policy — **bridges to JN0-281** | T2 | 4 | 60–70 min | built |

Status: `planned` → `built` (written, not run) → `documented` (run and verified, `output/`
contains real command output). None are `documented` until they have been run.

## Superseded

`Labs/Juniper/01-junos-fundamentals/` (formerly "Lab 05") predates the `JNCIA-<domain><seq>`
naming convention and was a stub with empty Steps and Verification sections. Its content is
absorbed into JNCIA-101, which covers the same objectives and actually has tasks in it.
Delete the old folder once JNCIA-101 has been run successfully.

---

## Platform

**Target:** vJunos-router on EVE-NG.

vJunos Labs images are free from Juniper for non-production use with no time limit.
**vJunos-switch is explicitly not supported on EVE-NG** because of nested-virtualization
constraints; vJunos-router has EVE-NG documentation and is the image this track targets.

### The nesting risk, stated plainly

If EVE-NG is itself running inside a VM, vJunos-router is being asked to run one layer
deeper than its EVE-NG documentation assumes. It may work. It may boot and then behave
oddly under load. It may not boot at all.

**JNCIA-101 Part 0 is a single-node boot test and it exists precisely for this.** Run it
before investing time in the other two labs. A failed boot test is a legitimate documented
result — record it in `output/00-boot-test.txt` either way.

### Platform fallbacks

If vJunos-router will not run on the nested setup, in preference order:

| Option | What it gives | Cost |
|---|---|---|
| **containerlab on a dedicated VM** | cRPD / cJunosEvolved in containers — no nesting problem, far lighter, and the toolchain Juniper's own DC material uses | Labs need rewriting against containerlab topology files rather than EVE-NG |
| **Juniper vLabs** | Free reservable hosted sandboxes with real Junos devices | Time-limited sessions, fixed topologies, weak for portfolio artifacts since you don't control the environment |
| **Bare-metal EVE-NG** | Removes the nesting layer entirely | Needs a spare machine; loses the convenience of running inside the existing cluster |

containerlab is the strongest fallback and arguably the better primary choice for the DC
track specifically — spine-leaf fabrics with eight or more nodes are routine in containerlab
and painful in any nested EVE-NG.

## RAM

vJunos-router is not a light image. Node counts climb across the track — 2, then 3, then 4 —
and JNCIA-103 is where a memory ceiling will show up if there is one. If three nodes are
already tight, run JNCIA-103's OSPF half on three nodes and defer the eBGP half rather than
fighting it mid-lab.

## Sourcing

vJunos images come from Juniper's support downloads and require a Juniper account. The
files stay local — never in this repo, never in a public one. Record what's imported in
[`../EVE-NG-Images/README.md`](../EVE-NG-Images/README.md).
