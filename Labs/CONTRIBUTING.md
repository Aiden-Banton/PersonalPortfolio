# Contributing

## The most useful thing you can do

**Tell me when a lab doesn't work.** A config that errors on paste, `show` output that
doesn't match what the lab claims, an addressing table with a conflict, a topology file
that won't open. These are the failures that waste someone's evening, and they're the
hardest to catch alone.

Open an issue with:

- Lab ID (e.g. `CCNA-301`)
- Simulator and version (Packet Tracer 8.2.2, EVE-NG with IOSvL2 15.2, etc.)
- What you ran and what happened
- What you expected

No template to fill in, no formatting requirements. A one-line "CCNA-301 task 4, the
floating static never installs" is genuinely useful.

## Submitting a lab

Pull requests are welcome. A lab needs to meet the same bar as the existing ones:

1. **Original work.** Nothing reproduced or adapted from Cisco Press, NetAcad, Pearson,
   Boson, or any other courseware — paid or free. Write your own scenario and topology.
2. **You've actually run it.** Configs executed, real `show` output captured in `output/`.
   A lab that was written but never run doesn't get merged.
3. **Self-contained.** Someone with only the simulator and your lab folder can complete it.
   No assumed textbook or course access.
4. **Solutions separated.** Full configs go in `solutions/`, never in the lab README.
5. **Documentation-range addressing only.** RFC 5737 (`192.0.2.0/24`, `198.51.100.0/24`,
   `203.0.113.0/24`) and RFC 3849 (`2001:db8::/32`) for anything "public". No real routable
   addresses, no credentials, no SNMP communities.

Copy `_TEMPLATE/` to start. Match the existing structure — it's what makes the collection
navigable.

## Improving an existing lab

Clearer wording, a better verification step, a trap worth adding to **What breaks** — all
welcome. If you're changing what a lab teaches rather than how it explains it, open an
issue first so we can agree on the direction before you write it.

## Scope

These labs target certification exam objectives — CCNA 200-301 now, JNCIA-Junos later.
Labs outside a cert blueprint are interesting but out of scope; a topic needs to map to a
published exam objective to belong here.

## Licence

Contributions are accepted under the MIT licence covering this repository. By submitting,
you confirm the work is yours to license.
