# Aiden Banton: Project Portfolio

I'm Aiden Banton, a Business Information Technology (BIT) student focused on networking and infrastructure. This repository holds two things I build outside of coursework: a home lab running on real hardware, and a set of hands-on networking labs written for certification study. Both are documented as I build them, not written up after the fact.

## Projects

| Project | Description |
|---|---|
| [**HomeLab**](./HomeLab/README.md) | A self-hosted lab on real hardware: a pfSense firewall on a Protectli Vault, a VLAN-segmented network, and a 3-node Proxmox cluster. Documentation covers the build order, the addressing scheme, per-node hardware budgets, and the services running or staged on top of the cluster. |
| [**Labs**](./Labs/README.md) | Hands-on networking labs for CCNA 200-301 and JNCIA-Junos, organized by vendor. Each lab is a self-contained exercise — original topology, pre-calculated addressing table, tasks, and a verification section — written from scratch, not adapted from any course, textbook, or vendor material. |
| [**RackLabs**](./RackLabs-Overview/README.md) | A small hardware company I run myself, designing rack enclosures for home lab and self-hosted setups — [racklabs.ca](https://www.racklabs.ca). A high-level overview, plus how I use AI and agentic tooling to actually build and run it. |

```
PersonalPortfolio/
    HomeLab/           the running lab: build guides, addressing, services
    Labs/              CCNA + JNCIA practice labs, organized by vendor
    RackLabs-Overview/ small hardware company: overview + AI tooling usage
```

## How these fit together

HomeLab is infrastructure that has to keep working. Labs is where I get to break things on purpose. They're built around the same underlying concepts — VLAN design, routing, DNS/DHCP — but HomeLab runs them for real on physical gear, while Labs practices them against exam objectives, with faults injected deliberately so the troubleshooting is honest, not just the configuration.

## Where things stand

I'd rather point at the source than duplicate a table here that goes stale. [Labs/README.md](./Labs/README.md) tracks every lab's status against a `planned` / `built` / `documented` legend. As of today: 7 labs across the CCNA and JNCIA tracks are written and ready to run in Packet Tracer or EVE-NG — status `built`. One more (CCNA-301) is `planned`: indexed and scoped, but not written yet. None are `documented` yet: that status means built, run, and verified with real captured output in `output/`, and I haven't run any of them start to finish. I'd rather ship an honest table than an inflated one.

## About

Each project folder has its own README covering the overview, hardware and software used, topology, and lessons learned. These are living documents — they get updated as the build changes, not written once and left alone.

## How this repo was written

I designed and built everything documented here myself: the hardware choices, the VLAN scheme, the cluster layout, the lab scenarios, and the reasoning behind each decision. I use Claude as a documentation and drafting tool — turning my notes into structured READMEs, keeping formatting and terminology consistent across a lot of files, and sanity-checking hardware specs against public sources before I rely on them. It doesn't make the engineering calls. If a page here explains why I chose one design over another, that reasoning is mine and verified by me; the model just helped me get it written down faster and more consistently than I would have on my own.

## License

MIT — see [LICENSE](./LICENSE). Copy or adapt any of this; attribution appreciated, not required.

## Contact

Aiden Banton — [LinkedIn](https://www.linkedin.com/in/aiden-banton/)
