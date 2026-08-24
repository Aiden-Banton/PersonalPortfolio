# Networking Labs

Hands-on networking labs for certification study, written to be worked through rather than
read. Each lab is a self-contained exercise: a topology, a pre-calculated addressing table,
tasks, and a verification section that proves the configuration actually does what it claims.

Built while studying for CCNA 200-301, and published because the labs I wanted while
studying mostly didn't exist in a form I could just open and run.

**Status: early.** The index below is the full plan; most of it isn't written yet. Labs
marked `documented` have been built and run end to end. Anything else is scaffolding.

---

## How to use these

1. Pick a lab from the tables below
2. Open `topology/` — there's a `.pkt` (Packet Tracer) or `.unl` (EVE-NG) file to load
3. Work through `README.md`, or print the PDF and work from paper
4. Check yourself against `solutions/` when you're done, not before

Each lab folder:

```
CCNA-XXX-slug/
├── README.md         the lab — scenario, topology, addressing, tasks, verification
├── CCNA-XXX.pdf      printable version, no answers
├── addressing.md     the addressing table
├── topology/         diagram + the runnable .pkt / .unl
├── solutions/        full configs and expected output — look after, not before
└── output/           real show output from a verified run
```

**Requirements:** Packet Tracer 8.2+ for most labs, EVE-NG for a few that need IOS features
Packet Tracer doesn't implement. Each lab states which it needs. No textbook, course login,
or paid resource is required for any lab.

## Difficulty tiers

| Tier | What it means |
|------|---------------|
| **T1 Foundational** | Guided. Commands shown. ~30 min |
| **T2 Applied** | Objectives given, you find the commands. ~45–60 min |
| **T3 Exam-pressure** | Scenario only, time-boxed, with deliberate faults to troubleshoot. ~30 min |

---

## Cisco — CCNA 200-301

Full plan in [`Cisco/LAB-INDEX.md`](./Cisco/LAB-INDEX.md). 25 labs allocated across the six
exam domains in proportion to their exam weight.

| # | Lab | Domain | Tier | Platform | Status |
|---|-----|--------|------|----------|--------|
| CCNA-101 | [Ethernet Switching Fundamentals](./Cisco/CCNA-101-ethernet-switching-fundamentals/) | 1.0 | T1 | PT | built |
| CCNA-108 | [IPv6 Addressing and Connectivity](./Cisco/CCNA-108-ipv6-addressing-connectivity/) | 1.0 | T2 | PT | built |
| CCNA-207 | [WLAN Deployment: WLC, SSIDs and the Management/Client Split](./Cisco/CCNA-207-wlan-wlc-deployment/) | 2.0 | T2 | PT | built |
| CCNA-301 | Reading the Routing Table: Static Routes, AD and Longest Match | 3.0 | T1 | PT+EVE | planned |
| CCNA-304 | [Single-Area OSPFv2: Adjacency, Cost and Stuck Neighbours](./Cisco/CCNA-304-single-area-ospfv2/) | 3.0 | T2 | PT | built |

**Coverage:** 0 of 6 domains have `documented` labs. Four labs are `built` — written and
ready to run, but not yet executed with output captured. The distinction is the whole point
of the status column. See [`Cisco/LAB-INDEX.md`](./Cisco/LAB-INDEX.md).

## Juniper — JNCIA-Junos → JNCIA-DC

Started 2026-08-04. JN0-106 fundamentals first, sequenced so JNCIA-103 lands on the IP
fabric underlay that JNCIA-DC (JN0-281) builds on. Full detail, including the vJunos
platform constraints, in [`Juniper/LAB-INDEX.md`](./Juniper/LAB-INDEX.md).

| # | Lab | Domain | Tier | Platform | Status |
|---|-----|--------|------|----------|--------|
| JNCIA-101 | [Junos CLI, Candidate Config and Rollback](./Juniper/JNCIA-101-junos-cli-commit-rollback/) | Junos fundamentals | T1 | EVE (vJunos-router) | built |
| JNCIA-102 | [Routing Tables, Preference and Routing Instances](./Juniper/JNCIA-102-interfaces-static-routing/) | Routing fundamentals | T2 | EVE (vJunos-router) | built |
| JNCIA-103 | [OSPF and eBGP: the IP Fabric Underlay](./Juniper/JNCIA-103-ospf-bgp-underlay/) | OSPF, BGP → DC underlay | T2 | EVE (vJunos-router) | built |

---

## Status values

| Status | Means |
|--------|-------|
| `planned` | In the index, not written yet |
| `built` | Written, not yet run end to end |
| `documented` | Built, run, and verified — `output/` contains real command output |

Only `documented` labs have been proven to work. The distinction is deliberate: a lab that
looks authoritative but was never executed is worse than no lab at all.

---

## Originality

Every lab here is original work. Scenarios, topologies, addressing plans and task sequences
are written from scratch. Nothing is reproduced or adapted from Cisco Press, Cisco NetAcad,
Pearson Test Prep, Boson, Jeremy's IT Lab, or any other commercial or free courseware.

Exam objective *topics* are referenced by their official identifiers (e.g. "covers 3.4,
single-area OSPFv2") because those are facts about the exam. No exam questions are
reproduced, paraphrased, or hinted at.

Command syntax and `show` output formats are factual and appear as they do on real devices.

## Contributing

Found a lab that doesn't work, or output that doesn't match? Open an issue — that's the most
useful thing you can do. See [`CONTRIBUTING.md`](./CONTRIBUTING.md).

## Licence

MIT — see [`../LICENSE`](../LICENSE). Use these however you like, including in teaching.
Attribution appreciated but not required.
