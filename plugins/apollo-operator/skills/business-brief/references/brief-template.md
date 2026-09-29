# brief.md template

The structure for the single-file business brief that `business-brief` produces. Copy the skeleton, fill what you know, mark the rest `[TBD]`. Keep the whole file readable in one sitting: one to two pages is the target, specific beats complete.

Sections exist to be read by an agent mid-task ("what proof do I cite in step 2?" jumps straight to Proof). Keep the headings exactly as below so downstream skills can rely on them.

---

```markdown
# [Business name]: brief

> Last updated: [date]. Read this before touching the account.

## Identity

One paragraph: what the business does, for whom, and the outcome it delivers.
Written like you would explain it to a smart friend, not like the homepage.

- Website: [url]
- One-liner: [what we sell, one sentence]

## Offer

- What is sold: [the product or service, concretely]
- Pricing model: [flat / tiered / usage / custom, rough range if shareable]
- The outbound offer: [the specific CTA or lead magnet cold prospects are
  asked to respond to. This is what sequences point at.]

## Who buys

One short paragraph describing the ideal customer as a story, not a filter
set (the filters live in profile.yaml).

The buying committee:
- **Champion:** [title(s), what they feel day to day]
- **Economic buyer:** [title(s), what they sign off on and why]
- **End user:** [title(s), if different]

## Why they buy

- Pains: [the 2 to 4 problems, in the buyer's words]
- Triggers: [what makes this urgent now: funding, hiring, a tool change,
  a season, a regulation]
- Desired outcome: [what "solved" looks like to them]

## Why they don't

The objections actually heard, each with the honest answer:

- "[objection]" → [the real answer, not the deflection]
- "[objection]" → [answer]

## Proof

Only what is real. Numbers beat adjectives.

- [Case study or result, with the number and the customer type]
- [Named customers or logos that can be referenced]
- [Testimonial lines worth quoting, with attribution]

## Language

- Buyers say: [the words buyers use for the problem and the product]
- We say: [terms the business prefers]
- Never say: [banned words, competitor framings, claims to avoid]
- Tone: [casual / peer-to-peer / formal, one line on voice]

## Competitive frame

- Alternatives buyers consider: [including "do nothing" and spreadsheets]
- Why us, in one line: [the honest differentiator]

## Constraints

Anything outreach must respect: compliance rules, geographies to avoid,
claims that are off-limits, existing customers or partners not to contact.

## Changelog

- [date]: [what changed and why]
```

---

## Filling notes

- **Identity** is the section agents read when everything else is skipped. Make it carry the whole picture.
- **The outbound offer** is not the product. "A free teardown of your current sequence" is an offer; "our platform" is not. If the business has no outbound offer yet, mark it `[TBD]` and flag it: sequences cannot be written without one.
- **Why they don't** is usually empty on a first draft because nobody volunteers objections. Ask directly (interview question in step 3 of the skill). This section writes the best step 2 and step 3 emails.
- **Proof** feeds `apollo-sequence-builder` step 2 (context and credibility). No proof in the brief means no credibility in the sequence, so chase at least one real number.
- **Language** feeds the `voice:` block of `profile.yaml` and the reviewer's robot-copy check in `sequence-reviewer`.
