# CCNA Lab Index

Generated 2026-07-30. This is a scoped first pass covering OCG Vol 1 Ch 1–3
(the chapters just scored) — not the full 25-lab plan across all six domains.
2 labs, ~1.5 hours total.

## Coverage audit — Ch1–3 sub-topics only

| Sub-topic | Labbed by | Notes |
|---|---|---|
| 1.1 Role/function of network components | NOT LABBED | Definitional — better served by drill-coach flashcards than a topology build |
| 1.2 Network topology architectures | NOT LABBED | Conceptual (2-tier/3-tier/spine-leaf/WAN/SOHO comparison) — drill-coach discrimination drill |
| 1.3 Physical interface and cabling types | CCNA-101 | Rides along in the switching lab rather than a dedicated build |
| 1.4 Interface and cable issues | CCNA-101 | Verified via duplex/speed mismatch task |
| 1.5 TCP vs UDP | NOT LABBED | Blueprint's own guidance: conceptual, not lab material — drill-coach discrimination drill |
| 1.13 Switching concepts | CCNA-101 | Dedicated — this is the only hands-on-heavy topic in Ch1–3 |
| 3.1 Routing table components | CCNA-301 | Dedicated |
| 3.2 Forwarding decision | CCNA-301 | Dedicated |
| 3.3 Static routing | CCNA-301 | Dedicated |

Full single-area OSPFv2 (3.4) is OCG Ch 20–24, not Ch 1–3 — out of scope for this pass,
already planned for a later lab once that material is studied.

## Recommended order

1. **CCNA-101** — Ethernet Switching Fundamentals
2. **CCNA-301** — Reading the Routing Table: Static Routes, AD and Longest Match (requires CCNA-101)

## Labs

### CCNA-101 — Ethernet Switching Fundamentals

- **id:** CCNA-101
- **slug:** ethernet-switching-fundamentals
- **domain:** 1.0 Network Fundamentals
- **covers:** 1.3, 1.4, 1.13
- **tier:** T1 Foundational
- **platform:** PT
- **duration:** 30 min
- **requires:** none
- **skippable:** no
- **devices:** SW1, PC1, PC2, PC3
- **addressing-lab:** no
- **objective:** Build a single switch with three hosts, observe MAC address table
  population as frames are switched, and diagnose an injected duplex mismatch.
- **exam-relevance:** MAC learning/aging and flooding-vs-forwarding behaviour is a
  reliable multiple-choice source; duplex/speed mismatch symptoms show up in
  troubleshooting-style questions.
- **status:** built

### CCNA-301 — Reading the Routing Table: Static Routes, AD and Longest Match

- **id:** CCNA-301
- **slug:** reading-the-routing-table
- **domain:** 3.0 IP Connectivity
- **covers:** 3.1, 3.2, 3.3
- **tier:** T1 Foundational
- **platform:** PT+EVE
- **duration:** 45 min
- **requires:** CCNA-101
- **skippable:** no
- **devices:** R1, R2, R3, PC1, PC2
- **addressing-lab:** no
- **objective:** Build a three-router topology, install a mix of static, default and
  floating static routes, and read `show ip route` to predict which route wins on
  administrative distance and longest match.
- **exam-relevance:** Sim-style tasks routinely ask for a default route plus a backup
  path; AD comparison and longest-match forwarding decisions are high-frequency
  multiple-choice topics.
- **status:** planned

## Not labbed

- **1.1, 1.2, 1.5** — genuinely conceptual/definitional sub-topics with no configuration
  surface. Covered by `drill-coach` flashcards and discrimination drills instead of a
  topology build. This matches the blueprint's own note that some sub-topics are better
  served by drills than labs.
