---
name: apollo-sequence-builder
description: "Write a 3-step, 2-variation cold email sequence matched to the persona (ATL or BTL) and build it in Apollo as an inactive draft. Use to write and create outbound sequences on Apollo."
---

# Apollo Sequence Builder (Message)

Write the sequence, then build it in Apollo as an inactive draft ready for a human to review and activate. Copy is where most outbound dies, so this level is opinionated: short, one idea per email, matched to the person you are writing to, and never activated by the machine.

## When to use

- `profile.yaml` has the offer, the ICP, and the voice.
- A graded list exists (List done), or you at least know the persona you are writing to.

## The rules that do not move

From `../operator-context/references/outbound-principles.md` and the core copy rules. Non-negotiable regardless of framework:
- **3 steps, 2 variations per step.** Day 0, Day 3, Day 7.
- **60 to 90 words** per email. Step 3 is the shortest.
- **One CTA, soft.** "Worth a quick look?" not "Book a 30-minute demo."
- **One focus per email: outcome or mechanism, never both.**
- **Rotate the angle across steps.** Never repeat the same value prop.
- **Subject: lowercase, 1 to 3 words. Reused down the thread (steps 2 and 3 auto-reply in-thread), but a different angle or offer usually earns its own subject** (see step 5).
- **Plain text. No em dashes. No buzzwords** (leverage, synergy, revolutionary, best-in-class). Oxford comma.
- **Step 3 is a routing question or a soft breakup. No new pitch.**

## Process

### 1. Match the person (ATL or BTL)
Before choosing words, decide who you are writing to. Executives (VP, C-level, director) and managers/ICs read completely different emails. See `references/atl-btl.md`. This decides length and language for the whole sequence. Pull seniority from the list or `profile.yaml`.

### 2. Pick the angle
Choose a framework that fits the offer and the person from `references/frameworks.md` (Do the Math, Ask Before Pitch, Challenge of Similar Companies, and others). One angle per email, a different one each step.

### 3. Write the three steps
- **Step 1 (Day 0):** one focus (a pain, an outcome, a question, or specific proof), a personalized opener, one soft CTA.
- **Step 2 (Day 3, same thread):** context and credibility, a different angle, a real case snippet, shorter than step 1.
- **Step 3 (Day 7, same thread):** routing ("am I reaching the right person?") or a soft breakup. No new pitch.
- Write **2 variations** of steps 1 and 2, each testing one idea (different pain, different proof, with vs without a lead magnet).

### 4. Personalize the opener honestly
The opener should read like you spent ten minutes on this person. Claude does the research (reads the site, finds a real, specific fact) and writes one suggestive opening line, merged per lead. Do not rely on a black-box AI opener, and never fake specificity. (Apollo has its own per-recipient AI opener, `has_personalized_opener`, which spends credits; our default is to write the opener ourselves so a human can approve it.)

### 5. Review, then build
Run `sequence-reviewer` on the draft. Fix what it flags. Then build it in Apollo with `apollo_sequences_create`:
- `active: false`. Always. The machine never turns a sequence on.
- 3 `auto_email` steps. Step 1 is a `new_thread`; steps 2 and 3 are `reply_to_thread` (same thread, no subject, keeps the conversation).
- Delays are **relative to the previous step**: step 1 `wait_time: 0`, step 2 `wait_time: 3`, step 3 `wait_time: 4` (that is Day 7, not Day 11). This is the mistake everyone makes.
- 2 `emailer_touches` per step (the A/B variants). `creation_type: manual` (a human wrote it). Tone from the profile voice (Direct, Formal, or Casual).
- **Format the body so it breaks into paragraphs.** Write 2 to 3 short paragraphs, not one wall of text. Separate them with `<br><br>` inside a single `<div>`, like `<div>First paragraph.<br><br>Second paragraph.<br><br>Closing line and CTA.</div>`. This is not cosmetic: live-tested on Apollo, `<br><br>` is the only structure Apollo keeps as real line breaks in the sent text. Multiple `<div>` blocks collapse to a single space and `<p>` tags collapse to nothing, so both send as a run-on paragraph. Never dump the whole email into one flat `<div>`.
- **Subjects: reuse down the thread, vary across angles.** Two different axes, do not confuse them. Down a thread (step 1 to 2 to 3) the subject carries so the follow-ups stay threaded, and Apollo handles this for you: steps 2 and 3 are `reply_to_thread` with an empty subject. Across A/B variants that test different angles or offers, the norm is a *different* subject per variant, because a new angle usually earns its own subject line. Each lead still threads correctly under whichever subject it got on step 1. Only reuse the same subject across variants when you already know which subject lines win (usually after running outbound for a while), which is the exception, not the default.

### 6. Hand to a human to activate
Report the sequence id and a short step summary. Activation is `apollo_emailer_campaigns_approve`, and only a person runs it, after confirming the sender mailbox, the schedule, and the copy. Recommend, do not activate.

