---
name: business-brief
description: "Build one readable brief.md capturing the business: identity, offer, who buys and why, objections, proof, and voice. The narrative context layer every other skill reads. Use before ICP work, when copy comes out generic, or when the business changes."
---

# Business Brief (Context)

Turn scattered knowledge about a business into one readable file, `brief.md`, that a person or an agent can read once and immediately understand: who the business is, what it sells, who buys it, why they buy, what proof exists, and how buyers actually talk. Every downstream skill gets sharper because this file exists. Searches target the right people, copy says something true, and objections get answered instead of ignored.

This is where context enters the system. `apollo-icp-builder` turns the brief into targeting. `apollo-sequence-builder` turns it into copy. Without a brief, both are guessing.

## When to use

- Setting up outbound for a business that has no `brief.md` yet. Run this before `apollo-icp-builder`.
- Onboarding an agent (or a new teammate) onto an existing Apollo account.
- The copy keeps coming out generic. That is a context problem, not a writing problem.
- Refreshing after something material changes: a new case study, a new offer, a repositioning.

## Principle

Read `references/outbound-principles.md` if you have not. The one that drives this skill: outbound is a system game, and account knowledge is part of the system. A brief that compounds beats a memory that resets every session.

Two rules of the brief itself:

- **It is an internal document, not marketing.** It records what is true, including weaknesses, lost deals, and real objections. A brief that reads like the website is a failed brief.
- **Never invent.** Every claim traces to a source: the website, a document the operator shared, or their own answers. What you do not know gets marked `[TBD]`, never filled with plausible fiction.

## Relationship to `profile.yaml`

Two layers, two jobs:

- **`brief.md` is the narrative layer and the source of truth** for who the business is, what it sells, why buyers buy, proof, and voice. Humans read it. Agents read it first.
- **`profile.yaml` is the machine layer**: Apollo filter sets, scoring signal definitions, saved IDs. Skills parse it.

When they disagree, the brief wins. Update the yaml to match, not the other way around.

## Process

### 1. Collect what already exists

Before reading anything, ask the operator for context you cannot see: pitch decks, case studies, sales call transcripts, onboarding docs, past campaign copy, positioning docs, a Notion page, anything. People always have more context than they think to share, and the brief is only as good as what goes in. Do not start drafting until you have asked.

### 2. Read the website

Home, product or services pages, pricing, case studies, and customer pages. Extract what they sell, who the named customers are, what claims and numbers they publish, and the exact language they use. The website is the floor of the brief, not the ceiling: it shows how the business presents itself, and hides everything else.

### 3. Interview the gaps

One batched round of questions, only for what steps 1 and 2 did not answer. The high-yield ones:

- What do you actually sell, in one sentence, and what does it cost (roughly, or the pricing model)?
- Who are your three best customers, and what made them great?
- What deals have you lost, and why?
- What is the objection you hear most, and what is your honest answer to it?
- What proof do you have with numbers in it?
- What would you never say, and what words do your buyers use that outsiders get wrong?

Do not interrogate. Batch the questions, take the answers, move on.

### 4. Draft the brief

Write `brief.md` following `references/brief-template.md`. Plain language, short sections, specific over complete. A one-page brief that is all true beats a five-page brief padded with guesses. Mark every gap `[TBD]`.

### 5. Get the human pass

The operator reads the draft and corrects it. The corrections are the most valuable content in the whole process ("we never say automation, we say workflow"; "that case study is stale, use this one"). Fold every correction in.

### 6. Sync `profile.yaml`

Derive or update the `business:` and `voice:` blocks of `profile.yaml` from the finished brief. If a profile already exists and disagrees with the brief, flag the mismatch to the operator, then align the yaml to the brief. Hand off to `apollo-icp-builder` to turn the "who buys" section into a validated Apollo search.

## Output

One `brief.md` at the project root, next to `profile.yaml`, following the template. Updated `business:` and `voice:` blocks in `profile.yaml`. A dated line in the brief's changelog saying what changed.

## Mine the team's own recorded calls first

**Added v1.3, after testing the surface live.** If the team records calls or meetings in Apollo, the highest-quality source for this brief is already sitting in their account, and reading it is **free**. Four commands, no credits consumed on any of them, verified across a 79-conversation account on 2026-08-31.

```bash
apollo conversations search --limit 25 -f json          # find them
apollo conversations show --id <conversation_id>        # the whole thing
apollo conversations export --start <iso> --end <iso> --email <teammate>
apollo conversations export-status --id <export_id>     # returns a signed download URL
```

**`show` is the one that matters**, because it returns a `call_summary` object Apollo has already structured for you:

| Field | Feeds |
|---|---|
| `objections` | The brief's objections section, in the buyer's own words |
| `pain_points` | Pains, with the ones that actually came up ranked over the ones marketing lists |
| `outcome` | What closes and what stalls |
| `next_steps` | The real call to action, as opposed to the one on the website |
| `pricing_discussion` | How price actually lands, which almost never matches the pricing page |

