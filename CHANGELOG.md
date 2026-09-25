# Changelog

All notable changes to Apollo Operator are documented here. This project uses
[semantic versioning](https://semver.org/): MAJOR.MINOR.PATCH.

## v1.4.1 (2026-09-25)

**Kept current, four days after v1.4.0.** Apollo published MCP v2 on 2026-09-23, so the numbers
were re-measured against the live endpoints. The catalog moved under us in eleven days.

- **The v1 catalog is 85 tools and about 485 KB** (2026-09-25), up from 74 and 344 KB on
  2026-09-14. v2 is still four tools and about 6 KB, so the router's share of the preload fell from
  1.6% to 1.2% without Apollo changing v2 at all. Both dates now sit side by side in the lanes skill.
  The same search returned 7,829 on both endpoints.
- **Eleven formerly unpublished tools are now published and dispatch on v2:** record collections,
  fields, dynamic AI enrichment, data sources, and CSV exports. They are in v1's list, in v2's
  dispatch enums, and in Apollo's docs. The record-collection plan gate is now called "Sheets"
  (it was "AI Studio"); still not a plan you can buy. The lanes skill now describes the journey a
  tool takes from `find_tools` to the docs, instead of a fixed list of what is missing.
- **Still v1 with a master key only:** the nine agent tools, domain authentication diagnosis,
  domain purchase, and prompt suggestions. Agent reachability re-verified; the how-to agent
  answers product questions from Apollo's knowledge base in under ten seconds, free.
- **How to call a v2 dispatcher:** the action's own parameters go flat at the top level next to
  `action`, not nested. Nesting them fails with `Invalid params` and the schema does not say why.
- **Apollo's plugin ships five skills now**, not four: a `gtm-strategist` skill landed on
  2026-09-16 and overlaps this library's Targeting phase. Noted in Setup with the same
  complement-not-compete rule that applies to the agent tools.
- The people-search schema is about 19 KB today, not the 29 KB measured in July.

## v1.4.0 (2026-09-21)

**MCP v2, Apollo's own agent, and three new skills.** Apollo shipped a four-tool router in place of
the 74-tool catalog, opened the MCP to master API keys, and put its own AI agent behind nine MCP
tools. This release tests all three on a live account and rewrites the lane model around what held.
It also adds campaign reporting, an account audit anyone can run on their own Apollo, and
account-based outbound, plus a compliance reference, a plugin install path, and the corrections
that using v1.3 for real turned up. **20 skills, up from 17.**

### Kept current

Eight shipped claims were wrong by the time we checked, and are corrected with dates.

- **Sequences are readable now.** v1.3 said every read path returned empty bodies.
  `apollo_emailer_campaigns_show` now returns every step, touch, subject, and body, and
  `GET /emailer_campaigns/{id}` does the same with a master API key (the CLI's token still gets 403).
  Human approval before activation stays as policy, not as a workaround for a missing read.
- **MCP can stop a live sequence.** `apollo_sequences_update` with `active: false` works, with a
  trap: the update is declarative, so every step, touch, and template id you leave out is deleted.
  The CLI's `sequences abort` stays the recommended kill switch.
- **"Step 3 is the shortest email" is right after all.** v1.3 reversed it on one measured sequence.
  The rule as practised: step 3 is one or two sentences, three at most, and `sequence-reviewer`
  now checks for it.
- **Bulk contact creation never updates on a match**, whatever its description says. Single create
  does match on email, and **overwrites the existing record's fields** when it does. Search by exact
  email before any create, and never test with a real address.
- **Enrolling by `label_names` still fails** (`contact_ids` required), re-verified. Section 6b's
  two-call method is the path.
- **Opens, clicks, and replies are on the MCP.** `apollo_emailer_messages_search` returns per-sequence
  counts in `metadata_mode` and per-message events by id; v1.3 said REST only.
- **Advanced filters are not MCP-only.** NAICS, SIC, tenure, headcount growth, founded year, and
  market segments have no CLI flag, but lane 3 (REST with the CLI's token) has all of them.
- **Pagination overlap is query-dependent, not guaranteed.** Two more clean pulls (4,202 rows,
  4,202 unique). Keep deduping by id anyway; it is free.

### Added

- **MCP v2 is a router, and the library prefers it.** `https://mcp.apollo.io/mcp-v2` exposes
  `apollo_find_tools`, `apollo_read`, `apollo_write`, and `apollo_write_destructive`: 5.6 KB of
  schema instead of 344 KB, 1.6% of the preload, every v1 tool dispatchable, identical search
  totals. The hand-maintained tool map is demoted to a convenience and `find_tools` is the
  authority. The dispatcher is a free cost signal: anything routed through `apollo_write_destructive`
  spends credits or changes live state, so it gets a cost quote and the operator first.
- **Master API keys run the MCP unattended.** `X-Api-Key` on `/mcp` or `/mcp-v2` removes the consent
  prompt and the session expiry that v1.3 documented. Must be a master key; scoped keys are refused.
  Treat it like a password. On v1 it also reaches tools the router documents but will not dispatch.
- **The router is ahead of its dispatcher.** Record collections, CSV exports, dynamic AI enrichment,
  and domain authentication diagnosis appear in `find_tools`, return `not_implemented` on v2, and run
  on v1 with a master key. Record collections carry a third lock, an "AI Studio" access flag no plan
  offers. Reported to Apollo.
- **Apollo's own AI agent, tested and placed.** Nine `apollo_agent_*` tools are nine doors to one
  agent: prospecting, campaign planning, sequence copy, lead scoring, workflow automation, mailbox
  connection, call summaries, product how-to, and billing. Planning is free and it is good: it read
  the Context Center, planned four concepts, sized four audiences live, and stopped for confirmation.
  Reachable today only on v1 with a master key. The stance: complement, do not compete. Apollo's
  agent is strong at the front of the motion and absent from deliverability, verification,
  composition, suppression, and warmup. `apollo-icp-builder` and `apollo-sequence-builder` now say
  the option out loud where the overlap happens, and everything the agent returns still goes through
  List and Infrastructure before it sends. A new routing row sends product how-to questions to
  Apollo's own knowledge rather than re-explaining its UI.
- **`campaign-reporting`** (Iterate, new): results per sequence, step, variant, mailbox, seniority,
  and day of week, plus meetings booked and opportunities, read-only from
  `apollo_analytics_sync_report`, with `reply_class` as a first-pass intent sort. Verified against
  the Apollo UI. Traps: archived sequences vanish from search but keep reporting, and
  `num_emails_bounced` counts undelivered sends as bounces.
- **`apollo-account-audit`** (Foundations, new): 14 read-only checks on an existing account,
  mailbox limits, domain authentication, Bounce Guard per sequence, idle and unapproved sequences,
  lists, credit pools, Context Center, website tracker, duplicates. Graded, stops at findings. Its
  first run found an active sequence sending nothing because its touches were never approved.
- **`account-based-outbound`** (Targeting to Launch, new): when to reach several people at one
  company, how many, how it changes suppression and subject lines, and the account tools that
  carry it (`apollo_accounts_bulk_create` with `run_dedupe`, account lists, account-level analytics).
  Tested live.
- **Compliance reference**: opt-outs, suppression beyond Apollo, the workspace GDPR setting and
  `disable_eu_prospecting`, do-not-call screening and `dnc_status_cd`, recorded-call consent.
  Operating rules, not legal advice, wired into five skills and the audit.
- **Multi-angle strategy with live evidence**: timing signals decay, technographic and structural
  angles do not; one angle per campaign for clean attribution; the company overlap between angles
  measured on a real four-angle run.
- **Company search, measured.** 1 credit per call **including the empty page**, so stop on a short
  page. A company row carries `founded_year`, revenue, NAICS, SIC, and headcount growth over 6, 12,
  and 24 months without enrichment, so growth and sector grade off one free join. And a zero-credit
  preview sample for a prospect or client, from people search alone (`apollo-list-builder` 4b).
- **Enrich-late, re-verified.** Re-search before enriching a held-back tranche: 8.4% of candidates
  left a headcount-growth query in 18 days, and the search is free. `has_email` yield held at 99.7%
  a third time. The CLI batch loop verified again at 95 batches, zero failures.
- **Two targeting traps with numbers.** A title filter for `founder` also returns Founding
  Engineer, Designer, and GTM: 9 to 14% of a raw list. "Recently funded" includes acquisitions:
  34% of one funded universe was Merger / Acquisition. Pair the date filter with the funding type.
- **`sequence-reviewer` checks for empty merge fields** (no fallback means "Hi ,"), groups repeated
  findings, and no longer imposes house style. `apollo-sequence-builder` matches.
- **Install as a plugin.** `/plugin marketplace add creatop-gtm/apollo-operator` then
  `/plugin install apollo-operator@creatop`, from any directory, updatable. The repo now carries both
  the plugin layout (`skills/`) and the project layout (`.claude/skills/`), rendered from one
  source; never hand-edit either. About 1,400 always-on tokens per session.
- **Setup covers the MCP**, not only the CLI: connecting v1 or v2, Apollo's own plugin and its four
  slash commands (which sit beside this library), and the master key for unattended runs.
- **Conversations on the MCP** (`apollo_conversations_search`, `get_insights`, `get_transcript`),
  domain authentication diagnosis for domains with a mailbox in Apollo, `apollo_organizations_lookup`
  as the free way to resolve a company id, `apollo_organizations_enrich` at 1 lead credit, the
  tool map additions (labels, activity feed, feedback log, users), and the CLI routes for labels,
  emails, and mailboxes.

### Read these before your next launch

- **AI-written touches are silently dropped.** `generation_options: ai_full_email` is accepted on
  create and comes back as `generation_options: []` on read. No content, no credits, no error.
  Always read a touch back after setting it.
- **`apollo_write` is not a safety classification.** All nine agent tools are advertised as
  `apollo_write`, and one of them builds a list once confirmed. `apollo_write_destructive` reliably
  means spending. `apollo_write` means read what the tool does.
- **Domain diagnosis only sees domains with a mailbox connected to Apollo**, returns the last stored
  check rather than a live one, and is not a general DNS checker.
- **NAICS codes are 2 to 5 digits.** A 6-digit code fails with a bare `Invalid params`.
- **Scripts against `mcp.apollo.io` need a curl-style `User-Agent`**, or Python's `urllib` gets 403.

## v1.3.0 (2026-08-31)

**Call intelligence, credit control, and verify-first lists.** Three new capabilities, all of
which make a campaign cheaper to build and more likely to land: your own recorded calls become
the best research source you have, credit spend gets predictable and much lower, and
verification moves ahead of copywriting so you never personalise an address that cannot receive
mail. Everything here came from taking the full motion live on our own account, and five claims
that moved under us since v1.2 are corrected with dates. Still 17 skills.

### Kept current

Five things moved under us since v1.2. Corrected with dates, so you can tell what changed and when.

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

### Read these before your next launch

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

## v1.2.0 (2026-07-29)

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
- **`people search -f csv` is broken**: it emits the whole result array in one cell.
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

## v1.1.0 (2026-07-23)

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

## v1.0.0 (2026-07-06)

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
