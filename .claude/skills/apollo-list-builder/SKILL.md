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

## The order matters: filter and dedupe before you enrich

Enrichment costs credits (1 per person *requested*, matched or not). People search costs nothing, and company search costs 1 credit per call. So the pipeline spends as little as possible: narrow hard in search, filter to verified, dedupe, suppress, then enrich only the people you actually want. Never enrich first and filter later.

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

**People search is free. Company search is not.** Measured: `people search` left `lead_credit.consumed` unchanged across dozens of pages, while `companies search` cost **1 credit per call** (a 21-page company pull cost 21 credits). So paginate people as wide as you like, but treat company pulls as a real, if cheap, spend.

**Narrow to verified emails at search time, not after enriching.** `--email-status verified` (CLI) / `contact_email_status: ["verified"]` (MCP) filters to people Apollo already believes have a good address, and it costs nothing. In a real run this cut a 3,240-person universe to 2,742, which meant not paying to enrich ~500 people whose emails would have been thrown away anyway. Apply it in the search, and you never spend the credit in the first place.

**Pick your lane first** (see `apollo-operator`):

- **CLI**, when the filters are expressible as CLI flags and the list runs to hundreds or thousands. Results redirect to disk and never enter the model's context. A real run pulled 3,238 people into 4 MB on disk with nothing in the window, which simply does not fit over MCP.
- **MCP** (`apollo_mixed_people_api_search`), when the ICP needs a filter the CLI does not have: NAICS, SIC, founded year, headcount growth, tenure, years of experience, market segments, or department headcounts. The CLI is not a superset.

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

`industry` comes back **null** in search results on both lanes, so the obvious ICP-fit check is not available pre-enrichment. `naics_codes` and `sic_codes` **are** populated, and they are enough.

Pull the matching company universe (also free), join it to the people list on company name, and read the sector mix. That tells you how much of the list is genuinely your ICP before a single credit is spent. The full `jq` join is in `cli-recipes.md`.

In a real run this turned 3,238 raw people into a readable mix: 43% professional and technical services, 19% software and information, then a tail of finance, manufacturing, education, health care, and real estate. Education and health care were drift for a B2B offer and got cut; the core 60% became the High tier. Name-based joining matched 88%, so treat the unmatched remainder as its own bucket rather than dropping it.

This is the cheap version of the Targeting sample-validation gate, run over the whole list instead of a sample.

### 3. Dedupe, suppress, and cap concentration (before enriching)

Three separate jobs, all of which must happen before you pay for enrichment.

**Dedupe by person id, first, before anything else.** This is not optional and it is not obvious: **Apollo's pagination returns overlapping records across pages.** A real 33-page pull returned 3,238 rows containing only 2,933 unique people, a 9% duplicate rate. If you dedupe by company or apply a per-company cap before deduping by id, the duplicates survive into the final list and you pay a credit for each one.

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
- Partners, vendors, investors, and your own employees.
- Anyone already sitting in another live sequence.

The brief's Constraints section is where these live (see `business-brief`). Build the suppression list as a file of domains and email addresses, and subtract it from `deduped.csv` before enriching. If the business cannot tell you who is on it, that is a finding worth surfacing, not a step to skip.

**Apollo enforces a second layer at enrollment, and it is opt-in.** `sequences add-contacts` blocks these by default, and each is a flag you have to deliberately turn on: `--active-in-other`, `--finished-in-other`, `--same-company`, `--unverified-email`, `--no-email`, `--skip-verification`. Leave them off. A clean list still produces a bounce spike if somebody passes `--skip-verification` at enrollment. Your file-level suppression and Apollo's flags are complementary, not redundant: yours catches customers and unsubscribes, Apollo's catches double-touching.

**Cap concentration.** No more than 2 to 3 people per company unless it is deliberate ABM. One company flooding your list skews everything and looks like spam.

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

### 5. Enrich (the credit step)

Before you start, confirm the total scope out loud with the exact wording the tool requires: "This will enrich [N] people and consume up to [N] credits (1 credit per match, no charge for unmatched). Do you want to proceed?" Confirm the whole scope up front, do not drip-confirm batch by batch.

**The 85% rule: never quietly spend most of what they have left.**

Before enriching, compare the cost against the **remaining** balance, not the monthly limit. If the step would consume **more than 85% of remaining credits**, stop and put the choice in front of the operator with three explicit options:

> Enriching this list costs **2,712 credits**. You have **3,055 left**, so this would use 89% of them and leave you with 343.
>
> 1. **Downsize** — enrich the top N and keep the rest staged for later. The list costs nothing to hold.
> 2. **Skip** — leave the list at the free stage and come back when credits reset.
> 3. **Proceed** — spend it, knowing what is left.
>
> Which would you like?

This is not the same as asking permission to spend credits, which you already do. It is a **separate, louder gate for a spend that materially changes what they can do next.** An operator who has run outbound before will know 343 credits is nearly nothing. Someone doing this for the first time will not, and "you have 343 credits left" only means something once it is too late.

Do not compute this against the plan limit. 2,712 of a 4,000 limit sounds fine; 2,712 of 3,055 remaining does not, and the second number is the real one.

