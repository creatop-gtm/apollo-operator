---
name: apollo-list-builder
description: "Build a clean, enriched, graded lead list on Apollo: paginated search, dedupe, credit-aware enrichment, scoring signals, email verification, then grade. Use to build the full prospect list after the ICP is set."
---

# Apollo List Builder (List)

Take the validated ICP from `profile.yaml` and turn it into a clean, enriched, graded lead list that is ready for a sequence. Search, enrich, apply the scoring signals, dedupe, then grade. Nothing goes to a sequence until it passes the scorecard.

## When to use

- Targeting is done: `profile.yaml` has a validated ICP, the Apollo filter set, and 1 to 3 scoring signals.
- You need the full list, not the 50-to-100 sample you already validated.

If `profile.yaml` has no validated ICP yet, stop and run **Targeting (`apollo-icp-builder`)** first. Building a full list on an unvalidated ICP is the most expensive mistake in outbound.

## The order matters: cheap filters first, expensive steps last

Every step costs more than the one before it, so the pipeline is ordered to throw records away while they are still free.

**Search → filter → dedupe → suppress → enrich → verify → write copy.**

People search costs nothing. Company search costs 1 credit per call. Enrichment costs **1 credit per person requested, matched or not**, so you pay for misses as well as hits. Independent verification costs a fraction of a cent per address. Copy and personalization are the most expensive step of all, in money and in tokens, because they are per-contact generative work.

Two rules fall out of that, and both are absolute:

- **Never enrich first and filter later.** Narrow hard in search, apply the free filters, dedupe, and suppress before a single credit is spent.
- **Never write copy before verification.** Personalizing an address that will never receive anything is pure waste, and an invalid contact is a list problem rather than a copy problem. Verify the enriched list, then write against only what survives. Apollo's own `email_status: verified` does not substitute for this, see step 5.

**The free filter most people miss:** people search returns a **`has_email` boolean** on every row, before you spend anything. Records with `has_email: false` are not worth enriching. Tested 2026-08-31 on a five-record sample from that bucket: three returned `unavailable`, meaning nothing at all, and two returned `extrapolated`, which is Apollo guessing `firstname@domain` rather than verifying it. All five still cost a credit. Filtering on the flag first took a 2,259-person list to 2,019 and the enrichment yield to **99.7% verified**, against a library assumption of roughly 85% attrition. It held on two more runs on 2026-09-14: **942 attempted, 939 verified (99.7%)**, and **490 of 490 (100%)**. Three for three.

**Enrichment does not remember what you already paid for.** Re-running three already-revealed people on 2026-08-31 cost three more credits. There is no discount for a record you have enriched before, which is exactly why the suppression step earns its place: skipping it means paying twice for the same person.

## Where the list lives (files on your drive)

Important: the list is not "in the model." Apollo's search returns records into the conversation, but you persist them to real files on the local drive, one per stage. This is so the list survives (you cannot hold thousands of rows in context), and so you can open any stage in a spreadsheet and see exactly what is happening. Convention:

```
lists/<campaign>/
  people_p*.json     # step 1: raw search pages, straight from the CLI
  people_all.json    # step 1: the pages merged, then deduped by id
  companies_all.json # step 2: company universe, for the NAICS composition check
  deduped.csv        # step 3: after dedupe, suppression, and per-company cap
  enriched.json      # step 5: FULL enrichment payload, every field
  enriched.csv       # step 5: the columns you actually want, generated from enriched.json
  scored.csv         # step 6: + scoring-signal tag and priority
  verified.csv       # step 7: after the independent verifier pass
  final.csv          # step 8: passed the scorecard, ready for a sequence
```

Each step reads the previous file and writes the next. Nothing lives only in the model, and you can inspect the work at any point. Raw people-search pages are free to regenerate, so they are the one thing you can safely delete; `enriched.json` is the opposite, because it is what the credits bought and it lives nowhere else.

**Route results to disk, never into the model.** On the CLI this is free and automatic: `> file.json` and the payload never touches context. On MCP it is a constraint you have to work around, because a 100-record search and even a 10-record enrichment each return a payload too big for the window, and the harness parks it in a file. Either way, process the file, do not read it into the conversation.

When you enrich, **preserve the full payload.** Apollo returns roughly 33 fields per person (emails, phones, LinkedIn, employment history, seniority, intent). Save the complete JSON (`enriched.json`), then generate whatever columns you want as a CSV view from it (`enriched.csv`), and let the user pick which to keep or drop. Never reduce to a handful of columns and throw the rest away: you paid credits for that data, and it does not live anywhere else unless you save it.

## Pipeline

### 1. Build the raw list (search)
Paginate 100 per page, up to Apollo's 50,000-record ceiling (500 pages). Collect each person's `id`. Search returns no emails and may mask last names, that is expected and fine, enrichment fixes it.

**Resolve company ids with the free tool.** `apollo_organizations_lookup` does fuzzy name lookup and returns shallow records (id, name, domain, website) at **no credit cost**. Use it whenever you only need to turn a name into an id. The paid company search is for when you genuinely need full firmographics back.

