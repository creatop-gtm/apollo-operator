# Changelog

All notable changes to Apollo Operator are documented here. This project uses
[semantic versioning](https://semver.org/): MAJOR.MINOR.PATCH.

## v1.3.0 — 2026-08-31

The surface re-check. Everything in v1.2 was true when we shipped it; five weeks later some
of it was not. This release re-ran the audit against a newer CLI and a wider MCP surface, then
took the whole motion live on our own account: an angle built on filters the CLI still cannot
express, a thousand records enriched, independently verified, uploaded as a real Apollo list,
and a sequence written, built, and activated. Nine shipped claims were wrong. They are corrected
here, with dates, so you can tell what moved and when. Still 17 skills.

### Corrected

- **The flagship MCP-only filter example moved to NAICS.** `--department-headcount` landed in
  Apollo CLI v2.1.0. Eight of the nine filters we listed remain CLI-less, and a new MCP-only
  family appeared in person-level website visitors, so the four-lane model got stronger, not
  weaker. Check `--help` against the list rather than trusting it.
- **Per-contact personalization no longer needs REST.** `apollo fields create` exists as of CLI
  v2.1.0, so the whole flow runs on the CLI's own token with no API key.
- **`conversations` is a full CLI command group**, not REST-only.
- **"Step 3 is the shortest email" was backwards.** Measured across a real sequence, step 2 is
  the lightest touch; step 3 runs longer because it carries the routing question and the fallback.
- **Pagination overlap is no longer guaranteed.** A 43-page pull returned zero duplicates against
  a 9% rate in July. Keep deduping, it is free, but the warning is now dated on both sides.

### Added

- **Credits are eleven separate pools**, documented in full, with the ratio that matters: the AI
  pool is 200 times the size of the contact-data pool. Data is scarce, generation is not. Never
  say "credits" without naming the pool.
- **The pipeline is reordered**: search, filter, dedupe, suppress, enrich, **verify**, then write
  copy. Verification moved ahead of copy because personalizing an address that will never receive
  anything is pure waste. The evidence: 997 addresses a provider graded `verified` were 564 safe
  to send, 413 catch-all, and 3 that would have bounced.
- **The free `has_email` filter.** Search returns it on every row, before you spend anything.
  Records without it return nothing or a guessed address, and both still cost a credit. Filtering
  first took an enrichment yield to 99.7% verified.
- **Enrichment economics, measured.** You pay per record *attempted*, not per email found.
  Re-enriching a record you already paid for charges again. There is a hard cap of **10 records
  per call** and the CLI does not batch for you.
- **Getting a list into Apollo as an actual list.** `add_to_my_prospects` returns 403 to a script,
  so it is bulk create in checkpointed batches, then a second call to attach the list.
- **Bounce Guard**, Apollo's new account-wide auto-pause, including the trap: it does not evaluate
  below 200 emails in the window, so small sends and the first days of a ramp are unprotected.
- **Sequence build mechanics**: the exact `emailer_touches` payload shape, step ids being required
  on update, and the schedule fields that decide when anything actually sends.
- **Conversations as the best research source you already own.** `call_summary` returns structured
  objections, pain points, outcome, next steps, and pricing discussion, for zero credits. A website
  is how a business describes itself; a recorded call is how its buyers describe their problem.
- **First-party intent as a fourth kind of angle**, plus a table mapping every advanced filter to
  the kind of angle it builds.
- **Follow-up craft**: how the greeting tightens across a thread, and the four things to cut from
  every follow-up.

### Warnings you should read before your next launch

- **Bulk contact creation does not deduplicate**, despite documenting that it does. Records per
  email equal submissions per email. Never retry on a timeout, and dedupe your own payload.
- **Enrollment fails silently.** Passing several ids to `--contact-id` joins them into one string;
  Apollo finds nothing and returns exit 0 with an empty campaign. Assert on `contacts` and
  `skipped_contact_ids` in the response body, never on the exit code.
- **Sequences are effectively write-only.** Four read paths, all returning empty bodies. The last
  check before activation is a human opening the UI, so every skill that creates or updates one
  now hands you the link.
- **The MCP session can expire mid-task** while the CLI token keeps working. For anything long
  running, prefer the other lanes.

## v1.2.0 — 2026-07-29

The release that came from actually running the library end to end on a live account
rather than reading it. Most of what follows was found by use, not by review: the
2026-07-23 audit read every skill and caught none of it. 17 skills.

### Added

- **`re-engagement`** (Iterate): going back to the 90%+ of any list who finished a
  sequence and never replied. When it is safe (60 to 90 days), the permanent
  exclusions, why re-verification is mandatory, and the requirement to bring a
  genuinely new angle rather than another follow-up.
- **`operator-voice.md`** (Foundations reference): the communication standard, wired in
  as prime directive 7. How the operator introduces itself, five named response shapes
  (introduction, step report, gate, problem report, handoff), a jargon gloss table, and
  a ceremony-matches-stakes ladder. Written on the assumption the operator has never run
  outbound before.
- **`cli-recipes.md`** (Foundations reference): verified commands, response-shape
  gotchas, the credit table, and the enrollment guard flags.
- **`angles.md`** (Targeting reference): running several angles against one ICP. The
  three kinds (timing signals decay, technographic fit does not, structural facts never
  do), free angle sizing, one angle per campaign, and suppressing between them.
- **The 85% rule**: when a step would consume more than 85% of *remaining* credits, the
  operator stops and offers named choices (downsize, skip, proceed) instead of just
  quoting a number.
- **A required explanation gate** before enrichment: never hand over a free-stage list
  without saying what is missing from it and why.
- **Waterfall enrichment and phone reveal**: both asynchronous, both polled via
  `apollo_webhook_result_show`, both variable in cost. Capability-check first.
- **Per-contact email personalization**: store an individually written body in a
  long-text custom field and merge it by name into a sequence step.
- **Apollo Context Center**: `brief.md` can now be projected into Apollo's own team-wide
  ICP and product profiles.
- **Mailbox purchasing**: the MCP can now buy mailboxes (300 shared / 800 google / 1500
  outlook unified credits). It still cannot buy domains or set DNS.

### Changed

- **Numbered levels replaced with named phases** across all 17 skills. Foundations →
  Context → Targeting → Infrastructure → List → Message → Launch → Iterate, with Signals
  cross-cutting. **Infrastructure moved ahead of List and Message**, because warmup is a
  14 to 21 day wall-clock wait that has to run in parallel with list and copy work.
- **`apollo-cli` rebuilt around four lanes, not three.** The new one: the OAuth token
  from `apollo auth login` authenticates against the same REST endpoint the MCP uses, so
  you get the full MCP filter surface with disk output and no API key. Verified against
  MCP on an identical query.
- **The context-cost claim is now measured, and the obvious version of it was wrong.**
  CLI and MCP return byte-identical payloads. The saving is ~40 preloaded tool schemas
  and the ability to bypass context entirely, not leaner data.
- **`apollo-list-builder` rebuilt** as a 9-step pipeline: search → grade composition →
  dedupe, suppress, cap → explain and wait → enrich → score → verify independently →
  grade → hand off.
- **Verification is now vendor-neutral** and independent of the data provider. Use the
  bulk path rather than looping a single-address endpoint, read the published
  concurrency limit, cap retries, complete unordered.
- **`weekly-rhythm`** gained the volume math for scaling a winner and its three-week
  lead time.

### Fixed

- **Title filters silently over-match.** A search for `founder` also returns Founding
  Engineer, Founding Designer, and Founding GTM: 9 to 14% of a raw list. There is now a
  required check that the filter returned what you asked for, covering titles,
  duplicates, and sector.
- **Apollo's pagination overlaps.** 3,238 rows contained 2,933 unique people. Dedupe by
  person id immediately after merging pages, before any per-company cap.
- **Suppression now covers other lists**, not just the current one. Running several
  angles against one ICP overlaps silently, and every overlap is a duplicate charge and
  a double-touch.
- **`companies search` splits results across `.accounts` and `.organizations`**, with
  the ratio shifting by page. Reading one array silently loses most of the result set.
- **`people search -f csv` is broken** — it emits the whole result array in one cell.
  Use `-f json` and shape with `jq`.
- **`sequences abort` documented as the kill switch.** Available on the CLI and REST;
  **MCP is the only lane that cannot stop a live send.**
- **People search is free; company search costs 1 credit per call.** The previous
  blanket "search is free" was wrong.
- **Website-visitor filtering moved out of the UI** and into company search, so it is
  now a real signal rather than a referral to the interface.
- **Enrichment guidance corrected**: looping `bulk_match` across many batches is the
  approach Apollo now warns against. The CLI batch loop is verified at 848 and 2,712
  records; the record-collection alternative is documented but not yet tested.
- **Bulk contact creation softened**: the old ghost-contact failure appears to be fixed,
  but it is flagged for re-verification rather than trusted.

## v1.1.0 — 2026-07-23

Apollo shipped a CLI and formalized three ways to run headless (MCP, CLI, raw
API). This release makes Apollo Operator aware of all three, and adds the two
skills that were missing from the front and back of the motion: understanding
the business before targeting, and building the sending stack before sending.
16 skills total.

### Added

- **`apollo-cli`** (Level 0, Access): the three lanes to run Apollo headless
  (MCP, CLI, raw REST API), the MCP-vs-CLI tradeoff (context cost,
  composability, determinism), when Operator reaches for each, CLI setup
  (`apollo auth login`, OAuth), and a zero-credit `GET /v1/auth/health`
  preflight. The library stays MCP-native by default; CLI-backed variants of
  the search and enrich steps are planned for a future release.
- **`business-brief`** (Level 0.5): produces one readable `brief.md` capturing
  the business (identity, offer, buyers, pains, objections, proof, voice), the
  narrative context layer every other skill reads. Includes a fill-in
  `brief-template.md`.
- **`sending-infrastructure`** (Level 4, Setup): build the sending stack from
  zero, dedicated domains separate from your primary, mailboxes, DNS
  (SPF, DKIM, DMARC, MX), redirect, and warmup, plus the volume math for how
  many mailboxes and domains a given daily send target needs. Includes a
  `dns-records.md` reference and pre-send checklist.

### Changed

- **`apollo-multichannel`**: reframed the human-in-the-loop design as the
  LinkedIn safety advantage (you act from your own profile, not an API that
  risks a ban), and added explicit pacing: keep connection requests under 30
  per day, 20 to be safe, and trickle them across days rather than clearing
  the queue in one session.
- **`operator-context`**: added the three-lane access note and routing, the
  two infrastructure read tools (`apollo_domain_purchase_index`,
  `apollo_email_account_purchase_index`) to the tool map, and the two-layer
  context model (`brief.md` narrative + `profile.yaml` machine).
- **`apollo-icp-builder`**: reads `brief.md` first when it exists, and offers
  to build one before targeting.
- **`apollo-deliverability`** and **`apollo-go-live`**: point back to
  `sending-infrastructure` when no sending stack exists yet.

## v1.0.0 — 2026-07-06

The launch release. 13 skills turning the raw Apollo MCP into a guided
outbound motion, built and tested live on a real Apollo account.

### Added

- **`operator-context`**: the router. Apollo tool map, universal
  deliverability and copy rules, shared `profile.yaml`, routing to every skill.
- **`apollo-icp-builder`**: ICP into a real Apollo search plus custom scoring
  signals.
- **`apollo-list-builder`** and **`list-quality-scorecard`**: search, enrich,
  verify, and grade a list before it sends.
- **`apollo-sequence-builder`** and **`sequence-reviewer`**: write and review
  short sequences matched to the persona, built as inactive drafts.
- **`apollo-multichannel`**: optional LinkedIn, call, and manual steps plus the
  task queue.
- **`apollo-go-live`**: enroll, human approval, and pulling contacts.
- **`apollo-deliverability`**: mailbox health, warmup, and the bounce rules.
- **`positive-reply-scoring`**, **`experiment-design`**, **`weekly-rhythm`**:
  measure intent, run clean experiments, keep a cadence.
- **`apollo-signals`**: find the offer-fit reason to reach out now.
