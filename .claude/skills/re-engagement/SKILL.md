---
name: re-engagement
description: "Go back to leads who finished a sequence and never replied, which is 90%+ of any list. Covers when it is safe to re-approach (60-90 days), the permanent exclusions, re-verification, and bringing a genuinely new angle rather than another follow-up. Use for quarterly re-engagement passes or when a finished list looks dead."
---

# Re-engagement (Iterate)

A sequence finishes. Somewhere between 90 and 97% of the people in it never replied. Most operators treat that as a dead list and go buy more leads.

That is backwards. Those people are the most valuable asset the campaign produced, and they cost nothing to reach again.

## The stance

**Silence is not rejection.** A non-reply means the message did not land, at that moment, in that inbox, in that framing. It does not mean the person is a bad fit, and it does not mean they will never buy. Every one of them already passed your ICP filter, was verified deliverable, and cost credits to enrich. Re-approaching them is the cheapest qualified reach available.

**Stale leads are an asset, never a write-off.** The list you built in March is not worthless in September. It is a warm-ish audience you already paid for, and the only thing it needs is a new reason to hear from you.

What re-engagement is **not**: sending the same sequence again with a different subject line. That is not a second attempt, it is the same attempt with worse odds, because the recipient has already ignored this exact argument once.

## When it is safe to go back

| Since the sequence finished | Do |
|---|---|
| Under 60 days | **Wait.** Too soon reads as pestering and earns spam complaints, which cost you the domain. |
| 60 to 90 days | **The window.** Long enough to be forgotten, short enough that the company situation is still recognisable. |
| 90 days to a year | Fine, and often better, because something has genuinely changed. Expect more job changes and stale data. |
| Over a year | Treat it as a fresh list. Re-verify everything before sending. |

Three hard exclusions, no matter how much time has passed:

- **Anyone who replied "not interested", asked to stop, or unsubscribed.** Permanent. Not a timing problem.
- **Anyone who bounced.** Re-sending to a bad address a second time is how a domain dies twice.
- **Anyone currently in another live sequence.** Check before enrolling, not after.

## Process

### 1. Pull the finished, non-replying set

These are contacts Apollo marked finished in a previous sequence. Apollo blocks re-enrolling them by default, which is the correct default and exactly the guard you are deliberately lifting:

- **MCP:** `apollo_emailer_campaigns_add_contact_ids` with `sequence_finished_in_other_campaigns: true`.
- **CLI:** `apollo sequences add-contacts --finished-in-other`.

Turning that flag on **is the whole decision.** It is off for a reason, so turn it on knowingly, for a list you have deliberately assembled, and never as a way to make a number go up.

### 2. Re-verify before you spend or send

Data decays, and it decays fastest in exactly the fields you need. Over 60 to 90 days expect people to have changed jobs and addresses to have gone dead. Run the list through your verification service again (List, step 7) and drop anything that is no longer deliverable. Cheaper than a bounce spike, by a wide margin.

**Job changes are an opportunity, not just decay.** Someone who moved to a new company is a *new* prospect with a warm prior touch, and a new-in-role person has budget and a mandate to change things. Apollo's `sequence_job_change` flag and the job-change signal (Signals) both surface this. Route them into a fresh angle rather than the re-engagement sequence.

### 3. Bring a new reason, not a new subject line

This is the part that decides whether it works. The recipient has already declined this argument once by ignoring it. Something must be different, and it has to be different in a way they can see in the first line.

Pick one:

- **A new signal.** They are hiring now, they raised, they adopted a tool, they visited your pricing page. This is the strongest option because it is genuinely about them, right now.
- **New proof.** A result, case study, or number you did not have last time. Especially strong if it is in their vertical.
- **A different angle on the same offer.** You led with cost last time; lead with speed or risk now. Same product, different pain.
- **A different persona at the same company.** Not re-engagement of a *person* but a second route into an *account*. Respect the per-company cap.
- **Honest acknowledgement.** "I wrote to you in March about X, it clearly was not the moment" is disarming, and it is true. Use it sparingly and never as a guilt trip.

Anything that boils down to "just following up" or "bumping this to the top of your inbox" is not a reason. It is an admission that you have nothing new to say, and it reads that way.

### 4. Keep it shorter than the original

Two steps, not three. These people have already had a full sequence from you, and the total volume they have received is what determines whether you read as persistent or as a problem. If two touches with a genuinely new angle do not land, stop and put them back in the pool.

### 5. Send it slower

Re-engagement carries more complaint risk than cold, because the recipient may remember you. Send at a lower daily volume than a cold campaign, watch the bounce and complaint rates harder, and keep the `apollo-deliverability` thresholds in front of you. Have the kill switch ready (`apollo-go-live`).

## Cadence

Re-engagement is not a one-off. The right rhythm is a **standing quarterly pass**: everything that finished more than 60 days ago, minus the exclusions, gets one new angle. Fold it into the weekly review (`weekly-rhythm`) as a monthly or quarterly checkpoint, so the pool never silently rots.

Track it separately. Re-engagement reply rates are not comparable to cold reply rates, so mixing them corrupts both numbers. See `experiment-design`.

## Common mistakes

- **Re-sending the same sequence.** They ignored that argument once. Nothing has changed except your patience.
- **Going back too soon.** Under 60 days is pestering, and it converts a non-reply into a complaint.
- **Re-engaging people who said no.** "Not interested" is an answer. Treat it as one, permanently.
- **Skipping re-verification** because the list was clean when you built it. It was. Months ago.
- **Flipping `finished_in_other` on by default** so more people get added. That guard exists to stop double-touching, and switching it on quietly is how a list gets burned.
- **Treating a job change as decay.** It is the single best thing that can happen to a stale lead.
- **Running it at cold volume.** More complaint risk per send, so less volume per day.