**People search is free. Company search is not.** Measured: `people search` left `lead_credit.consumed` unchanged across dozens of pages, while `companies search` cost **1 credit per call** (a 21-page company pull cost 21 credits). So paginate people as wide as you like, but treat company pulls as a real, if cheap, spend. **The empty page costs too** (measured 2026-09-14: pages 1 to 4 took the balance from 416 to 412, and page 4 returned zero rows), so stop paginating companies when a page comes back with fewer than `per_page` rows rather than fetching the confirming empty one.

**Narrow to verified emails at search time, not after enriching.** `--email-status verified` (CLI) / `contact_email_status: ["verified"]` (MCP) filters to people Apollo already believes have a good address, and it costs nothing. In a real run this cut a 3,240-person universe to 2,742, which meant not paying to enrich ~500 people whose emails would have been thrown away anyway. Apply it in the search, and you never spend the credit in the first place.

**Pick your lane first** (see `apollo-operator`):

- **CLI**, when the filters are expressible as CLI flags and the list runs to hundreds or thousands. Results redirect to disk and never enter the model's context. A real run pulled 3,238 people into 4 MB on disk with nothing in the window, which simply does not fit over MCP.
- **MCP** (`apollo_mixed_people_api_search`), when the ICP needs a filter the CLI does not have: founded year, tenure, years of experience, market segments, LinkedIn URL lookup, or person-level website visitors. NAICS, SIC, and headcount growth moved into the CLI in v2.2.0 (2026-10-28), department headcount in v2.1.0, so check `--help` before assuming a filter is still MCP-only, and sign in again after upgrading. The CLI is not a superset.

The CLI loop, with the exact `people_search_filters` from `profile.yaml`:

```bash
for p in $(seq 1 <pages>); do
  apollo people search <filters> --per-page 100 --page $p -f json > lists/<campaign>/people_p$p.json
  [ "$(jq '.people|length' lists/<campaign>/people_p$p.json)" = "0" ] && break
  sleep 0.4
done
jq -s '[.[].people[]]' lists/<campaign>/people_p*.json > lists/<campaign>/people_all.json
```

Never `-f csv` here: it dumps the whole array into one cell. Shape with `jq … | @csv` instead. Recipes: `cli-recipes.md`.

If the count is far above what you need, tighten filters rather than enriching a bloated list.

### Never trust that a filter returned what you asked for

This is the single most repeated lesson in this skill, and it shows up in three different disguises. Run all three checks the moment the pages are merged, before a credit is spent. They cost nothing.

**1. Titles.** Apollo's title matching is looser than it looks, and the failure is quiet: a search for `founder` also returns **Founding Engineer, Founding Designer, Founding GTM, Founding Recruiter, and Founding Member.** Those people are not the buyer and will not become the buyer. In real runs this was **9 to 14% of the list**. Nobody notices until a founding engineer replies asking why you are pitching them.

```bash
jq -r '.[].title' people_all.json | sort | uniq -c | sort -rn | head -40
```

Read that with your ICP in hand and write an explicit persona filter: an allow-pattern for the titles you want, plus a **deny-pattern for the near-misses**. The same trap exists for seniority, where `c_suite` sweeps in Chief Business Officer, Chief People Officer, and similar roles that may be irrelevant to your offer.

**2. Duplicates.** Pagination overlaps (see below). Compare row count to unique-id count.

**3. Sector.** Grade the NAICS mix (step 2). A keyword tag like `b2b saas` will pull in adjacent industries, some legitimate and some not.

The underlying rule: **a filter expresses your intent, it does not guarantee the result.** Print the distribution of anything you filtered on and read it before you pay for it.

### 2. Grade the composition before you spend anything

`industry` comes back **null** in search results on both lanes, so the obvious ICP-fit check is not available pre-enrichment. `naics_codes` and `sic_codes` **are** populated, and they are enough. The company payload also carries `founded_year`, `organization_revenue`, `primary_domain`, `linkedin_url`, and **headcount growth over six, twelve, and twenty-four months** (`organization_headcount_six_month_growth` and its `twelve_month` and `twenty_four_month` siblings), all without enrichment (verified 2026-09-14). So growth, age, and sector can all be graded by the same join, at 1 credit per 100 companies instead of 1 credit per company enriched. What it does not carry: `industry` and `estimated_num_employees`.

Pull the matching company universe (also free), join it to the people list on company name, and read the sector mix. That tells you how much of the list is genuinely your ICP before a single credit is spent. The full `jq` join is in `cli-recipes.md`.

In a real run this turned 3,238 raw people into a readable mix: 43% professional and technical services, 19% software and information, then a tail of finance, manufacturing, education, health care, and real estate. Education and health care were drift for a B2B offer and got cut; the core 60% became the High tier. Name-based joining matched 88%, so treat the unmatched remainder as its own bucket rather than dropping it.

This is the cheap version of the Targeting sample-validation gate, run over the whole list instead of a sample.

