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

### Read the configured daily limit instead of assuming it

The most common cause of "the campaign is barely sending" is not a broken campaign, it is a per-mailbox daily limit left at its warmup value and never raised. It is invisible unless you look, because everything reports as active and healthy.

**`email_daily_limit` is an analytics metric**, available through the analytics report alongside the send metrics. Group it by `email_account_id` to see the configured ceiling per mailbox, or by `smart_datetime_day` to see it against actual volume:

```
metrics:  ["num_emails_sent", "email_daily_limit"]
group_by: ["email_account_id"]
```

The arithmetic is worth doing explicitly at every review: **mailboxes × daily limit = your real ceiling**, regardless of how many contacts are enrolled. A stack of 200 mailboxes still capped at 1 per day from warmup sends 200 a day, not the 5,000 the mailbox count implies. If observed volume matches a suspiciously round number, that number is almost certainly a limit rather than a coincidence.

Read this before diagnosing list size, copy, or scheduling. It is a five-second check that explains most volume complaints.

### Bounce Guard: turn it on, then tighten it

**New Apollo feature, confirmed live 2026-08-31.** Apollo now pauses a sequence automatically when its bounce rate gets too high. It is **account-wide**, applying to every active sequence, and it is configured in the UI under deliverability settings. Turn it on first, then change the numbers, because the defaults are looser than they should be.

| Setting | Apollo default | Use instead | Note |
|---|---|---|---|
| Warning threshold | 4% | **3%** | 3 is the floor. Apollo will not accept a lower value, so 2% is not available even though it would be better. |
| Auto-pause threshold | 6% | **4%** | 6% is far past the point where a domain is taking damage. |
| Minimum volume | 200 emails | 200 (fixed) | **The trap. See below.** |
| Time range | Past 7 days | Past 7 days | |

**The minimum-volume gate is the thing to understand.** Bounce Guard does not evaluate at all until the sequence has sent **200 emails inside the time window**. Below that it is inert. So the protection is absent exactly when a new sending stack is most fragile: the first few days of a ramp, a small test cohort, or a low-volume campaign that never reaches 200 in seven days may **never** trigger the guard no matter how badly it bounces.

Two rules follow:

- **A small send is not a protected send.** If you are testing with 5, 20, or 50 contacts, Bounce Guard will not save you. Verify the list independently before sending and watch the results by hand.
- **Bounce Guard is a backstop, not the control.** It catches a disaster after roughly 8 bounces in 200 sends. Verifying the list before it goes near a sequence prevents them. Treat the guard as the thing that limits the damage from a mistake you have already made.

**Not reachable from any headless lane.** Probed 2026-08-31: `bounce_guard`, `bounce_guards`, `email_bounce_guard`, `settings/bounce_guard`, `email_deliverability_settings`, and `team_settings` all return 404 on REST, and there is no MCP tool. So it cannot be read, audited, or set programmatically, and an agent cannot confirm it is on. **Check it in the UI as part of onboarding any account** and record the two thresholds in the account's own notes, because that is the only place they will be written down.
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
