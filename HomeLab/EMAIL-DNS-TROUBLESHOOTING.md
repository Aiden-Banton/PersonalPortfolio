# Multi-Provider Email on One DNS Zone: A Debugging Write-Up

This isn't a HomeLab build doc in the usual sense — no VLANs or Proxmox nodes involved —
but it's the same category of problem this repo is otherwise about: real infrastructure,
configured by hand, that broke in a way that only showed up by reading DNS records closely.
It's here because a DNS zone is a DNS zone whether it's fronting a homelab service or a
personal project's mail, and the failure mode is worth writing down.

## The setup

One domain, three services sharing a single DNS zone:

- **Inbound routing** — Cloudflare Email Routing, forwarding a handful of role addresses
  (e.g. `contact@`, `support@`) to a single real inbox, instead of paying for a full mailbox
  provider just to receive a few addresses.
- **Outbound sending** — a transactional email API (Resend) for app-generated mail, using
  its own DKIM key on a `send.` subdomain and its own SPF include.
- **Reply identity** — Gmail's "Send As" feature, so replies to those role addresses go out
  from the correct alias, routed through the outbound provider's SMTP relay with shared
  credentials, with "reply from the address it was sent to" set account-wide.

Each of these three services wants to own DNS records on the same zone: Email Routing needs
its own MX and TXT records to receive mail, the sending provider needs its own DKIM/SPF TXT
records to authenticate outbound mail, and neither is aware the other exists.

## The failure

Inbound routing worked, verified by sending and receiving a real email. Weeks later, mail to
those addresses started silently disappearing — no bounce, no error, nothing in any log
anyone was watching, because nobody was watching for a "mail just doesn't arrive" event.

The cause: setting up the outbound provider's sending records on the same zone had
overwritten the required Email Routing MX and TXT records rather than adding alongside them.
DNS providers don't warn you when one automated setup flow clobbers another provider's
records on the same zone — from the zone's point of view, a record just got replaced with a
different value. There's no conflict error, because technically there isn't a conflict; the
old value is just gone.

## Root-causing it

1. Started from the symptom (mail not arriving) rather than assuming which layer was at
   fault — checked application logs first, found nothing, which pointed at the transport
   layer rather than the app.
2. Pulled the live DNS records for the zone and diffed them mentally against what Email
   Routing's setup docs said should exist.
3. Found the required MX and TXT records for inbound routing were simply missing — not
   malformed, not conflicting, just absent.
4. Cross-referenced the timeline: the records had been present when inbound routing was
   first verified, and were gone sometime after the outbound provider's sending setup was
   run against the same zone. That timing was the strongest signal for cause, since nothing
   else had touched the zone in between.

## The fix

Re-added the missing MX and TXT records through Email Routing's own "fix missing records"
flow, which regenerates the exact values it expects without touching anything the sending
provider owns. Verified the fix the same way the original setup was verified: sent a real
email through the full path and watched it land in the inbox, rather than trusting that the
DNS panel showing green checkmarks meant the same thing as mail actually being deliverable.

## Lesson

Shared-zone DNS conflicts between independent providers fail **silently** — there's no error
surface for "another provider's automation just replaced your records." If you're running
more than one email-related service on the same zone (inbound routing, outbound sending,
alias identity, or any combination), the safe practice is:

- Pull and save the full DNS record set before running any new provider's automated setup
  against a zone that already has records on it.
- After any provider's setup flow finishes, diff the zone against the pre-setup snapshot
  instead of assuming it only added what it claimed to add.
- Verify with a real message end-to-end, not just a "verified" status in a provider
  dashboard — a green checkmark can be checking a different thing than the one you care
  about.
- If you can, keep each provider's records on distinct, non-overlapping record types or
  subdomains (e.g. DKIM on a dedicated `send.` subdomain) so automated setup for one has
  less surface to collide with another's.

This is a small case of a general infrastructure lesson: anything that lets multiple
independent systems write to the same shared configuration surface — a DNS zone, a shared
config file, a shared database schema — needs an explicit ownership boundary or a diff step,
because "it worked when I set it up" and "it still works" are different claims, and nothing
tells you when the gap between them opens up.