### 3. Dedupe, suppress, and cap concentration (before enriching)

Three separate jobs, all of which must happen before you pay for enrichment.

**Dedupe by person id, first, before anything else.** Cheap insurance, because **Apollo's pagination has been observed returning overlapping records across pages.** A 33-page pull in July 2026 returned 3,238 rows containing only 2,933 unique people, a 9% duplicate rate. A 43-page pull on 2026-08-27 returned 4,202 rows and 4,202 unique ids, and a 15-page pull on 2026-09-14 was also clean, so the behaviour is either fixed or query-dependent. Keep deduping: it costs nothing, and the failure mode when it does happen is paying a credit per duplicate. If you dedupe by company or apply a per-company cap before deduping by id, the duplicates survive into the final list and you pay a credit for each one.

```bash
jq 'unique_by(.id)' people_all.json > deduped_ids.json
```

Do this the moment the pages are merged. Then check it: `jq 'length'` and `jq '[.[].id]|unique|length'` should be the same number. If they are not, you have not deduped.

**Suppress against your other lists, not just this one.** If you are running more than one angle against the same ICP, the angles overlap, and the overlap is invisible unless you check for it. In a real run of three angles against one ICP: **85 people in the second list had already been enriched for the first**, and **19 more appeared in both of the new lists.** Every one would have been paid for twice and emailed twice.

Keep a per-business suppression file of every person id you have already enriched or enrolled, and subtract it from each new list before enriching. When two live angles both want the same person, assign them to the higher-intent one and remove them from the other, deliberately rather than by accident.

```bash
comm -12 new_ids.txt already_enriched.txt | wc -l   # what the overlap costs you
```

**Suppress against people you must never email.** Deduping the list against itself is not enough. There are people who fit the ICP perfectly and must still be excluded, and every one of them is a credit wasted and a relationship damaged:

- Existing customers, and anyone at a customer account.
- Open deals and active opportunities.
- Anyone who has ever unsubscribed, replied "not interested", or complained.
- EU-located people, unless someone has decided on purpose that the business prospects them. Check the workspace GDPR setting first: contact search responses expose it as `disable_eu_prospecting`. See `compliance.md`.
- Partners, vendors, investors, and your own employees.
- Anyone already sitting in another live sequence.

The brief's Constraints section is where these live (see `business-brief`). Build the suppression list as a file of domains and email addresses, and subtract it from `deduped.csv` before enriching. If the business cannot tell you who is on it, that is a finding worth surfacing, not a step to skip.

**Apollo enforces a second layer at enrollment, and it is opt-in.** `sequences add-contacts` blocks these by default, and each is a flag you have to deliberately turn on: `--active-in-other`, `--finished-in-other`, `--same-company`, `--unverified-email`, `--no-email`, `--skip-verification`. Leave them off. A clean list still produces a bounce spike if somebody passes `--skip-verification` at enrollment. Your file-level suppression and Apollo's flags are complementary, not redundant: yours catches customers and unsubscribes, Apollo's catches double-touching. **A workspace admin can add a third layer** (Settings, Outbound, Sequences, Enrollment criteria): verified email, phone, LinkedIn URL, no other active sequence, contact and account stages, owner. Those rules beat the call's flags, so a contact they keep out stays out whatever you pass (verified 2026-11-09; details in `apollo-go-live`).

**Test the guards rather than trusting them.** On 2026-09-14 the job-change guard skipped a contact with `contacts_with_job_change`, but the same-company guard let a second person from the same company into an inactive sequence. Enforce the per-company cap in the list file before enrolling; treat Apollo's guard as a second net, not the first.

**Cap concentration.** No more than 2 to 3 people per company unless it is deliberate ABM (see `account-based-outbound`). One company flooding your list skews everything and looks like spam.

### 4. STOP. Show the operator what they are looking at, and wait.

**This is a required gate, not a courtesy.** Everything up to here is free and unenriched, which means the list you are about to show has **no email addresses, no phone numbers, obfuscated last names, and no company industry or headcount.** Those fields are blank because nobody has paid for them yet, not because anything went wrong.

You know that. The person you are working with very likely does not. If you hand over a CSV full of empty columns without saying this, the reasonable conclusion is that the tool is broken, and the next thing that happens is either a lost user or a re-run that wastes credits.

So say it plainly, in this shape:

> Here is the list at the free stage: **1,759 people across 1,245 companies.** Nothing has been enriched, so **there are no emails yet** and last names are masked. That is expected at this point.
>
> What I can tell you now: 999 are High priority (IT services, software, consulting), 508 Medium, 252 Low. Companies are capped at 2 people each, and I cut education, health care, and real estate as a poor fit for your offer.
>
> **Please review the tiering and the cuts before I spend anything.** Once you are happy, enriching the 999 High-priority leads costs about 999 credits and returns emails, phones, and roughly 33 fields per person. You have 3,900 credits.

Three rules for this gate:

