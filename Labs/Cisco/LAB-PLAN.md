# CCNA Lab Plan — full chapter + capstone allocation

> ## ▶ P2 REVERSED — LABS RESUME (2026-08-04)
>
> **Policy P2 (below) is overturned. Lab building restarts now, selectively.**
>
> **Why.** P2 optimised for exam readiness and treated labs as post-exam portfolio work.
> That trade looked right in July. Two things changed it:
>
> 1. **The portfolio has zero `documented` labs.** A career scan on 2026-08-04 found 20
>    projects and exactly one resume-eligible item. Every claim in the portfolio currently
>    rests on prose Aiden wrote about his own work — there is not one captured `show`
>    command in the repo. That is the single credibility gap a reviewer cannot wave away,
>    and it does not close by itself while the labs wait for exam day.
> 2. **The strongest lab candidates are also the weakest exam domains.** The three labs
>    resuming are targeted at the clusters `mastery-ledger.md` actually flags — wireless
>    (4 of 11 boson-01 misses plus a fifth in exsim-domain1-01), OSPF 3.4 (two independent
>    weak signals; the ledger's own note says "worth a lab"), and IPv6 1.8/1.9 (entirely
>    unmeasured, highest-weight domain). Lab time and remediation time are the same time.
>
> P2's own mitigation clause — Packet Tracer walkthroughs inside the drills — remains valid
> and is unaffected. This reversal is narrow: it authorises the three labs listed below plus
> the Juniper track, not the full 56-lab plan. The rest of this file stays a backlog.
>
> **Resuming now:** CCNA-304 (3.4 OSPFv2), CCNA-207 (2.7–2.9 WLAN/WLC), CCNA-108 (1.8–1.9 IPv6).
> **Also resuming:** the Juniper track — see [`../Juniper/LAB-INDEX.md`](../Juniper/LAB-INDEX.md).
> This additionally supersedes the "Do not build Juniper labs yet" line in
> `_study/reference/conventions.md`, on Aiden's decision of 2026-08-04 to run JN0-106
> fundamentals now and ladder into JNCIA-DC (JN0-281) afterwards.
>
> ---
>
> ### Superseded text, kept for the record (policy P2, set 2026-07-30)
>
> **Do not build any lab in this file before exam day.** The whole lab track is now
> post-exam portfolio work: once Aiden sits the exam, these get built and prepped for the
> public repo. Until then this document is a backlog, not a schedule.
>
> The weekly batch cadence described further down is **superseded** — there are no Batch 1
> through Batch 5 weeks any more. `ROADMAP.md` is an urgency queue with no lab track in it.
>
> What this costs, stated plainly: config-heavy sub-topics (3.4 OSPF, 2.1–2.4 switching,
> 4.3/4.6 DHCP, 2.7–2.9 WLC, 5.10 WPA2) lose their hands-on reps before the exam.
> The mitigation is a Packet Tracer walkthrough section inside the matching drill in
> `Certifications/CCNA/_study/drills/` — see `DRILL-3.4-single-area-ospfv2.md`, which
> exists specifically because the OSPF lab is no longer being built in time.
>
> Policy source: `Certifications/CCNA/_study/reference/ccna-blueprint.md`.

Generated 2026-07-30. This is the **plan**, not the index. `LAB-INDEX.md` holds labs that
actually exist; this file holds every lab that *should* exist and the order they get built.

**Scope:** one lab per OCG chapter, plus one or two capstone labs per exam domain.

| | Count |
|---|---|
| Chapter labs | 47 |
| Capstone labs (1–2 per domain) | 9 |
| **Total** | **56** |

47 chapter labs against 44 chapters: three chapters that straddle two domains get a lab in
each (e.g. V2 Ch13 → CCNA-203 for WLAN config and CCNA-501 for WLAN security). That is
deliberate — a single lab covering both would map to two domains at once and break the
coverage audit.

Two labs already exist and keep their IDs: **CCNA-101** (built, awaiting `.pkt`) and
**CCNA-301** (planned). At 3–4 labs/day the remaining 54 are roughly two weeks of build time.

---

## Read this before building anything

**Chapter numbers here come from the chapter→domain map in `_study/reference/ccna-blueprint.md`.
Chapter _titles_ do not — they are working titles derived from the sub-topics each range
covers.** The blueprint records ranges (`Vol 1 Ch 11–15 → 1.6, 1.7`), not per-chapter
titles, and the ledger's chapter references don't match any OCG edition cleanly enough to
infer the rest safely.

So: **the first action of every batch is to confirm that batch's chapter titles against
Aiden's actual table of contents.** `lab-batch` does this and will not proceed without it.
A lab named after the wrong chapter is worse than no lab — it silently breaks the mapping
between drill scores, ledger rows and lab coverage, which is the whole point of the system.

