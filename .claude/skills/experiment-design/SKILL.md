---
name: experiment-design
description: "Design single-variable outbound experiments (list-only, copy-only, combined) so you learn what actually works. Use to improve a campaign deliberately."
---

# Experiment Design (Iterate)

If you change the list, the copy, and the offer at once, and it works, you learn nothing, because you cannot say what worked. This forces one variable per experiment so you actually learn.

## When to use

- A campaign is running and you want to improve it deliberately, not by guessing.
- You have a baseline: at least one campaign that ran three weeks, so you know your normal.

Do not experiment before you have a baseline. You need a control before you can run a test.

## The three kinds

- **List-only:** change the targeting, hold copy and offer fixed. Learn whether a segment is a better fit.
- **Copy-only:** change the copy or a variant, hold the list and offer fixed. Learn whether a message resonates.
- **Combined:** a whole new campaign for a new ICP. You cannot isolate, so treat the result as a hypothesis, not a conclusion.

## The framework

1. **Write the hypothesis in one sentence.** "Targeting heads of ops instead of VPs of sales will get a higher positive reply rate, because they feel this pain daily." If you cannot write it in a sentence, you do not understand the experiment yet.
2. **Name the single variable.** Write down exactly what changes and everything that stays the same. If any "constant" is actually moving, stop and fix it, or call it a combined experiment.
3. **Size it honestly.** Small samples lie. If a test arm has only a handful of sends, you cannot tell signal from noise. Bigger effects need fewer sends; small effects need many. When unsure, run more before you conclude.
4. **Decide success up front.** Write the target and the failure line before you launch, so you cannot rationalize a different "learning" after seeing the data.
5. **Launch both arms at the same time**, same infrastructure split, same schedule. Day-of-week and warmup differences will confound a staggered test.
6. **Measure after the full sequence has run** (through the last step plus a reply grace period), using `positive-reply-scoring`. Measuring early biases toward the first email.
7. **Weight the result by confidence.** A clean single-variable test with enough volume is high confidence. A combined test is low. Say which.

## Priority order

If you are unsure what to test first, this is usually the impact order: **list, then offer, then subject, then opener, then CTA, then timing.** A bad list beats any copy, so fix targeting before you A/B a subject line.

## No borrowed benchmarks

Judge every result against your own past campaigns, not an industry number. Your baseline is the only honest yardstick.

## Common mistakes

- Changing three things and claiming victory. You learned nothing.
- Calling it early. Cold replies trickle in over weeks.
- Testing copy on a broken list. Fix the list first.
- A combined win adopted as a new baseline. Split it into single-variable follow-ups to find what actually drove it.