1. **Name the missing columns out loud.** Do not rely on the operator noticing, and do not rely on them knowing why.
2. **Give them something reviewable.** Tier counts, what you excluded and why, and the per-company cap. "Here is a CSV" is not a review.
3. **State the credit cost of the next step before taking it**, then wait for an actual yes.

The whole point of separating the free stage from the paid stage is that the operator gets to change their mind while it is still free. That only works if they understand what they are looking at.

### 4b. Giving someone a sample without spending

Sometimes the person who needs to see the list is not the operator: a prospect deciding whether to buy a list, a client approving a direction, a colleague checking the targeting. You can show them a real sample at zero cost, because people search already returns first name, an obfuscated last name, title, company, location, and the `has_email` flag on every row. Pull 25 to 200 rows, keep those columns, and hand it over labelled as a preview. Join the company universe for sector, founded year, and headcount growth and the sample is still near-free (1 credit per 100 companies). Only enrich the handful they ask to see in full, and say what that costs first. Built this way on 2026-09-14 for a real data prospect: a 200-row preview, then 25 enriched, then 10 verified, each stage a separate file.

### 5. Enrich (the credit step)

Before you start, confirm the total scope out loud with the exact wording the tool requires: "This will enrich [N] people and consume up to [N] credits (1 credit per match, no charge for unmatched). Do you want to proceed?" Confirm the whole scope up front, do not drip-confirm batch by batch.

**The 85% rule: never quietly spend most of what they have left.**

Before enriching, compare the cost against the **remaining** balance, not the monthly limit. If the step would consume **more than 85% of remaining credits**, stop and put the choice in front of the operator with three explicit options:

> Enriching this list costs **2,712 credits**. You have **3,055 left**, so this would use 89% of them and leave you with 343.
>
> 1. **Downsize**: enrich the top N and keep the rest staged for later. The list costs nothing to hold.
> 2. **Skip**: leave the list at the free stage and come back when credits reset.
> 3. **Proceed**: spend it, knowing what is left.
>
> Which would you like?

This is not the same as asking permission to spend credits, which you already do. It is a **separate, louder gate for a spend that materially changes what they can do next.** An operator who has run outbound before will know 343 credits is nearly nothing. Someone doing this for the first time will not, and "you have 343 credits left" only means something once it is too late.

Do not compute this against the plan limit. 2,712 of a 4,000 limit sounds fine; 2,712 of 3,055 remaining does not, and the second number is the real one.

**Hard cap: 10 records per enrichment call, and the CLI does not batch for you.** `apollo people bulk-enrich --file <path>` accepts a file of any length and passes it straight through, so a 1,000-record file returns `400 RECORD_LIMIT_EXCEEDED: cannot enrich more than 10 in a single request`. The cap is not in `--help`. Write the loop yourself, 10 per call, and checkpoint to disk after every batch so an interruption costs nothing: a 1,000-record run took about 100 seconds with zero failures. No credits are consumed by the rejected oversized call, which is correct.

**Enrichment recharges for records you already paid for.** Re-running three already-enriched people cost three more credits. There is no "already revealed" discount, which is the concrete reason the suppression step in section 3 earns its place rather than being hygiene: skipping it means paying twice for the same person.

**Enrich as late as you can.** Enriched data has a shelf life: people change jobs, and a verified address goes stale. If sending is weeks away (a warmup clock still running, for instance), build and grade the list now and enrich when you are close to actually sending. The list costs nothing to hold; the enrichment does.

**Timing-signal lists drift, so re-search before enriching a held-back tranche.** A headcount-growth universe lost **106 of 1,259 candidates (8.4%) in 18 days**: they no longer matched the growth filter. Re-running the search is free. Enriching from the stale file would have spent 106 credits on people who no longer carry the signal. Any list built on a timing signal (growth, funding, hiring, new in role) gets a fresh search immediately before enrichment, and only the intersection is enriched.

**Pick the method by size.**

- **Under ~20 people, in conversation:** `apollo_people_bulk_match` on MCP, passing the `id` from search (never the names). Max 10 per call, 1 credit per match.
- **Anything larger: use the CLI.** `apollo people bulk-enrich --file <batch.json>` takes a JSON array of match records, and `{"id": "<apollo person id>"}` is a valid record. Split the id list into batches of 10, loop, and write each response to disk. **Live-tested at 848 people across 85 batches: zero failures, exactly 848 credits, 33 fields per person, about 100 seconds.** Re-verified 2026-09-14 at 95 and 49 batches, zero failures, credits equal to records attempted. The payloads never touch context.

```bash
split -l 10 enrich_ids.txt batches/batch_
for b in batches/batch_*; do
  jq -R '{id:.}' "$b" | jq -s '.' > "$b.json"
  apollo people bulk-enrich --file "$b.json" -f json > "$b.out.json"
  sleep 0.25
done
jq -s '[.[].matches[]?] | unique_by(.id)' batches/*.out.json > enriched_full.json
```

