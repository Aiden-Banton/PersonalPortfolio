# CCNA-XXX — <title>

> Template. Copy this folder to `Cisco/CCNA-XXX-slug/`, delete this line, replace every
> `<placeholder>`. Two stale folders (`configs/`, `instructions/`) may still be here from an
> earlier version — delete them. The current structure is `solutions/configs/` for configs
> and the PDF at the lab root.

| | |
|---|---|
| **Domain** | <e.g. 3.0 IP Connectivity> |
| **Covers** | <blueprint sub-topic IDs, e.g. 3.1, 3.2, 3.3> |
| **Tier** | T1 Foundational / T2 Applied / T3 Exam-pressure |
| **Platform** | Packet Tracer 8.2+ / EVE-NG / both |
| **Duration** | <mm> min |
| **Prerequisites** | <lab IDs, or none> |
| **Status** | planned / built / documented |

## What you'll build

<Two or three sentences stating the capability this proves. Written for a reader who is
studying for the exam, not for the author. No exam-anxiety framing.>

## Objectives

<3–5, each an observable outcome. "Predict which route enters the routing table when two
sources advertise the same prefix" — not "understand administrative distance".>

## Topology

<ASCII diagram of the topology goes here. Only add an SVG image reference once a topology
file actually exists in `topology/` — an image link to a file that isn't drawn yet is a
broken link on the public side. See a built lab (e.g. `CCNA-304`) for the pattern: ASCII
diagram in the README, prose noting the diagram isn't drawn yet if it isn't.>

| Device A | Interface | Device B | Interface | Link |
|---|---|---|---|---|
| | | | | |

## Addressing

<Full table for most labs. For subnetting/addressing labs (blueprint 1.6–1.9), leave the
address and mask cells blank and state the requirement the reader derives them from.>

| Device | Interface | IPv4 Address | Mask | Default Gateway | Notes |
|---|---|---|---|---|---|
| | | | | | |

## Before you start

<Simulator and version. Which file in `topology/` to open. Estimated time. Assume the
reader has nothing else — no textbook, no course login, no prior labs beyond the
prerequisites listed above.>

## Tasks

<Numbered. Each task: the goal, any constraint, and the verification that closes it.
T1 shows commands. T2 states objectives only. T3 gives the scenario only.
Individual example commands are fine. A complete paste-ready config is not — that belongs
in `solutions/`.>

1.
2.
3.

## Verification

The point of the lab. Each row is a check with an expected result. Paste real output into
`output/` and reference the filename.

| Check | Command | Expected result | Output file |
|---|---|---|---|
| | | | |

Include at least one check that proves the thing works rather than merely exists — shut an
interface and confirm failover, or send traffic the ACL should drop.

## What breaks

<3–5 specific traps, each with the symptom it produces and how to spot it in show output.
Written generally: "a common mistake here is…", not "I forgot to…". Readers find this the
most useful section.>

## How this maps to the exam

<How the topic appears on the exam — sim, multiple choice, drag-and-drop — and briefly why
it matters. Factual. Never reproduce or paraphrase an exam question.>

## Solutions

Full configurations and expected output are in [`solutions/`](./solutions/).
Work the lab first.