## Fully individual emails at scale (custom-field merge)

Standard merge tags (`{{first_name}}`, company, title) personalise the *surface* of a shared template. There is a second mechanism that personalises the *entire body*, and most people do not know it exists.

**Store each contact's own written email body in a long-text contact custom field, then reference that field by name as a merge variable inside the sequence step.** At send time Apollo resolves the variable per contact, so every enrolled person receives a genuinely different email while the sequence stays a single object with a single set of steps.

If you have a CSV with an `opener` or `email_body` column, that column is the input. The mechanic is: **column → custom field → merge variable in the template.**

### The four steps

**1. Create the custom field, once.** It must be **long-text / multi-line**, on the **contact** modality. `apollo_fields_index` only *lists* fields, so on MCP alone you cannot create one. Use `POST /fields` (REST, lanes 3 or 4) or the Apollo UI. Note both the field's **id** and its **name**: you need the id to write values and the name to reference it in copy.

**2. Write each contact's body into that field.** Pass `typed_custom_fields` keyed by **field id** on `apollo_contacts_create`, `apollo_contacts_bulk_create` (up to 100 per call), or `apollo_contacts_update` for contacts that already exist.

```json
{ "email": "…", "first_name": "…",
  "typed_custom_fields": { "<field_id>": "Saw you're hiring two SDRs in Austin — most founders…" } }
```

**3. Reference it by name in the step template.** In `body_html`, write `{{custom_email_body_seq_1}}` using the field's **name**, not its id. Mix it with normal tags freely:

```html
<div>Hi {{first_name}},</div><div><br /></div>
<div>{{custom_email_body_seq_1}}</div><div><br /></div>
<div>Worth a quick call?</div>
```

**4. Set the values before enrolling.** The merge resolves at send time, but a contact enrolled with an empty field will send an email with a hole in it.

### Cautions

- **A blank field renders as nothing**, silently. Verify coverage before activating: every enrolled contact needs a value, or the surrounding copy has to still read correctly without it.
- **Write the field so it fits the template.** If the template supplies the greeting and the CTA, the field holds only the middle. Decide that boundary once and write every value to it.
- **Approval moves up a level.** Nobody reads 800 individual emails, so review happens at the template and process level, with a spot-check across tiers. That is the same standard this library already applies to AI-assisted copy.
- **Individual bodies do not rescue a weak offer.** A thousand bespoke emails to a bad list is still a bad campaign, just a more expensive one.

### The alternative: Apollo's own AI opener

`emailer_touches` accepts `has_personalized_opener: true`, which has Apollo generate a per-recipient opening line, plus `generation_tone` (Direct, Formal, Casual) and a required `personalized_opener_fallback_option` (`skip`, or `use_generic` with a `generic_personalized_opener`). **It uses credits.**

Use it when you want personalization without doing the research yourself. Use the custom-field route when you want control over exactly what each person reads, which for a high-value list is usually the better trade.

## Editing a live sequence: the diff is destructive

`apollo_sequences_update` uses **declarative diff semantics**, and this is the single easiest way to destroy a sequence by accident. The `emailer_steps` array you send is the **full intended state after the call**, not a patch:

- A step **with** an `id` that matches → updated in place.
- A step **without** an `id` → created.
- An existing step whose `id` you **omit** → **deleted.**

The same applies per touch inside each step. So "change the subject line on step 2" means sending **every** step with its existing id, not just step 2. Send step 2 alone and you have deleted steps 1 and 3.

Three more traps in the same call:

- **`position` is required on every step** and must reflow as a complete 1..N.
- **`active` is required**, so mirror the current value unless you intend to change it. Passing `true` starts real sends.
- **`label_names` is a full replacement, not an append.** Read the current labels first (the search response returns `label_ids`, so resolve them via `apollo_labels_index`) and send the complete intended set. Passing only the new label erases the others.

**Always fetch current state first** (`apollo_emailer_campaigns_search`), show a before-and-after of what will be added, changed, removed, or reordered, and get a yes. If the sequence is active with contacts enrolled, say plainly that step changes can affect people mid-sequence.

## Optional: multichannel

Email-only is a complete campaign. If you want to add LinkedIn, call, or manual steps to this sequence and work the resulting task queue, see `../apollo-multichannel`. It is opt-in and does not change any of the rules above.

## Common mistakes

- Selling the outcome and the mechanism in one email. Pick one.
- A new pitch in step 3. It is for routing or a graceful exit.
- Absolute delays. Day 7 is `wait_time: 4` after a Day 3 step, not 7.
- Activating without a human. Never.
- A "personalized" opener that is generic ("love what you're doing at {{company}}"). That is theater, and it reads like it.
- Dumping the whole email into one `<div>`. It sends as a wall of text. Break it with `<br><br>` (see step 5).
