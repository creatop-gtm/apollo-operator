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
- **Word count by step: 60 to 75 for step 1, 45 to 60 for step 2, 55 to 75 for step 3.** Step 2 is the lightest touch. Step 3 runs longer than people expect because it carries the routing question *and* the lower-commitment fallback.
- **One CTA, soft.** "Worth a look?" not "Book a 30-minute demo."
- **Never ask for a "quick call."** Hard rule, along with "hop on a call", "quick chat", "quick sync", and "jump on a call". "Quick" is the sender pre-apologising for their own ask: it says the request is an imposition, and no call has ever been quick anyway. Minimising the ask makes it easier to decline, not easier to accept. Use a real number or a question about interest instead: **"Worth 15 minutes?"**, "Worth a look?", "Open to hearing more?", "Want me to show you what that looks like?" Naming the actual time is fine. Advertising the smallness of the ask is not.
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
- **Step 2 (Day 3, same thread):** context and credibility, a different angle, a real case snippet, and **materially** shorter than step 1.
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

**The exact shape, verified live 2026-08-31.** Each step nests a template inside a touch. Sending the variants as `emailer_templates` on the step is rejected with `422 Step 1 (auto_email) requires a template. Include an emailer_touches entry with a template.`

```json
{"emailer_steps":[
  {"type":"auto_email","wait_time":0,"wait_mode":"day",
   "emailer_touches":[
     {"emailer_template":{"subject":"pipeline","body_html":"<div>…</div>"}},
     {"emailer_template":{"subject":"pipeline","body_html":"<div>…</div>"}}]},
  {"type":"auto_email","wait_time":3,"wait_mode":"day","emailer_touches":[…]},
  {"type":"auto_email","wait_time":4,"wait_mode":"day","emailer_touches":[…]}]}
```

Leave `subject` **empty on steps 2 and 3** so they reply in-thread rather than starting a new one. `--schedule-id` comes from `apollo sequences schedules`. A sequence created without `--active` lands inactive, which is what you want: create, verify, then activate deliberately.

**⚠️ You cannot verify the copy landed from any headless lane.** After a successful create, every read path reports `touches: 0` on every step: the create response itself, `apollo sequences search`, and the MCP campaign search. `GET /emailer_campaigns/{id}` returns **403** on the CLI's OAuth token, so there is no per-sequence read at all. The 422 above proves the API *validates* that templates are present, so a clean create means they were accepted, but nothing available to a script will show you the body text afterwards.

**So the last check before activating a sequence is a human opening it in the Apollo UI.** Do not skip it and do not treat a successful create as verification. The failure mode is sending blank or wrong-bodied emails to a real list, and it is unrecoverable once sent.

### Always hand back the sequence URL. Every create, every update.

Because the copy can only be verified by eye, **the operator needs a link, not an id.** End every create and every update by printing it:

```
https://app.apollo.io/#/sequences/<sequence_id>
```

Say plainly what it is for: *open this and confirm the bodies are right before anyone is enrolled.* An id alone makes the human go hunting, and a step they have to hunt for is a step they will skip. This is the whole handoff between what a script can do and what only a person can check, so make it one click.

(Apollo is prototyping sequence review and editing inside Claude and GPT. Until that ships, the link is the handoff.)

### Updating an existing sequence needs the step ids

`sequences update` replaces the full set of steps, and sending your original payload back will fail with `422 Missing Step 1 for this sequence.` Each step in the payload must carry the `id` of the step it is replacing.

Read them from `apollo sequences search`, which **does** expand `emailer_steps` with `id`, `position`, `type`, `wait_mode`, and `wait_time`, even though it never expands the touches. So step structure is readable and step content is not.

Two consequences. Keep the steps payload in a local file as the source of truth, because you cannot reconstruct the bodies from the API. And re-send **every** step on every update, with its id, since an omitted step is a deleted step.
### Shape the greeting to the step

The greeting tightens as the thread goes on, and this is what makes three emails read as one conversation rather than three separate approaches:

| Step | Shape |
|---|---|
| 1 | Block greeting on its own line, then a blank line, then the body |
| 2 | `Hi again {{first_name}},` with the body continuing **on the same line** |
| 3 | Tighter still, and it can drop the greeting word entirely: `{{first_name}}, are you the right person…` |

### Four things to cut from every follow-up

All four are the writer being visible in a message that should sound like a person typing:

