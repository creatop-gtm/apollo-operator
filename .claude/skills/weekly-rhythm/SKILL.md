---
name: weekly-rhythm
description: "The weekly operating cadence for running outbound continuously: Monday deliverability, Wednesday reply sweep, Friday retrospective, plus biweekly, monthly, and quarterly reviews. Use to run outbound as an ongoing system."
---

# Weekly Rhythm (Iterate)

Having a dozen skills does not help if you do not run them on a schedule. This is the schedule. What separates a hobbyist from a real operator is not tooling, it is doing the boring cadence every week without fail.

This is a pure playbook: no automation, no reminders on purpose. Put it on your own calendar and run the prescribed skill at the prescribed time.

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

For each campaign that just hit its 21-day mark, run `positive-reply-scoring` and decide:
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
