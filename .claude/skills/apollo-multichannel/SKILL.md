---
name: apollo-multichannel
description: "Optional layer on top of email: add LinkedIn, call, and manual steps to an Apollo sequence and work the task queue they create. Apollo automates email only; the rest are human tasks. Use to run multichannel outbound on Apollo."
---

# Apollo Multichannel (Message, optional)

Email is the automated core of this library. This skill is the optional layer on top: add LinkedIn, call, and manual steps to a sequence, and work the task queue those steps create. It is opt-in on two levels. You do not have to add multichannel steps to a sequence, and you do not have to use this skill at all. A clean email-only sequence is a complete, valid campaign.

## The one thing to be honest about

**Apollo automates email only.** Every non-email step is a **task for a human to do**, not something Apollo performs. A `linkedin_step_message` does not send a LinkedIn message; it drops a task in the rep's queue that says "send this message." A `call` step queues a call task. So multichannel here means "automated email plus a structured human task queue for LinkedIn and calls." Never tell the operator Apollo will send the LinkedIn touch or place the call. It queues the work; a person does it.

This is a feature, not a limitation, and it is the safer way to run LinkedIn. Because a person executes each LinkedIn touch from their own browser and logged-in profile, the action looks like normal human activity, not an API acting on your account while you are away. Tools that fully automate LinkedIn through the API carry real account-ban risk. The tradeoff is that the safety now depends on the human pacing themselves, because Apollo does not enforce a daily limit (see below).

Note: call and LinkedIn steps require an Apollo plan with task functions, so they may return a plan error. If they do, that capability is gated on your plan, and email-only still works.

## Part A: add multichannel steps to a sequence

Multichannel steps go in the same `apollo_sequences_create` call as the email steps (see `../apollo-sequence-builder`), you just use more step types. The step types:

| Step type | What it is | Needs content? |
|---|---|---|
| `auto_email` | Automated email (the core) | Yes (`emailer_touches`) |
| `manual_email` | Email the rep reviews and sends | Yes (`emailer_touches`) |
| `linkedin_step_view_profile` | Soft touch, just view the profile | No |
| `linkedin_step_connect` | Send a connection request | Yes (short note) |
| `linkedin_step_message` | Message a 1st-degree connection | Yes (message body) |
| `linkedin_step_interact_post` | Like or comment on a recent post | No |
| `call` | Call task, with a script in `note` and a `priority` | No touch; use `note` |
| `action_item` | Generic manual to-do | No |

A sensible, non-annoying default cadence (relative `wait_time`, so Day 0/2/3/5/8):
1. `auto_email`, wait 0
2. `linkedin_step_view_profile`, wait 2 (warm the name before the next email)
3. `auto_email`, wait 1
4. `linkedin_step_message`, wait 2 (only fires if they connected; write it like a human)
5. `call`, wait 3, `priority: high`, `note` = a two-line call script

Rules that carry over: still build the sequence `active: false`, still let a human activate it, still keep the email copy to the Message standard. LinkedIn and call steps do not change any of that.

## LinkedIn safety: you set the daily cap, Apollo does not

Apollo queues LinkedIn tasks but does not throttle them. When a batch of `linkedin_step_connect` tasks comes due on the same day and you clear the whole queue in one sitting, you can fire dozens of connection requests in a few minutes, and that burst is exactly the pattern LinkedIn's automated defenses flag. Unlike dedicated LinkedIn tools, Apollo has no built-in cap to stop you, so the pacing is on the human working the queue.

The rule we run by, from experience:

- **Keep connection requests under 30 per day. 20 is the safer number, and the one to default to**, especially on a newer account or one that has not sent invites at volume before. A well-established account can sit near 30; a fresh one should stay well under 20 and ramp up slowly over a few weeks, the same gradual-ramp logic as email warmup.
- **Spread the work across multiple days, never clear the whole LinkedIn queue in one day.** This is the opposite of how you work calls or email. A steady trickle (a handful of connects a day, most days) reads as a real person using LinkedIn. Sitting down once and firing off a week's worth of invites in an afternoon is exactly the burst pattern LinkedIn's defenses look for, even if the total is modest. Consistency over days beats volume in a session.
- **The cap is on new connection requests specifically.** Profile views, post interactions, and messages to people you are already connected to are lower risk, but do not turn those into their own blast either.

Practically, this means working `linkedin_step_connect` tasks as a small daily habit: sort the queue, clear only up to your daily number, and leave the rest for the following days rather than pushing to zero. If the connect tasks are piling up faster than 20 to 30 a day can clear them, the sequence is enrolling LinkedIn touches faster than a human can safely execute them, so slow the enrollment, do not raise the cap or binge the backlog.

## Part B: work the task queue