It also returns the full `transcript` as a list of segments (`participant_name`, `spoken_sentence`, `start_time`, `end_time`), `key_topics` with question and tracker insights, `participants` split into `internal` and `external`, and signed URLs for the audio and video.

**Why this outranks the website.** The website is how a business describes itself. A recorded call is how its buyers actually describe their problem, and which objections come up often enough to need an answer. When the two disagree, the call wins. Marketing copy is aspirational; a prospect saying "we tried this before and it didn't stick" is evidence.

Two cautions. Conversations are **real customer conversations**, so treat them as confidential source material: mine them for patterns and never paste names, quotes, or company details into anything that ships or goes to a third party. And `search` returns `state`, which is worth checking, since a conversation only carries a summary once it reaches `insights_generated`.

**Search field names are not the obvious ones**: it is `topic` rather than title, `start_time` rather than started_at, and `conversation_type` (`video_conference` or `phone_call`) rather than type. Filters cover account, contact, organization, tag, tracker, and date range. `export` fails with `404 Unable to find conversations with the given time range` when the window is empty, which is a real answer rather than an error.

## Push it into Apollo's Context Center

Apollo has a native, team-wide home for exactly this information: the **Context Center**, an ICP profile plus a set of product profiles that Apollo's own AI features read when generating outreach. Its fields line up almost one to one with `brief.md`:

| `brief.md` section | Context Center field |
|---|---|
| What the business does | `company_overview`, `company_or_product_name`, `domain` |
| Who they sell to | `customer_profile`, `icp_fit_criteria`, `high_value_fit_criteria` |
| Who they do *not* sell to | `disqualification_criteria` |
| Pains | `customer_pain_points` |
| Offer and why it wins | `value_proposition`, `product_differentiators` |
| Proof | `social_proof` |
| Competitive frame | `primary_competitors` |
| The ask | `call_to_action` |

Tools: `apollo_context_center_show` to read (**always read before writing**), `apollo_context_center_create_profile` / `update_profile` for the ICP, and `apollo_context_center_create_product` / `update_product` for each offering.

**Worth doing**, because it means the brief stops being a file only this library reads and starts powering Apollo's own AI too, for everyone on the team.

**Seven cautions.** The first four were known; the last three came out of running this against a real, badly stale Context Center on 2026-08-31.

- **It is shared and destructive.** One ICP per team, and every field you send **replaces** the previous value. An edit changes it for everyone, and it is not reversible. Read first, send only what changes, and confirm before writing.
- **`approved` decides whether it is live.** `false` keeps it a draft; `true` publishes it to Apollo's AI features. Ask which the operator wants rather than defaulting.
- **Each `create_product` call makes a new product.** Calling twice makes two. To change one, read it and update it.
- **Never invent positioning to fill a field.** Same rule as the brief itself: a `[TBD]` is honest, and a fabricated value proposition will show up in generated copy sent to real people.

- **There is no way to delete a product.** `create_product`, `show_product`, and `update_product` exist. There is no delete, on MCP or REST. A stale product set is therefore **permanent**, and the only move is to repurpose each one in place. Count the existing products before you plan: if a business has four services and the Context Center holds five stale products, something has to absorb the fifth. Map deliberately rather than leaving an old service sitting there contradicting the new ones.

- **Back up before writing, because there is no read-back path outside MCP.** `GET /context_center` returns 404, so `apollo_context_center_show` is the only way to read the object. Combined with edits being irreversible, that means: read the whole thing, save it to disk, and only then write. Without that, a bad overwrite has nothing to restore from.

- **`additional_context` is the highest-leverage field in the whole object, and the mapping table above assigns it nothing.** It is free text that Apollo's AI reads when generating copy, which makes it the right home for everything a field-shaped schema cannot hold: banned self-descriptions, the exact register to use, prices that must never be invented, tone and mechanics rules, and the words that are forbidden. If a business has copy guardrails anywhere in its brief, they belong here. A correct `value_proposition` with no guardrails still produces copy that breaks the rules.

`brief.md` stays the source of truth, because it holds nuance and open questions that a structured profile cannot. The Context Center is a **projection** of it into Apollo. When they disagree, the brief wins and the Context Center gets updated.

## Common mistakes

- Writing marketing copy instead of an internal document. The brief admits weaknesses; the website does not.
- Inventing proof points or customer names to fill a section. `[TBD]` is a correct answer.
- Skipping the interview because the website "covered it." Websites never contain lost deals or real objections, and those write the best emails.
- Duplicating targeting mechanics into the brief. Filter sets and scoring signals live in `profile.yaml` (Targeting), the brief describes who buys and why in prose.
- Letting the brief go stale. New case study, new offer, new pricing: update the brief the week it happens, and date the change.