- **Throat-clearing lines.** "Following up on this." / "Quick follow-up." / "Last one from me." These spend the most valuable line in the email on nothing, and `Hi again` has already said it.
- **Labelling phrases.** "How it works:" / "The part that matters is". Say the thing rather than announcing that you are about to.
- **Setup-and-contrast.** "Most vendors hand over a report. We hand over the asset." The contrast is your reasoning, not their need. Cutting it usually takes 20 to 30 words out of a follow-up and loses nothing.
- **The summarising payoff.** A sentence explaining a benefit you just stated ("That is why month six beats month one") is admiring your own point. Trust the reader.

Contractions are fine and usually help. Formal follow-ups read like documents.

- **Format the body so it breaks into paragraphs.** Write 2 to 3 short paragraphs, not one wall of text. Separate them with `<br><br>` inside a single `<div>`, like `<div>First paragraph.<br><br>Second paragraph.<br><br>Closing line and CTA.</div>`. This is not cosmetic: live-tested on Apollo, `<br><br>` is the only structure Apollo keeps as real line breaks in the sent text. Multiple `<div>` blocks collapse to a single space and `<p>` tags collapse to nothing, so both send as a run-on paragraph. Never dump the whole email into one flat `<div>`.
- **Subjects: reuse down the thread, vary across angles.** Two different axes, do not confuse them. Down a thread (step 1 to 2 to 3) the subject carries so the follow-ups stay threaded, and Apollo handles this for you: steps 2 and 3 are `reply_to_thread` with an empty subject. Across A/B variants that test different angles or offers, the norm is a *different* subject per variant, because a new angle usually earns its own subject line. Each lead still threads correctly under whichever subject it got on step 1. Only reuse the same subject across variants when you already know which subject lines win (usually after running outbound for a while), which is the exception, not the default.

### 6. Hand to a human to activate
Report the sequence id and a short step summary. Activation is `apollo_emailer_campaigns_approve`, and only a person runs it, after confirming the sender mailbox, the schedule, and the copy. Recommend, do not activate.

## Fully individual emails at scale (custom-field merge)

Standard merge tags (`{{first_name}}`, company, title) personalise the *surface* of a shared template. There is a second mechanism that personalises the *entire body*, and most people do not know it exists.

**Store each contact's own written email body in a long-text contact custom field, then reference that field by name as a merge variable inside the sequence step.** At send time Apollo resolves the variable per contact, so every enrolled person receives a genuinely different email while the sequence stays a single object with a single set of steps.

If you have a CSV with an `opener` or `email_body` column, that column is the input. The mechanic is: **column → custom field → merge variable in the template.**

### The four steps

**1. Create the custom field, once.** It must be **long-text / multi-line**, on the **contact** modality. `apollo_fields_index` only *lists* fields, so on MCP alone you still cannot create one. The cheapest route is now **`apollo fields create` (CLI, v2.1.0+)**, which needs no API key: before that release this step forced you onto `POST /fields` over REST, and that is no longer true. `POST /fields` (lanes 3 and 4) and the Apollo UI both still work. Note both the field's **id** and its **name**: you need the id to write values and the name to reference it in copy.

**Trap, verified 2026-08-31: the two surfaces return the id in different shapes.** `apollo fields create` returns it **modality-prefixed**, as `contact.66e34b81740c50074e3d1bd4`. `apollo fields custom` and `apollo_fields_index` return the same field **bare**, as `66e34b81740c50074e3d1bd4`. `typed_custom_fields` wants the bare form, and the MCP schema validates it against `^[a-f0-9]{24}$`, so the prefixed form is rejected outright. Piping the create output straight into a contact write therefore fails. Strip everything up to and including the dot, or just re-read the field from the index after creating it.

**2. Write each contact's body into that field.** Pass `typed_custom_fields` keyed by **field id** on `apollo_contacts_create`, `apollo_contacts_bulk_create` (up to 100 per call), or `apollo_contacts_update` for contacts that already exist.

```json
{ "email": "…", "first_name": "…",
  "typed_custom_fields": { "<field_id>": "Saw you're hiring two SDRs in Austin — most founders…" } }
```

**3. Reference it by name in the step template.** In `body_html`, write `{{custom_email_body_seq_1}}` using the field's **name**, not its id. Mix it with normal tags freely:

```html
<div>Hi {{first_name}},</div><div><br /></div>
<div>{{custom_email_body_seq_1}}</div><div><br /></div>
<div>Worth 15 minutes?</div>
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

**Kill switch, verified 2026-08-31.** `apollo sequences abort --id <id>` deactivates a running sequence. On a sequence that is already inactive it refuses cleanly with `422 The sequence is already inactive.` rather than erroring obscurely or silently succeeding, so it is safe to call speculatively when you are unsure of the state. Test it on the real sequence *before* activation, not after, so the abort path is proven while nothing is live.

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
