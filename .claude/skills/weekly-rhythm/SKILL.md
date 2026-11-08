---
name: weekly-rhythm
description: "The weekly operating cadence for running outbound continuously: Monday deliverability, Wednesday reply sweep, Friday retrospective, plus biweekly, monthly, and quarterly reviews. Use to run outbound as an ongoing system."
---

# Weekly Rhythm (Iterate)

Having a dozen skills does not help if you do not run them on a schedule. This is the schedule. What separates a hobbyist from a real operator is not tooling, it is doing the boring cadence every week without fail.

This is a playbook for a person: put it on your own calendar and run the prescribed skill at the prescribed time. Some of the checks can also run on a schedule without anyone at the keyboard, and a few must never. Where that line sits is the last section, "Running checks unattended".

## Put it on your calendar (do this first)

Create these as recurring events, exactly:

| Event | Cadence |
|---|---|
| Monday deliverability check | Every Monday |
| Wednesday positive-reply sweep | Every Wednesday |
| Friday campaign retrospective | Every Friday |
| Inbox rotation | Every other Monday |
| Monthly spam-placement check | 1st of the month |
| Quarterly review | First Monday of the quarter |

## Monday: deliverability (15 min)

Run `apollo-deliverability` on active campaigns. Check bounce rate and mailbox health over the last 7 days. If anything is flagged, fix deliverability before touching anything else. If clean, log it and move on.

## Wednesday: positive-reply sweep (30 to 60 min)

Run `positive-reply-scoring` on everything active. Respond to interested replies within minutes. Reach referrals within a day. Read any hostile replies yourself and check for a targeting problem. If you are getting more than a handful of positive replies a week, this is where you hand them to whoever closes.

## Friday: retrospective (20 min per campaign hitting 21 days)

For each campaign that just hit its 21-day mark, pull its numbers with `campaign-reporting`, run `positive-reply-scoring`, and decide:
- **Winner** (well above your baseline): keep it, and scale it. See below.

### Scaling a winner

"Scale it" means more sends, and more sends means more mailboxes, because the per-mailbox ceiling does not move. It is fixed by deliverability, not by ambition: **25 outbound emails per day per mailbox**, and never more than 50 per day per mailbox once warmup traffic is counted.

The arithmetic is the whole job:

- **Mailboxes needed** = target sends per day ÷ 25.
- **Domains needed** = mailboxes ÷ 3 (roughly 3 mailboxes per sending domain).

So going from 100 sends a day to 400 is not a settings change, it is 12 more mailboxes across 4 more domains. And every one of them needs the full 14-day minimum warmup (21 is better) before it sends a single campaign email.

**This is why scaling has a three-week lead time, and why you start it before you need it.** The moment a campaign looks like a winner is the moment to buy the domains and start the warmup clock, not the moment to increase the daily cap on the mailboxes you already have. Pushing existing mailboxes past 25 a day to hit a number is the single fastest way to convert a winner into a domain reputation problem.

Two things that are not scaling, and get mistaken for it:

- **Adding more leads to the same mailboxes.** That changes how fast you burn the list, not how many people you reach per day. The sends per day are unchanged.
- **Shortening the wait between steps.** Same daily ceiling, more crowding, worse results.

Build the stack with `sending-infrastructure`, then bring it online with `apollo-deliverability`.
- **Middling:** iterate. Plan the next single-variable test with `experiment-design`.
- **Loser:** kill it, and write down why. The loss is a lesson if you record it.

Log the decision and the reasoning. Over a quarter these notes are what you learn from.

## Every other Monday: inbox rotation (30 min)

Review mailbox health (`apollo_email_accounts_index`). Retire failing mailboxes, promote warmed backups into rotation. If your backup pool is thin, start warming new domains now, it takes weeks.

## Monthly: spam-placement check

