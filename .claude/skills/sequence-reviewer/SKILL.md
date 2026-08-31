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
- **Word count by step: 60 to 75 for step 1, 45 to 60 for step 2, 55 to 75 for step 3.** Step 2 is the lightest touch, not step 3. Step 3 runs longer because it carries the routing question *and* the lower-commitment fallback. (Corrected v1.3 from "step 3 the shortest", which was wrong.)
- **One CTA, soft.** One focus per email: outcome or mechanism, not both.
- **Never "quick call", and flag it as a rule rather than a preference.** Also "hop on a call", "quick chat", "quick sync", "jump on a call". "Quick" is the sender pre-apologising for their own ask: it signals the request is an imposition, and it is not believable anyway. Minimising the ask makes it easier to decline. Replace with a real number or a question about interest: "Worth 15 minutes?", "Worth a look?", "Open to hearing more?", "Want me to show you what that looks like?" Naming the actual time is fine; advertising the smallness of the ask is not.
- **Greeting shape should tighten across the steps.** Step 1 takes a block greeting on its own line. Step 2 opens `Hi again {{first_name}},` and continues on the same line. Step 3 is tighter still and can drop the greeting word entirely (`{{first_name}}, are you the right person…`). A sequence where all three steps open identically reads like three separate emails rather than a thread.
- **Flag throat-clearing lines.** "Following up on this.", "Quick follow-up.", "Last one from me." Standalone transition lines occupy the most valuable line in the email and carry nothing. `Hi again` already signals a follow-up.
- **Flag labelling phrases.** "How it works:", "The part that matters is". Say the thing instead of announcing you are about to say it.
- **Flag setup-and-contrast.** "Most vendors do X. We do Y." is the writer showing their reasoning; the reader only needs Y. Cutting the setup usually removes 20 to 30 words from a follow-up with no loss.
- **Flag the summarising payoff.** A sentence that explains a benefit already stated ("That is why month six beats month one") is the writer admiring the point. Cut it.
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