- **Apollo's own route: `apollo_agent_bulk_enrich`.** For a search's results or more than 10 named people, Apollo's agent picks the source (waterfall where the team has it), quotes the credit cost, pauses for approval, and saves the enriched people as contacts. It replaces the record-collection route Apollo's tool descriptions recommended in September, which left the MCP tool list by 2026-11-08 without ever opening on our account. Offer it as an option, with two things to say out loud: it saves people as contacts, so the create-and-dedupe rules below apply to what it writes, and its results arrive through the conversation rather than in files on disk. **The CLI loop above stays the default for anything you want to resume, re-run, or grade off-context**, because every batch is already on disk. Not yet run end to end here.

**Check the response, do not assume.** Each call returns `credits_consumed`, `total_requested_enrichments`, `matches`, and `missing_records`. Report the real `credits_consumed`, summed across batches.

**Waterfall enrichment, when Apollo's own data comes up empty.** `apollo_people_bulk_match` accepts `run_waterfall_email` and `run_waterfall_phone`, which fill *missing* fields by cascading through partner data providers, stopping at the first hit per person. Three things make this different from a normal enrichment and all three matter:

1. **It is asynchronous.** The response carries `waterfall.status: "accepted"` and one `request_id` for the whole batch, with no data inline. You poll `apollo_webhook_result_show` with that id, backing off up to about three minutes.
2. **The cost is variable and plan-dependent.** Zero when Apollo's own data satisfies it, otherwise partner credits that vary by plan and can exceed a standard match. **Never quote a fixed number.** Say it is variable, and get the operator to accept that before running it.
3. **It has to be enabled.** Call `apollo_users_api_profile` with `include_waterfall_capability=true` and read `waterfall_email_enabled` / `waterfall_phone_enabled` first. If the team is not configured for it, `waterfall.status` comes back `failed` and you have wasted a round trip.

Same shape applies to `reveal_phone_number`. **Before initiating any async reveal, confirm `apollo_webhook_result_show` is actually in your available tools**, because without it you will spend credits on a result you cannot collect. If it is missing, tell the operator to reconnect their Apollo authorization rather than proceeding.

For an email-first outbound motion none of this is needed. Reach for it when a list is important enough that the misses are worth paying a premium to fill, not as a default.

Credit reality, learned the hard way: Apollo may bill **every requested record, not just the matches** (a 10-person batch with 7 matches charged 10 credits, not 7). Do not promise "unmatched are free." Read the actual `credits_consumed` field in the response and report the true number. Enriching costs credits; creating contacts or pushing them into Apollo afterward does not, once a lead is enriched, moving that data around is free.

### 6. Apply the scoring signals
From `profile.yaml`:
- **Filter signals** were already applied in search (step 1), so they are done.
- **Research signals** get applied now, per company. Claude reads the company site and decides (e.g. "has a public pricing page"), or you pull a firmographic signal with `apollo_organizations_enrich`, or a hiring signal with `apollo_organizations_job_postings` (1 credit per org, so use it only when the signal is worth it). Tag each lead by priority: **High** (hits the signal), **Medium** (fits the ICP but not the signal), **Low** (edge of the ICP). Write the tier to `scored.csv`. Work the High-priority leads first.

### 6b. Getting the list into Apollo as an actual list

A list that only exists as a CSV on your drive is not a list your client can see, and it is not something a sequence can be pointed at. Finish the job in Apollo. **It takes two calls, not one, and the obvious path is closed.**

**`mixed_people/add_to_my_prospects` is not available to a script.** This is the endpoint the UI uses for "save these search results", and it returns `403 API_INACCESSIBLE` on the CLI's OAuth token. Verified 2026-08-31, and it is the only endpoint found so far that this token cannot reach, so lane 3 is *not* a complete superset of the UI.

That leaves `contacts/bulk_create` as the only programmatic way in, and over REST it does not dedupe unless you ask (see the rules below). So:

1. **Create the contacts in checkpointed batches of 100**, recording each batch as complete *before* issuing the next call, and send `run_dedupe: true` on REST. Without it, a naive re-run after a mid-way failure duplicates everything it already created. The checkpoint is what makes a re-run safe either way.
2. **Attach the list afterwards** with `labels/add_entity_ids_to_label_names`, passing `entity_ids`, `label_names`, and `modality: "contacts"`. `label_names` on bulk create is silently ignored, so this second call is mandatory, not optional.
3. **Confirm by count.** `apollo labels list` returns `cached_count` per list. It should equal the number you uploaded.

On the CLI, `apollo labels add` does step 2 in one command, by list name, and creates the list if it does not exist yet.

A real run: 564 contacts in 6 batches, then 6 label calls, verified at `cached_count: 564`.

### 7. Verify emails independently, before anything sends

Every email gets verified before it can be sent to. Unverified means bounce risk, and bounces kill domains.

**The measured version of why this step exists, 2026-08-31.** A list of **997 addresses that Apollo returned as `email_status: verified`** was run through an independent verifier. The result:

| Verdict | Count | Share |
|---|---|---|
| Safe to send | 564 | 56.6% |
| Risky (catch-all) | 413 | 41.4% |
| Unknown / unreachable | 16 | 1.6% |
| **Invalid, would bounce** | **3** | 0.3% |