Working titles are marked `~` below. Confirmed titles lose the tilde.

---

## ID scheme

Unchanged from `conventions.md`: `CCNA-<domain><seq>`.

| Range | Meaning |
|-------|---------|
| `CCNA-<d>01` … `<d>89` | Chapter labs, sequential within the domain |
| `CCNA-<d>90` … `<d>99` | **Capstone labs** — multi-chapter, domain-level integration |

So `CCNA-104` is the fourth Domain 1 chapter lab; `CCNA-190` is Domain 1's first capstone.
Existing `CCNA-101` and `CCNA-301` keep their IDs.

## Tiers

| Tier | Meaning | Typical duration |
|------|---------|------------------|
| T0 | Concept-only — no topology. Lab is a guided reading + prediction exercise. | 15 min |
| T1 | Foundational — single device or simple pair. | 30 min |
| T2 | Standard — multi-device, one protocol. | 45–60 min |
| T3 | Capstone — multi-protocol integration, fault injection, verification matrix. | 90–120 min |

Not every chapter earns a topology. Chapters covering purely definitional sub-topics
(1.1, 1.2, 1.5, 5.1, 5.2, 6.1, 6.4) get **T0** labs — a prediction-and-justification
exercise, not a build. Do not manufacture a topology to justify a lab; say T0 and move on.
The blueprint explicitly endorses this for conceptual sub-topics.

---

## Batch schedule

One batch per week, ordered by **exam impact**, not by book order. Batch 1 leads with the
open `Remediate` clusters from the mastery ledger, because those are the labs that change
the exam outcome.

| Batch | Week | Domain / focus | Chapter labs | Capstones | Why this order |
|-------|------|----------------|--------------|-----------|----------------|
| **1** | 30 Jul – 5 Aug | **Wireless + Automation** (D2/D5 wireless, D6) | Vol 2 Ch 11–13, Ch 14–17 | CCNA-290, CCNA-690 | Both are open `Remediate` clusters. Highest impact available. |
| **2** | 6–12 Aug | **D1 Network Fundamentals** | Vol 1 Ch 1–4, Ch 11–15, Ch 25–27 | CCNA-190, CCNA-191 | Ch 14 subnetting is the third open `Remediate`. IPv6 (Ch 25–27) is entirely unmeasured. |
| **3** | 13–19 Aug | **D3 IP Connectivity** | Vol 1 Ch 16–19, Ch 20–24 | CCNA-390, CCNA-391 | Heaviest domain (25%). OSPF has two independent weak signals. |
| **4** | 20–26 Aug | **D2 Network Access** (switching) | Vol 1 Ch 5–10 | CCNA-291 | Strongest existing material — lowest risk, so it waits. |
| **5** | 27 Aug – 2 Sep | **D4 + D5** | Vol 2 Ch 1–3, Ch 4–7, Ch 8–10 | CCNA-490, CCNA-590 | Both already 87%+ on the mixed set. Consolidation, not remediation. |

Exam is early September (**date not yet confirmed — see `ROADMAP.md`**). Batches 1–3 are
the ones that matter for it; 4 and 5 are portfolio consolidation and can slip past the exam
without harm.

Batch dates assume the exam holds at early September. If it moves again, re-sequence by
impact rather than shifting every batch uniformly — the ordering above is driven by open
`Remediate` clusters, not by the calendar.

---

## Allocation

Sub-topics are from `ccna-blueprint.md`. `covers` must be exhaustive across all 53 labs or
the sub-topic goes in the not-labbed list with a justification.

### Batch 1 — Wireless (Vol 2 Ch 11–13) · domains 2.0, 5.0

| ID | Chapter | ~Working title | Covers | Tier |
|----|---------|----------------|--------|------|
| CCNA-201 | V2 Ch11 | ~Wireless fundamentals and RF | 1.11 | T0 |
| CCNA-202 | V2 Ch12 | ~Cisco wireless architectures and AP modes | 2.6, 2.7 | T2 |
| CCNA-203 | V2 Ch13 | ~WLC management access and WLAN creation | 2.8, 2.9 | T2 |
| CCNA-501 | V2 Ch13 | ~Wireless security — WPA2/WPA3 and PSK config | 5.9, 5.10 | T2 |
| **CCNA-290** | capstone | Build a secured WLAN end to end | 2.6–2.9, 5.9, 5.10, 1.11 | T3 |

> This batch targets boson-01 misses #7 (WLC ports), #8 (WLAN L2 security), #50 (802.11 MAC
> frames) and #52 (WPA3) directly. `DRILL-1.11-2.6-2.9-wireless-fundamentals.md` should be
> run *before* CCNA-202 to get a cold baseline.

### Batch 1 — Automation (Vol 2 Ch 14–17) · domain 6.0