Run a spam-placement test (Apollo's inbox-placement tooling, see the KB map). Above 90% inbox placement, keep going. Below 80%, pause and fix before sending more.

## Quarterly: review

Read the quarter's retrospectives. Which campaigns, lists, and angles worked? Which ICP converted best? Adjust the ICP in `profile.yaml` if the data points somewhere better, and set the next quarter's experiments.

## What to skip

You do not need to check Apollo every day. Daily poking is procrastination dressed as diligence. Wait for 7-day averages, and trust the rhythm.
## Quarterly: go back to the silent majority

Every quarter, sweep everything that finished a sequence more than 60 days ago and give it one new angle. That pool is 90%+ of every list you have ever built, it is already enriched and verified, and it costs nothing to reach again. See `re-engagement`.

## Running checks unattended

A master API key changes what is possible. Sent as `X-Api-Key` to `https://mcp.apollo.io/mcp`, it removes the sign-in prompt and the session that can lapse mid-task, so an agent or a script can run Apollo on a timer with nobody watching. The CLI's stored token does the same for lanes 2 and 3. Nothing about that makes every job safe to schedule. The line is simple, and it does not move.

**The rule: nothing that spends or changes runs unattended.** Apollo marks every MCP tool with `readOnlyHint` or `destructiveHint`. An unattended job calls read-only tools and nothing else. If a tool is marked destructive, or you cannot tell, a person runs it.

**Safe to schedule (reads that report):**

| Check | What it reads | Fits |
|---|---|---|
| Account audit | Mailboxes, limits, sequences, Bounce Guard, schedules, tracker, credits (`apollo-account-audit`) | Monthly, or before onboarding a new operator |
| Campaign numbers | Sequence, step, variant, and mailbox results (`campaign-reporting`) | Friday, before the retrospective |
| Bounce watch | Bounce rate per sequence over 7 days, from contact statuses | Monday, and daily in a sequence's first two weeks |
| Mailbox health | `apollo_email_accounts_index` | Every other Monday |
| Credit balance | `apollo_usage_stats_credit_usage_stats`, per pool | Weekly |
| Stored copy check | `apollo_emailer_campaigns_step_content` on enrolled contacts | Before a person activates |
| Duplicate sample | `apollo_contacts_find_potential_duplicate` on recent contacts | After a large import |

**Never unattended:**

- **Enrolling, activating, approving, or stopping a sequence.** Activation sends real email from a real person's mailbox. Stopping looks harmless and is not: `apollo_sequences_update` deletes every step you fail to echo back, so a scheduled "pause" can wipe a live sequence. Bounce Guard, inside Apollo, is the automatic brake. A script that sees a bounce spike tells a person, who runs `apollo sequences abort`.
- **Anything that spends credits.** Enrichment, company search, waterfalls, email and phone validation, and mailbox purchases. A loop with a bug spends the month's lead credits before anyone looks.
- **Writing or merging records.** Contact creates and updates overwrite fields on a match, and a merge permanently deletes the duplicates, in the CRM too if deletion sync is on.
- **Copy.** Humans approve what sends. A scheduled job may draft and stage copy for review; it never writes it into a live sequence.
- **Apollo's agent tools.** All of them are marked destructive, and several act once approved. Approval belongs to a person in the conversation, not to a script.

**What an unattended job does with what it finds: it tells a person.** Write the result somewhere a human reads it (a message, a dashboard, a file), say what crossed a threshold and what the next step is, and stop. The 4% bounce pause, a mailbox gone inactive, a sequence active with nothing scheduled: each is a reason to alert, never a reason for the job to fix it.

**Treat the master key like a password.** It is not limited to any set of endpoints, scoped keys do not reach the MCP, and whoever holds it can send email and spend credits as the account. Keep it in a secret store or an environment variable, never in a repo, a prompt, or a log line. Give the scheduled job its own copy so you can revoke it without breaking anything else, and rotate it when someone with access leaves.

**The check before you schedule anything:** list every tool the job calls and read its annotation. If one of them is destructive, it is not an unattended job. Split it into the read that runs on the timer and the action a person takes.

