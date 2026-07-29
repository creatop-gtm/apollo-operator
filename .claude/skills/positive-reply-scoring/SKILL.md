---
name: positive-reply-scoring
description: "Classify replies and compute positive reply rate, the metric that predicts revenue. Use to tell whether a campaign is actually working, not just getting replies."
---

# Positive Reply Scoring (Iterate)

Reply rate tells you people are paying attention. Positive reply rate tells you they want what you are selling. This computes the second, because it is the number that actually predicts revenue.

A campaign can get a 5% reply rate and still be a disaster: if most of those replies are "unsubscribe" and "not a fit," you are burning domains for nothing.

```
positive reply rate = positive replies / total sent
```

## When to use

- After a campaign has run at least 14 days (before that, the sample is too small to trust).
- Weekly, as the Wednesday task in `weekly-rhythm`.
- When comparing two campaigns in an experiment (use the same cutoff for both).

## Getting the replies

Pull reply data from Apollo (sequence analytics via `apollo_analytics_sync_report`, or the inbox), or work from an export the operator provides. If the reply bodies are not available through the MCP, ask the operator to paste or export them: the value of this skill is the classification and the metric, not the fetch.

## Classify every reply

One bucket each. Classify only the **first** reply per lead (later messages are the conversation, not the signal).

| Label | Meaning | Positive? |
|---|---|---|
| interested | "yes, tell me more," booked | yes |
| soft | "send info," "reach out in Q3" | yes |
| referral | "not me, talk to X" | yes (high value) |
| neutral | a question, no commitment | no |
| not-now | "not right now" | no |
| not-fit | "not a fit" | no |
| hostile | angry, complaint, report | no (track as risk) |
| unsubscribe | opt-out | no |
| ooo / bounce | auto-reply / delivery fail | exclude from the denominator |

Positive reply rate = (interested + soft + referral) / total sent. Exclude OOO and bounces from the denominator, they are not real replies.

## Report

- **The number:** positive reply rate, and positive as a share of real replies.
- **Act now:** list the interested replies. Respond to these within minutes, not hours. A fast reply converts far better than a slow one.
- **Follow up:** referrals, reach the named person within a day and mention who sent you.
- **Watch:** hostile replies and the unsubscribe rate. Elevated hostility usually means bad targeting, not bad luck.

## Reading the number

No borrowed benchmarks. Use the reply-rate reality from `../operator-context/references/outbound-principles.md`: below 1% something is broken, 1 to 5% is normal, near 5% is good, above 5% is excellent. Positive reply rate is the stricter cut of that, so judge it against your own past campaigns, not an industry chart.

## Common mistakes

- Trusting reply rate alone. Spam-trap and unsubscribe replies inflate it while the campaign fails.
- Scoring later messages instead of the first reply.
- Sitting on interested replies. Speed is the whole game once someone raises a hand.