| ID | Chapter | ~Working title | Covers | Tier |
|----|---------|----------------|--------|------|
| CCNA-601 | V2 Ch14 | ~How automation changes network management | 6.1 | T0 |
| CCNA-602 | V2 Ch15 | ~Controller-based vs traditional networking | 6.2, 6.3 | T1 |
| CCNA-603 | V2 Ch16 | ~REST APIs and JSON interpretation | 6.5, 6.7 | T1 |
| CCNA-604 | V2 Ch17 | ~Config management — Ansible and Terraform | 6.6 | T0 |
| **CCNA-690** | capstone | Read a controller's northbound API and predict the config | 6.2, 6.3, 6.5, 6.7 | T2 |

> 6.4 (AI/ML in netops) is **not labbed** — purely conceptual, no configuration surface.
> Drill only. Targets misses #10 and #65.

### Batch 2 — Network Fundamentals (Vol 1 Ch 1–4, 11–15, 25–27) · domain 1.0

| ID | Chapter | ~Working title | Covers | Tier |
|----|---------|----------------|--------|------|
| CCNA-102 | V1 Ch1 | ~TCP/IP model and encapsulation ordering | 1.1, 1.5 | T0 |
| CCNA-103 | V1 Ch2 | ~Ethernet LAN fundamentals and cabling | 1.3, 1.4 | T1 |
| CCNA-104 | V1 Ch3 | ~WAN and IP routing fundamentals | 1.2, 3.1 | T1 |
| CCNA-105 | V1 Ch4 | ~Cisco CLI navigation and device basics | 1.10 | T1 |
| CCNA-106 | V1 Ch11 | ~Perspectives on IPv4 subnetting | 1.6 | T0 |
| CCNA-107 | V1 Ch12 | ~Analyzing classful IPv4 networks | 1.7 | T1 |
| CCNA-108 | V1 Ch13 | ~Analyzing subnet masks | 1.6 | T1 |
| CCNA-109 | V1 Ch14 | ~Analyzing existing subnets | 1.6 | T2 |
| CCNA-110 | V1 Ch15 | ~Subnet design and VLSM | 1.6 | T2 |
| CCNA-111 | V1 Ch25 | ~IPv6 fundamentals and address types | 1.8, 1.9 | T1 |
| CCNA-112 | V1 Ch26 | ~IPv6 addressing and subnetting | 1.8, 1.9 | T2 |
| CCNA-113 | V1 Ch27 | ~Implementing IPv6 on routers | 1.8, 1.9 | T2 |
| **CCNA-190** | capstone | Design and verify a complete VLSM addressing plan | 1.6, 1.7, 1.10 | T3 |
| **CCNA-191** | capstone | Dual-stack IPv4/IPv6 site build | 1.8, 1.9, 1.10, 3.1 | T3 |

> `CCNA-101` (Ethernet Switching Fundamentals, covers 1.3/1.4/1.13) already exists and
> stays — CCNA-103 must not duplicate it. Check `LAB-INDEX.md` before writing.
> **CCNA-109 and CCNA-110 are addressing labs** — per `conventions.md`, their addressing
> tables ship with blank cells for the student to fill in.
> 1.12 (virtualization fundamentals) is **not labbed** — conceptual, drill only.

### Batch 3 — IP Connectivity (Vol 1 Ch 16–24) · domain 3.0

| ID | Chapter | ~Working title | Covers | Tier |
|----|---------|----------------|--------|------|
| CCNA-302 | V1 Ch16 | ~Operating Cisco routers | 3.1 | T1 |
| CCNA-303 | V1 Ch17 | ~Configuring IPv4 addresses and static routes | 3.3 | T2 |
| CCNA-304 | V1 Ch18 | ~IP routing in the LAN — SVIs and router-on-a-stick | 2.1, 3.1 | T2 |
| CCNA-305 | V1 Ch19 | ~Troubleshooting IPv4 routing | 3.1, 3.2 | T2 |
| CCNA-306 | V1 Ch20 | ~OSPF concepts — LSAs, areas, DR/BDR | 3.4 | T0 |
| CCNA-307 | V1 Ch21 | ~Implementing single-area OSPFv2 | 3.4 | T2 |
| CCNA-308 | V1 Ch22 | ~OSPF network types and interface config | 3.4 | T2 |
| CCNA-309 | V1 Ch23 | ~OSPF neighbor troubleshooting | 3.4 | T2 |
| CCNA-310 | V1 Ch24 | ~OSPF route selection and metrics | 3.4, 3.2 | T2 |
| **CCNA-390** | capstone | Multi-router OSPF with failover and reconvergence | 3.1–3.4 | T3 |
| **CCNA-391** | capstone | FHRP and gateway redundancy | 3.5, 3.1 | T3 |