**Apollo called every one of those 997 verified, and only 57% were actually safe to send.** The 41% catch-all share is normal and expected. The three **Invalid** are the point: they were graded `verified` by the provider and would have bounced. Three bounces inside a small first batch is a live deliverability incident on a young domain, and no amount of good copy prevents it.

So "narrow to verified at search time" (step 1) and "verify independently" (this step) are **not** the same control and neither replaces the other. The first stops you paying to enrich rubbish. The second stops you sending to it.

**Apollo now has a validation check of its own, and it is a different control from `email_status`.** `apollo_email_validation_validate` (published by 2026-11-08) sends addresses you already have to an outside validation vendor and reports `validated` or `not_validated`, naming the validator that answered. Three facts decide where it fits:

- **It costs up to 1 lead credit per address**, sometimes 0, from the same scarce pool as enrichment, and a repeat check inside the cache window is still charged. A dedicated verification service costs a fraction of a cent per address.
- **It runs 10 addresses per call, asynchronously**, through `apollo_webhook_result_show`. The tool asks you to say an exact confirmation sentence with the count before running it.
- **The same check is on the public API, lane 4 only.** `POST /api/v1/enrich` with `action: "email_verifier"` (or `phone_verifier`), `action_type: "tool"`, up to 10 `input_attributes`, and `poll_only: true`; then poll `GET /api/v1/webhook_result/{request_id}` until it stops answering `404 result_pending` (it suggests 10 seconds between polls; about 25 seconds in all on 2026-11-09). It needs a master key or an API key with `api/v1/enrich/create` access; the CLI's OAuth token is not accepted, so lane 3 cannot run it. Verified 2026-11-09: 11 rows or a missing delivery option are refused with a free `422`; a malformed address is accepted, costs 0, and counts in `total_failures`; an example.com address cost 1 credit and came back `not_validated`. Read `data_sources.success_step.raw_vendor_status` as well as the verdict: it said `invalid` there, which separates a dead address from a check that reached no verdict. That field was seen on one address only, so test it on your own catch-alls before relying on it.
- **`not_validated` is not "invalid".** It also covers catch-all domains and checks that reached no verdict, so it does not separate the three buckets below.

**Measured against an independent verifier, 2026-11-08.** Ten addresses an independent verifier had already graded, run through Apollo's check (the vendor that answered was LeadMagic). It cost **9 lead credits for 10 addresses**.

| Independent verifier said | Count | Apollo said |
|---|---:|---|
| Safe to send | 4 | validated, all four |
| Invalid (bounce) | 2 | not validated, both |
| Risky (catch-all) | 3 | **validated, all three** |
| Unknown (unreachable) | 1 | **validated** |

They agree on the clear cases. They part on exactly the addresses this step exists for: **Apollo's check returned `validated` for every catch-all and the unreachable one.** Its vendor may resolve some catch-alls that a standard verifier cannot, or it may be generous; ten addresses cannot tell you which, and only your own bounce data will. What it does settle: Apollo's check does not flag catch-alls, so it cannot be the control that holds them back on a new sending stack. **Apollo confirms that reading (2026-11-10):** `validated` means the address was verified, and a verified address can still sit on a catch-all domain, so catch-all is a separate signal that `validation_status` does not carry. A catch-all flag next to it is under consideration; until one appears in a response, get that signal from your own verifier.

So use it where it is the convenient tool: a handful of addresses inside a conversation, an account-based list of 30 named people, a spot-check before re-engaging a small segment. For a full list, the dedicated service below stays the deciding vote, on cost and because it sorts catch-alls and unknowns apart.

**Use a dedicated email-verification service, and treat it as the deciding vote.** Any data provider's own email status, Apollo's included, is a useful first filter: narrowing to verified at search time (step 1) is what stops you paying to enrich addresses you would throw away. But a provider grading its own data is not an independent check, and the two layers answer different questions. The provider tells you the address exists in its database. A verifier tells you what the receiving mail server will actually do today.

This library does not pick a verifier for you. Several good ones exist, they all return roughly the same categories, and which one you use is your call.

**Pay particular attention to catch-all detection.** A catch-all domain accepts mail to any address, so the server cannot confirm whether a person exists there. Coverage of catch-alls varies a lot between sources, and a provider flag saying a domain is clean is not a substitute for checking. In practice a **30 to 40% catch-all share is normal for B2B lists**, so a result in that band is not a sign of a bad list, it is a sign that you finally measured it.

**The four buckets, and what to do with each:**

- **Deliverable / safe to send:** send.
- **Risky / catch-all / accept-all:** a judgment call that depends on the age of your sending stack. On a brand-new stack with no reputation, hold them and launch on the confirmed addresses only. Once the domains have a track record, send to them as a separate later batch so any bounce damage is contained and attributable to a known cause.
- **Unknown / unreachable:** the server was temporarily down. Re-verify in a few days rather than sending blind.
- **Invalid / undeliverable:** drop, and never let it near a sequence.

