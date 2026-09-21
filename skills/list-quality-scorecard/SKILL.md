---
name: list-quality-scorecard
description: "Check a lead list before it goes into a sequence. Gives a deliverability readout (firm) and a targeting readout (advisory). Surfaces issues, never blocks. Use before enrolling any list."
---

# List Quality Scorecard (List)

Check a lead list before it goes into a sequence, and tell the operator what is worth knowing. This is advice, not a gate. The operator always decides whether to send.

Apollo lists almost always come back technically clean, so the two things this actually surfaces are: **deliverability risks** (things that will hurt your sending) and **targeting notes** (how much of the list matches the ICP you set). Keep those two separate, because they mean different things.

## Surface, do not block

Never refuse to proceed. Show what you found, recommend, and then do what the operator asks. Some people deliberately run outside their ICP, or want to test an adjacent segment, and that is a legitimate choice. The scorecard informs that choice, it does not veto it.

- **Deliverability risks hurt the operator**, so be direct about them: unverified emails bounce, catch-alls are low-intent, duplicates double-send. Strongly recommend fixing these, because they cost domain reputation. Still their call.
- **Targeting is a strategic choice**, so present it as information: "N% of this list is outside the ICP you set." Not a problem unless they decide it is.

## When to use

- After `apollo-list-builder` has searched, enriched, and deduped a list (run it on `enriched.csv` or `scored.csv`).
- Before enrolling anyone in a sequence, as a habit, not a checkpoint.
- Any time you inherit a list from elsewhere.

## What the user sees

Two readouts and a recommendation, in plain language:

> **Deliverability: clean.** 2,591 leads, all emails verified, no duplicates, no catch-alls, clean names. Good to send.
> **Targeting: heads up.** About a third (868) are outside the ICP you set: 310 management consulting, 62 real estate, 42 insurance, 32 PR (which your ICP excludes), and others. If that is intentional, send away. If not, tighten the industry filter and re-pull.
> **Recommendation:** deliverability is clean, so this is ready to send. The only question is targeting, and that is your call.

Or, when there is a real deliverability problem:

> **Deliverability: risk.** 18% of emails are unverified and 9% are catch-all addresses. These will bounce and drag your domain. I would fix this before sending. Want me to drop the unverified and catch-all rows?
> **Targeting: on-ICP.** 94% match. No concerns there.

Lead with the two readouts, name the specifics, recommend, then ask what they want to do.

## Background scoring (for the agent)

Score each dimension 0 to 100. **Deliverability** dimensions decide the deliverability readout; **targeting** dimensions decide the targeting note. Do not blend them into one number the user has to interpret.

| # | Dimension | Group | Scored on |
|---|---|---|---|
| 1 | Email verification coverage | Deliverability | % of emails verified. Unverified equals bounce risk. |
| 2 | Duplicate email rate | Deliverability | 100 at 0% dupes, scaling down. |
| 3 | Catch-all / generic inbox density | Deliverability | 100 minus the % on info@/contact@/sales@ inboxes. |
| 4 | Per-domain concentration | Deliverability | 100 if avg <= 3 per company, lower as it climbs. |
| 5 | Bad-title rate | Deliverability/quality | 100 minus the % of intern/assistant/student/retired titles. |
| 6 | Name quality | Deliverability/quality | % with a real first and last name. |
| 7 | ICP fit | Targeting | % matching the profile's industries, headcount, geo. |
| 8 | Title relevance | Targeting | % of titles matching the ICP titles. |

**Deliverability readout:** clean if verification is 100% and the other deliverability dimensions are healthy. Otherwise "risk," with the specific offenders named and a recommended fix. This is the one to be firm about.

**Targeting readout:** just report the ICP-fit and title-relevance percentages, and name the drift (which off-ICP industries, how many). Never phrase it as a failure.

**ICP fit is the dimension that lies if you let it.** Match strictly against the profile's industries AND its exclusions, not a loose keyword OR. A loose match rates almost anything as a fit. In a real run on a 2,591-row list, a loose keyword match scored it 99% fit, while a strict check found 33% industry drift, including contacts from an industry the ICP explicitly excluded. When a row is ambiguous, judge it against the ICP individually. Slow down on this one, then report it as a note, not a verdict.

## Why it matters

The deliverability half protects the operator's domains, which is not free to recover once burned, so it is worth being direct about. The targeting half is just intelligence: it tells the operator what they are actually about to email, so the decision to send (or not) is informed rather than blind.

## Common mistakes

- **Blocking.** The scorecard advises, it never refuses. The operator sends if they want to.
- **Treating ICP drift as a defect.** It is a choice. Surface it, do not scold it.
- **Blending deliverability and targeting into one grade.** They are different questions with different owners.
- **Showing the user the weight table.** They want the two readouts and a recommendation, not the math.