The steps above (and any standalone tasks) create tasks a person clears each day. The tools:

- **See what is due:** `apollo_tasks_search` with no status filter returns open (scheduled) tasks. Sort `task_due_at` ascending for soonest-first. Filter by `task_type_cds` (for example the four `linkedin_step_*` values for a LinkedIn block, or `phone_call` for a call block), by `priority`, by `date_range`, or by `emailer_campaign_id` to work one sequence's tasks.
- **Clear them:** `apollo_tasks_complete` when done, `apollo_tasks_skip` when skipping, `apollo_tasks_update` to reschedule or re-prioritize.
- **Create work in bulk:** `apollo_tasks_bulk_create` to drop a task on a whole segment, for example a `call` task on every High-priority lead who opened three times. Each task needs `user_id` (the assignee), `type`, and at least one association (`contact_id`, `account_id`, or `opportunity_id`). Put call scripts in `note`; put LinkedIn or email content in `standalone_outreach_task_message`.

A good habit: batch by channel so you are not context-switching per lead, which is what makes multichannel feel like a slog. One caveat, and it matters: calls and manual emails are fine to work in a single focused sitting, but LinkedIn connection requests are not. Pace those across days per the cap above, never clear the whole LinkedIn queue in one session. So batch calls, but trickle LinkedIn. This slots into `../weekly-rhythm`.

## Part C: one-off emails (sending to a single person)

Not everything belongs in a sequence. A referral introduction, a reply to someone who reached out, a single high-value prospect worth a hand-written note, a follow-up after a call: these are one email to one person, and Apollo sends them properly.

```
apollo_emailer_messages_create   → drafts it (contact_id, subject, body_html)
apollo_emailer_messages_send_now → sends it (needs the sender mailbox)
apollo_emailer_messages_email_send_status → confirms delivery
```

Three reasons to send it through Apollo rather than from your own inbox:

1. **It is logged against the contact.** The email becomes part of that person's record instead of disappearing into a personal sent folder, so the next person to touch the account can see it.
2. **It sends from a real mailbox** you choose, with the same sending rules and pacing as everything else.
3. **You get a delivery answer.** `email_send_status` returns `completed` or `failed`, and on failure gives `not_sent_reason` and `failure_reason` (bounce, quota, spam block).

### Delivery is not opens: two different endpoints

These get conflated constantly, so be precise:

| Question | Call | Gives |
|---|---|---|
| Did it arrive? | `apollo_emailer_messages_email_send_status` | `completed` / `failed`, plus `not_sent_reason` and `failure_reason` on failure |
| Did they open or click it? | `GET /emailer_messages/{id}/activities` ("Check Email Stats", REST) | Open and click activity |
| What have we sent them? | `GET /emailer_messages/search` (REST) | Sent outreach history |

Only the first is on MCP. The other two are REST, so lane 3 or 4.

**And open tracking has a cost this library normally refuses to pay.** It works by embedding a tracking pixel, which hurts deliverability and is increasingly stripped or pre-fetched by mail providers anyway, which makes the number unreliable as well as expensive. That is why cold campaigns here run with `open_tracking: false`.

So: **delivery confirmation always, open tracking only when you deliberately accept the trade.** For a warm one-off to someone expecting to hear from you, that trade is far more defensible than for a cold campaign. Never turn pixels on for cold just to get a nicer dashboard.

Practical notes:

- `contact_id` is required, so the person must exist as a contact first. Enrich and create them (List) before you can email them.
- You can override `recipients` to add cc, bcc, or send to a non-contact address, while the message stays attached to the contact record. Useful for looping in a colleague on an intro.
- The draft-then-send split exists so a human can approve. If the operator has already confirmed recipient, subject, and body, run both calls back to back; from their side it is one action.
- Keep it short. The tool's own guidance is 25 to 85 words and one clear ask, which matches this library's copy rules.

**This still sends a real email.** Same rule as everything else: show recipient, subject, and body, and wait for an explicit yes.

## Common mistakes

- **Saying Apollo "sends" the LinkedIn message or "makes" the call.** It queues a task. A human acts.
- **Turning every lead into a five-channel gauntlet.** Reserve calls and LinkedIn for High-priority leads (from List scoring or a signal match). Blanket multichannel is just more noise, faster.
- **Clearing the LinkedIn queue in one sitting.** Apollo does not cap invites, so you have to. Keep new connection requests under 30 a day, 20 to be safe, and trickle them across days. A week's invites fired off in one afternoon is the exact burst LinkedIn flags.
- **A task with no association.** `contact_id`, `account_id`, or `opportunity_id` is required, or the create fails.
- **Forgetting these are gated.** If task-function steps error, it is your Apollo plan, not your setup. Fall back to email-only.
