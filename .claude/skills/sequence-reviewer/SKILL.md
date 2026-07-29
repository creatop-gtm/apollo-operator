---
name: sequence-reviewer
description: "Review a drafted cold email sequence for spam and deliverability risks (firm) and copy craft (suggestions). Advisory, never blocks. Use before building or activating a sequence."
---

# Sequence Reviewer (Message)

Read a drafted sequence and tell the operator what will hurt it before it sends. Advice, not a gate: you flag, you recommend, the operator decides. Split findings into what genuinely damages deliverability (be firm) and what is a copy or taste call (suggest, do not insist).

## When to use

- After `apollo-sequence-builder` drafts a sequence, before it is built or activated.
- On any inherited or hand-written sequence someone wants a second read on.

## What to check

### Deliverability and spam risk (be firm, this hurts them)
- **Spam-trigger words:** free, guarantee, act now, limited time, risk-free, "100%", "$$$". Flag each.
- **Links and images:** more than one link, any image, or a tracking-heavy footer in a cold email. Plain text wins.
- **Shouting:** ALL CAPS words, multiple exclamation marks, emoji stacks.
- **Length as a spam signal:** a 200-word cold email is both a spam signal and a worse email.

### Structure and voice (recommend, this is craft)
- **3 steps, 2 variations each.** Day 0, Day 3, Day 7, delays relative to the previous step.
- **60 to 90 words per email**, step 3 the shortest.
- **One CTA, soft.** One focus per email: outcome or mechanism, not both.
- **Angle rotates across steps.** No repeated value prop.
- **Subject: lowercase, 1 to 3 words, reused across the thread** (steps 2 and 3 reply in-thread, no new subject).
- **Voice:** no em dashes, no buzzwords (leverage, synergy, revolutionary, best-in-class, game-changing), Oxford comma, sentence case, short paragraphs.
- **Step 3 is routing or a soft breakup.** Flag any new pitch there.
- **Persona fit:** ATL emails are 2 to 3 tight strategic sentences; BTL are 3 to 4 operational ones. Flag an exec email full of time-saved operational detail, or an IC email full of board-level abstraction. See `../apollo-sequence-builder/references/atl-btl.md`.

### Personalization
- **Opener is specific and real**, not "love what you're doing at {{company}}." Generic openers read as theater. Flag them.
- **Not creepy.** Specific is good, surveillance is not.

## How to report

Two buckets, then a recommendation:

> **Will hurt deliverability (worth fixing):** step 2 variation B uses "risk-free guarantee" and has two links. Both are spam signals.
> **Copy notes (your call):** step 1 is 120 words, longer than ideal. Step 3 adds a new pitch instead of a routing question. Subject lines are title-case; lowercase reads more personal.
> **Recommendation:** fix the two spam signals, then it is safe to send. The copy notes are improvements, not blockers.

Lead with the deliverability bucket, keep the copy bucket as suggestions, and let the operator decide what to change. Never refuse to proceed.

## Common mistakes

- **Treating a copy preference as a rule.** Word count and subject casing are strong defaults, not law. Flag, do not block.
- **Missing the real spam signals** while nitpicking style. The spam words and links are what actually land you in the promotions tab.
- **Rewriting instead of reviewing.** Point at the problem and suggest; let the writer (human or the builder skill) fix it.