Also read past the headline verdict. Most verifiers flag **role accounts** (`info@`, `sales@`, `contact@`) separately. They are usually deliverable and usually poor outbound targets, so they are worth removing even though nothing is wrong with them.

**Verify in bulk. Do not loop a single-email endpoint over a list.**

Verification services generally expose two paths: a real-time single-address check and an asynchronous bulk job. They are for different work, and using the wrong one is slow in a way that is easy to miss. A single-address check performs a live SMTP probe, so it costs on the order of a second or two per address no matter how fast your code is, and providers cap how many you may run at once.

- **Under ~100 addresses:** the single endpoint is fine.
- **Anything larger:** use the bulk job (upload the list, poll or receive a webhook, download results), or simply upload the CSV in the vendor's own web app. For a one-off list, the web app is often the fastest route available and needs no code at all. Do not write a loop out of habit.

If you do loop a single-address endpoint, four things save real time:

- **Read the vendor's published concurrency limit** instead of discovering it by hitting the wall and then guessing something safely below it. Guessing low leaves throughput on the table.
- **Expect rejections that are not HTTP errors.** A concurrency refusal often arrives as a `200` with a valid-looking body carrying a failure flag. Naive code stores that as a verdict. Treat anything that is not an explicit success as a retry, never as an answer.
- **Cap the retry ladder.** A long ladder on a slow timeout means one pathological address can block a queue for minutes. Retry two or three times, then set it aside and continue.
- **Complete unordered, and make the run resumable.** Ordered processing stalls all progress behind the single slowest item. One result file per address, skipping any that already succeeded, means an interrupted run costs nothing to restart.

### 8. Grade before you ship
Run **`list-quality-scorecard`** on the finished list. If it grades below B, fix the top issues and re-grade. Do not hand a C-grade list to a sequence.

### 9. Hand off
A clean, enriched, verified, graded list, with scoring tags, ready for **Message (copy and sequences)**.

### Rules for every contact write

The way to put a list into Apollo is section 6b: create the contacts in checkpointed batches, then attach the list with a separate label call. `label_names` on create is silently ignored, so never rely on it. The rules below apply to every contact write, 6b included.

Two hard-won rules here:
- **Bulk create dedupes on one surface and not the other, and updates on neither. Verified 2026-09-29.** On MCP, `apollo_contacts_bulk_create` now matches by email: sent twice, the second call returned the same ids under `existing_contacts`. But its description promises that a match *updates* the existing contact, and it does not: a changed `last_name` came back unchanged. Over REST, `POST /contacts/bulk_create` **creates a new record on every call** unless you send `run_dedupe: true`, which Apollo's REST docs now say plainly. (On 2026-08-31 neither surface deduped: the same five contacts sent twice made two records per email.) So a pipeline that moves from MCP to a script starts creating duplicates silently, and every retry adds another. Two consequences: **never retry a REST bulk create without `run_dedupe: true`**, and **dedupe your payload before sending** either way. If you need certainty that a record is updated rather than added, search for it first and use `apollo_contacts_update` on the id you find. The historical ghost-contact failure (`existence_level: "none"`, invisible in the UI) does appear to be fixed: records now land properly with `person_id` and `organization_id` populated and are findable by `contacts/search`. The duplication is a separate and current defect. Also verified in the same run: **`label_names` on bulk create silently does nothing.** No label was created and none was attached. Add contacts to a list afterwards with `apollo_labels_add_entity_ids_to_label_names` instead.
- **Bulk create is destructive on a match.** Where it matches an existing contact, the values you send **overwrite** that contact's current fields, and that cannot be undone. Do not push a thin record over a rich one.
- **Single create behaves differently from bulk create, and that is its own trap. Verified 2026-09-14.** `apollo_contacts_create` does match on email: it returned `was_existing: true` and updated the existing record rather than adding a second one. But it **overwrote that record's first and last name** with the test values sent. A keyword search for the person beforehand had returned nothing, so an empty search is not proof the contact is absent. **Never create a test contact with a real person's address**, including your own; use an address nobody's record carries, or search by the exact email first.
- **Verify, do not trust the response.** After creating, confirm with `apollo_contacts_search` (by name or keyword) that the contact actually landed. A success payload is not proof.
- **Check for duplicates the API cannot see.** `apollo_contacts_find_potential_duplicate` takes one contact id and returns at most one likely duplicate, free. It matches on more than email: on 2026-11-08 it paired a contact created in August with a 2023 record carrying a different address, which no email dedupe would ever catch. Run it on a sample after a large import. If it finds pairs, **show them to the operator and stop**: `apollo_duplicate_contacts_merge` permanently deletes the duplicates (and, with CRM deletion sync on, their Salesforce or HubSpot records), needs the Admin role, and must never run in parallel. Which record survives is the operator's call, not yours.

Pushing enriched people into the account costs no credits: the enrichment already paid for the data.

## Sourcing beyond Apollo

