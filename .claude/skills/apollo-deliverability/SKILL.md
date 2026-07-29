---
name: apollo-deliverability
description: "Protect the sending asset on Apollo: mailbox health, warmup, volume limits, bounce and incident response, sending schedule. Use before launching and whenever deliverability looks off."
---

# Apollo Deliverability & Send (Infrastructure)

Keep the sending asset alive. Deliverability is the one place this library is firm rather than advisory, because a burned domain hurts the operator and does not come back quickly. Everything here protects the machine that makes outbound possible.

This level assumes the sending stack already exists. If you have no dedicated domains or mailboxes yet, build them first with `sending-infrastructure` (Infrastructure), then come back here to keep them healthy.

## When to use

- Before launching (confirm the setup is safe).
- Weekly, as a health check (see `weekly-rhythm`).
- The moment bounces spike or replies fall off a cliff (see `references/incident-response.md`).

## Preflight: check the mailboxes

Run `apollo_email_accounts_index`. For each connected sending mailbox, confirm:
- It is a **dedicated sending domain**, not the company's primary domain, and it redirects to the main site.
- Warmup is on and has been running long enough.
- Reputation and daily send state look healthy.

## The rules that keep domains alive (firm)

- **Warm up 14 days minimum, 21 optimal, before any cold send.** Warmup never turns off; it runs in parallel with campaigns forever.
- **Up to 25 cold sends per mailbox per day.** Ramp from 5 to 10/day to the ceiling over 4 to 5 weeks.
- **Verify 100% of emails before sending.** Unverified equals bounce, and bounces kill domains.
- **Bounce rate under 2%.** 2 to 4% is a warning. At 4%, pause the campaign now and investigate.
- **Plain text only.** No HTML, images, tracking pixels, or link-heavy footers for cold.
- **Send business hours, weekdays, in the prospect's timezone.** Set this with `apollo_emailer_schedules_index`.
- **Keep backup infrastructure ready.** Run double the mailboxes you need, warmed and waiting, so when something degrades you swap instead of stopping. This is the lesson that separates operators who scale from operators who stall.

## Sending schedule

Use `apollo_emailer_schedules_index` to pick or confirm a schedule that sends only in business hours on weekdays, in the target timezone. A sequence with no schedule uses the workspace default, so check it rather than assuming.

## Monitoring

Watch bounce rate and mailbox reputation continuously. Apollo's Data Health Center and inbox-placement tooling live in the UI (see `../operator-context/references/apollo-kb-map.md`); point the operator there for the deeper checks the MCP does not expose. If anything looks off, stop optimizing copy and fix deliverability first. You cannot out-write a reputation problem.

## Common mistakes

- Sending cold from the primary domain. Never.
- Launching before warmup is done to "save a week." You lose the domain instead.
- Optimizing copy while bounces are high. Deliverability gates everything.
- No backup mailboxes, so one bad week stops all sending.
