# RackLabs

RackLabs is a small hardware company I started and run myself: I design physical rack enclosures for home lab and small-scale self-hosted setups — compact server racks for people who run their own infrastructure, instead of oversized data-center equipment or shelving that was never built for the job. It has a real product line and a live site at [racklabs.ca](https://www.racklabs.ca). This page is a deliberately high-level, sanitized look at what it is and, more relevant to a technical portfolio, how I use AI and agentic tooling to actually run it.

## What it is

RackLabs makes compact rack enclosures sized for the kind of hardware people run at home — a handful of nodes, a switch, maybe a small UPS — the same category of gear the HomeLab project elsewhere in this portfolio runs, just packaged as something other people can buy rather than assemble themselves out of spare parts. It's a company with a public site, an ongoing design process, and real day-to-day operations, not a class project or a concept sketch.

## Why I built it

I'm part of the same home lab and self-hosting community that the HomeLab project comes out of, and the rack hardware available to that community has always felt like an afterthought: either full-size equipment meant for a server room, or improvised shelving never designed to hold networking and compute gear safely. I wanted to build the product I kept wishing existed, rather than just write about the gap.

## Running a hardware company as a student

I'm effectively the whole company outside of class hours — design, the public site, and day-to-day operations are all things I own directly, on top of a full course load. That forces a specific kind of discipline: almost everything I set up for RackLabs has to keep working while I'm unavailable for stretches during the term, because nothing can depend on me being online to babysit it. That constraint is a big part of why the tooling described below stopped being a side experiment and became how the company actually runs.

## How I use AI and agentic tooling to build it

This is the part of RackLabs most relevant to a technical portfolio, and it's genuinely new. I run Claude Code as a working tool across the company rather than as a one-off assistant for a single task, using multi-agent setups where different agents own different responsibilities and hand work off to each other in sequence — the same pattern this portfolio repository itself uses to decide what's safe to make public and what stays private. This page is a direct example of that pattern in action: one agent's only job is to read RackLabs' internal documentation and draft a public-safe summary from it, in its own words, without touching anything sensitive; a separate, independent check then scans that draft against a strict rule set before I personally review and approve it. I didn't build that pipeline as a demo — it's the actual process this page went through before you're reading it. Treating a small, real company as a place to build and prove out agentic workflows, with real stakes if the guardrails fail, is the skill this page is meant to show.

## Scope of this page

This is intentionally a summary, not a full picture of the company. Day-to-day planning, financials, and how the business actually operates internally stay private, the same way they would at any small company that isn't ready to publish its internals. What's here is enough to show what RackLabs is and how I build it.