> `CCNA-301` (routing table / static routes / AD / longest match) already exists as
> `planned` — build it in this batch. 3.5 (FHRP) has no chapter of its own in the
> blueprint's map; CCNA-391 is where it lands.

### Batch 4 — Network Access switching (Vol 1 Ch 5–10) · domain 2.0

| ID | Chapter | ~Working title | Covers | Tier |
|----|---------|----------------|--------|------|
| CCNA-204 | V1 Ch5 | ~Analyzing Ethernet LAN switching | 1.13 | T1 |
| CCNA-205 | V1 Ch6 | ~Basic switch management and access | 2.1 | T1 |
| CCNA-206 | V1 Ch7 | ~Switch interfaces — speed, duplex, errors | 1.4 | T1 |
| CCNA-207 | V1 Ch8 | ~VLANs and 802.1Q trunking | 2.1, 2.2 | T2 |
| CCNA-208 | V1 Ch9 | ~Spanning Tree Protocol concepts | 2.5 | T0 |
| CCNA-209 | V1 Ch10 | ~RSTP and EtherChannel configuration | 2.4, 2.5 | T2 |
| **CCNA-291** | capstone | Two-switch campus — VLANs, trunks, RSTP root election, LACP | 2.1–2.5 | T3 |

> 2.3 (CDP/LLDP) rides along in CCNA-207. Targets boson-01 misses #4 and #22.

### Batch 5 — IP Services + Security (Vol 2 Ch 1–10) · domains 4.0, 5.0

| ID | Chapter | ~Working title | Covers | Tier |
|----|---------|----------------|--------|------|
| CCNA-502 | V2 Ch1 | ~TCP/IP transport and applications | 1.5, 5.1 | T0 |
| CCNA-503 | V2 Ch2 | ~Basic IPv4 access control lists | 5.6 | T2 |
| CCNA-504 | V2 Ch3 | ~Advanced and named IPv4 ACLs | 5.6 | T2 |
| CCNA-505 | V2 Ch4 | ~Security architectures and threat concepts | 5.1, 5.2 | T0 |
| CCNA-506 | V2 Ch5 | ~Securing network devices — passwords, SSH, AAA | 5.3, 5.4, 5.8, 4.8 | T2 |
| CCNA-507 | V2 Ch6 | ~Port security | 5.7 | T2 |
| CCNA-508 | V2 Ch7 | ~DHCP snooping and dynamic ARP inspection | 5.7 | T2 |
| CCNA-401 | V2 Ch8 | ~DHCP server, client and relay | 4.3, 4.6 | T2 |
| CCNA-402 | V2 Ch9 | ~Device management protocols — NTP, syslog, SNMP | 4.2, 4.4, 4.5 | T2 |
| CCNA-403 | V2 Ch10 | ~NAT — static, dynamic and PAT | 4.1 | T2 |
| **CCNA-490** | capstone | Edge services build — NAT, DHCP relay, NTP, syslog, SSH | 4.1–4.6, 4.8, 4.9 | T3 |
| **CCNA-590** | capstone | Layer 2 hardening — ACLs, port security, DHCP snooping, DAI | 5.3–5.8 | T3 |

> 4.7 (QoS/PHB) has no dedicated lab — minimal config surface in Packet Tracer. **Drill
> only**, and flag it: it is the VoIP-adjacent sub-topic Aiden self-reports as weak and it
> is still entirely unmeasured. 5.5 (VPNs) is conceptual at CCNA level — drill only.

---

## Not labbed — with justification

Every one of these needs a `drill-coach` drill instead. A sub-topic that is neither labbed
nor drilled is a coverage hole, and `final-review` will flag it as one.

| Sub-topic | Why not labbed |
|-----------|----------------|
| 1.1 Network component roles | Definitional. Flashcards. |
| 1.2 Topology architectures | Comparative/conceptual. Discrimination drill. |
| 1.5 TCP vs UDP | Conceptual. Discrimination drill. |
| 1.12 Virtualization fundamentals | No CCNA-level config surface. |
| 4.7 QoS / PHB | Minimal PT config surface. **Unmeasured and self-reported weak — drill is mandatory, not optional.** |
| 5.1, 5.2 Security concepts / program elements | Definitional. |
| 5.5 VPNs | Conceptual at CCNA level. |
| 6.1 Automation impact | Conceptual. |
| 6.4 AI/ML in netops | Conceptual. |

---

## Coverage audit

All 56 labs plus the not-labbed list must account for every one of the 53 blueprint
sub-topics. `lab-batch` re-runs this audit at the end of each batch and reports any
sub-topic that is neither in a lab's `covers` list nor in the table above.

The tracker's To Do box reads this file directly: any ID here without an entry in
`LAB-INDEX.md` shows as **not started**. So adding a row here is how a lab enters the
queue, and there is no separate checklist to keep in sync.