**Enrich as late as you can.** Enriched data has a shelf life: people change jobs, and a verified address goes stale. If sending is weeks away (a warmup clock still running, for instance), build and grade the list now and enrich when you are close to actually sending. The list costs nothing to hold; the enrichment does.

**Pick the method by size.**

- **Under ~20 people, in conversation:** `apollo_people_bulk_match` on MCP, passing the `id` from search (never the names). Max 10 per call, 1 credit per match.
- **Anything larger: use the CLI.** `apollo people bulk-enrich --file <batch.json>` takes a JSON array of match records, and `{"id": "<apollo person id>"}` is a valid record. Split the id list into batches of 10, loop, and write each response to disk. **Live-tested at 848 people across 85 batches: zero failures, exactly 848 credits, 33 fields per person, about 100 seconds.** The payloads never touch context.

```bash
split -l 10 enrich_ids.txt batches/batch_
for b in batches/batch_*; do
  jq -R '{id:.}' "$b" | jq -s '.' > "$b.json"
  apollo people bulk-enrich --file "$b.json" -f json > "$b.out.json"
  sleep 0.25
done
jq -s '[.[].matches[]?] | unique_by(.id)' batches/*.out.json > enriched_full.json
```

- **The MCP alternative, untested.** Apollo's MCP schema now steers 20-30+ person enrichments toward record collections (`apollo_custom_objects_create` plus `apollo_fields_create` plus `apollo_dynamic_field_enrichment_enrich`) and warns against looping `bulk_match` there, because that path has no persistence or resumability *inside a conversation*. The CLI loop above does not have that problem, because every batch is already written to disk and is trivially resumable. Prefer the CLI loop until the record-collection path has been verified.

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

### 7. Verify emails independently, before anything sends

Every email gets verified before it can be sent to. Unverified means bounce risk, and bounces kill domains.

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

### Optional: also put the list into Apollo (not just the CSV)

The CSV lives on the drive; it does not touch the Apollo account. To also have the list *in Apollo* (under Lists), create each enriched person as a contact with `apollo_contacts_create`, passing `label_names: ["<list name>"]`. The label is the Apollo List, and it appears in the UI.

Two hard-won rules here:
- **Bulk create: re-verify before trusting it.** In mid-2026 `apollo_contacts_bulk_create` silently produced *ghost* contacts (`existence_level: "none"`): invisible in the UI, unsearchable, no label attached, behind a confident success payload. Apollo has since reworked this surface, and the tool now documents deduplication (matching by email updates the existing contact) and accepts up to 100 per call. **We have not re-tested it.** Until someone does, either use single `apollo_contacts_create` looped, which is known-good, or run bulk create on a batch of 2 or 3 first and confirm with `apollo_contacts_search` that the records really landed.
- **Bulk create is destructive on a match.** Where it matches an existing contact, the values you send **overwrite** that contact's current fields, and that cannot be undone. Do not push a thin record over a rich one.
- **Verify, do not trust the response.** After creating, confirm with `apollo_contacts_search` (by name or keyword) that the contact actually landed. A success payload is not proof.

Apollo dedupes on create (it matches by email and updates the existing record rather than duplicating), and pushing enriched people into the account costs no credits, the enrichment already paid for the data.

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
- **Writing to the Apollo CRM before dedupe.** The bulk-create tools make a new record every time, duplicates included.
- **Flooding on a few big domains.** Cap per-company concentration.
- **Deduping the list against itself and calling it suppression.** Customers, open deals, and past unsubscribes are not duplicates, and they are the ones that actually cost you something.
- **Not deduping by person id after paginating.** Apollo's pages overlap, roughly 9% in a real run. Dedupe by `id` before the per-company cap, or you pay for the same person twice.
- **Handing over a pre-enrichment list without explaining it.** Empty email columns look like a broken tool to anyone who has not done this before. Name what is missing and why, every time. See step 4.
- **Enriching first and filtering to verified afterwards.** Filter with `--email-status verified` in the search and you never spend the credit.
- **Assuming a title filter returned the titles you asked for.** `founder` also matches Founding Engineer, Founding Designer, and Founding GTM. Always print the title distribution and write an explicit deny-pattern.
- **Trusting a data provider's catch-all flag.** Catch-all coverage varies between sources and can miss a large share. Verify with a dedicated service, every time, before sending.
- **Treating "Risky" as "bad".** Catch-all is not a bad address, it is an unconfirmable one. Decide based on how much sending reputation you have to risk, not on the label.
- **Looping a single-email verification endpoint over a whole list.** There is a bulk endpoint, and a web app, for exactly this. The loop is slower by an order of magnitude and gains nothing.
- **Guessing a rate limit instead of reading it.** Hitting a wall and then picking a cautious number below it leaves throughput on the table. The limit is usually published.
- **Looping `bulk_match` across dozens of batches.** No persistence, no resumability, no export. Use a record collection past ~20 to 30 people, and decide before you start.
- **Enriching weeks before you send.** People change jobs and addresses go stale. Build and grade early, enrich late.
- **Turning on the enrollment guard flags to make a number go up.** `--skip-verification` and `--unverified-email` convert a clean list back into a bounce spike.