This library keeps sourcing strictly in Apollo, because that is what it can automate. Other real sources exist (Sales Navigator, Google Maps for local SMBs, trade-show exhibitor lists, industry directories, government databases). They are manual and out of scope here. See `references/sourcing-beyond-apollo.md` for when each is worth the manual effort.

## In action: a real run

What actually happens when someone says "build my list":

1. **Check.** Claude confirms `profile.yaml` has a validated ICP. It does.
2. **Pick the lane.** The filters are titles, seniority, employee range, location, and a hiring signal, all expressible on the CLI. So: CLI, straight to disk.
3. **Search.** Paginates 33 pages at 100 per page with `--email-status verified`, writes `people_all.json`, dedupes by id. Reports: "3,238 rows, 2,933 unique people after removing Apollo's page overlap. 4 MB on disk, zero credits, nothing through context."
4. **Grade composition.** Pulls the company universe (1 credit per page), joins on company name, reads the NAICS mix. Reports: "60% IT services, software, and consulting. 5% education and health care, which is a poor fit for your offer. 12% unmatched." You approve cutting the drift.
5. **Suppress and cap.** Subtracts the suppression file (customers, open deals, past unsubscribes), caps at 2 per company, writes `deduped.csv`. Reports: "1,759 left across 1,245 companies."
6. **Score.** Tags each row High, Medium, or Low against the signals in `profile.yaml`. Reports: "999 High, 508 Medium, 252 Low."
7. **Stop and explain (step 4 gate).** "Here is the list at the free stage. **There are no emails in it yet** and last names are masked, which is normal, because nothing has been enriched. 999 are the strongest fit. I cut education and health care, and capped each company at 2 people. Review the tiering before I spend anything. Enriching the 999 costs about 999 credits of your 3,900." You review, and say go.
8. **Enrich.** Batches of 10 through `apollo people bulk-enrich`, full payloads to `enriched.json`. Reports the real summed `credits_consumed`, not an estimate.
9. **Verify independently.** Runs the enriched addresses through your email-verification service. Reports: "504 deliverable, 327 catch-all (the mail server accepts any address, so nobody can confirm the person exists), 14 unreachable, 1 invalid. Catch-all around a third is normal for B2B. I would drop the invalid, re-check the unreachable in a few days, and hold the catch-alls until your domains have a track record."
10. **Grade.** Runs `list-quality-scorecard`, writes `final.csv`.

You end with `final.csv` on your drive: clean, verified, prioritized, ready for Message. Every intermediate file is still there if you want to check the work.

## Common mistakes

- **Enriching before deduping and filtering.** You pay credits for leads you were going to cut.
- **Building 50,000 leads on an ICP you never sampled.** That is a Targeting failure showing up as a List bill.
- **Skipping verification.** One bouncy list can take a domain down.
- **Writing to the Apollo CRM before dedupe.** REST bulk create makes a new record every time unless you send `run_dedupe: true`, and MCP bulk create matches on email only. Dedupe the payload yourself.
- **Flooding on a few big domains.** Cap per-company concentration.
- **Deduping the list against itself and calling it suppression.** Customers, open deals, and past unsubscribes are not duplicates, and they are the ones that actually cost you something.
- **Not deduping by person id after paginating.** Apollo's pages overlapped by roughly 9% in a July 2026 run, and later pulls were clean. Dedupe by `id` before the per-company cap either way, or you risk paying for the same person twice.
- **Handing over a pre-enrichment list without explaining it.** Empty email columns look like a broken tool to anyone who has not done this before. Name what is missing and why, every time. See step 4.
- **Enriching first and filtering to verified afterwards.** Filter with `--email-status verified` in the search and you never spend the credit.
- **Assuming a title filter returned the titles you asked for.** `founder` also matches Founding Engineer, Founding Designer, and Founding GTM. Always print the title distribution and write an explicit deny-pattern.
- **Trusting a data provider's catch-all flag.** Catch-all coverage varies between sources and can miss a large share. Verify with a dedicated service, every time, before sending.
- **Treating "Risky" as "bad".** Catch-all is not a bad address, it is an unconfirmable one. Decide based on how much sending reputation you have to risk, not on the label.
- **Looping a single-email verification endpoint over a whole list.** There is a bulk endpoint, and a web app, for exactly this. The loop is slower by an order of magnitude and gains nothing.
- **Guessing a rate limit instead of reading it.** Hitting a wall and then picking a cautious number below it leaves throughput on the table. The limit is usually published.
- **Looping `bulk_match` across dozens of batches in a conversation.** No persistence, no resumability, no export. Past ~20 to 30 people, use the CLI batch loop, which writes every batch to disk. Record collections are the path Apollo's tool descriptions point to, but they are not reachable on most accounts (see `apollo-operator`).
- **Enriching weeks before you send.** People change jobs and addresses go stale. Build and grade early, enrich late, and re-run the search first if the list rests on a timing signal.
- **Turning on the enrollment guard flags to make a number go up.** `--skip-verification` and `--unverified-email` convert a clean list back into a bounce spike.
