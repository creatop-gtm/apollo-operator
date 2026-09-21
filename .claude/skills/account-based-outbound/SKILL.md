---
name: account-based-outbound
description: "Run outbound against a named set of target accounts, or build a Dream 100 when there is no list: resolve accounts for free, map 2 to 3 people per account by default (or everyone, free or enriched), one sequence per persona, and stop the whole account on a positive reply. Use for high-value accounts, client target lists, or any time the operator wants to work specific companies."
---

# Account-Based Outbound (Targeting to Launch)

Most of this library works person by person: build a large list, one or two people per company, and let volume find the buyers. Account-based outbound turns that around. You start from a named set of companies you actually want, and you reach several of the right people inside each one, on purpose.

It uses the same pipeline as everything else (suppress, verify, write, approve, launch). What changes is where the list comes from, how many people per company, how their sequences relate, and what happens when one of them replies.

## When to use

Whenever the operator wants to work a named set of accounts: a short list of high-value customers, a client-supplied target list, a territory, or a known account that just showed a signal.

**No list to start from?** Build a Dream 100 (section 1). It is the fastest way to turn "we want better customers" into something you can run.

## 1. The target accounts

### If the operator already has a list

Resolve every company to an Apollo organization id with the **free** lookup: `apollo_organizations_lookup` with `q_organization_fuzzy_name` (a name, or better, a domain) and `display_mode: "fuzzy_select_mode"`. Verified free on 2026-09-14: lead credits did not move.

**Confirm every match by eye.** Fuzzy matching returns look-alikes (a lookup for "Gong" also returned "Gong cha Global"), and the domain on the right row can still be wrong (it came back as a preview-site URL). Resolve by domain wherever you have one.

**Saved accounts:** `apollo accounts search --label-ids <list id>` (CLI) returns a saved account list with each account's `organization_id`, `num_contacts`, and a per-account campaign status tally. The MCP has no saved-accounts search; `apollo_mixed_companies_search` returns saved accounts in its `accounts` bucket but costs 1 credit per call.

**Use `organization_id` in every filter, never the account's own `id`.** On saved accounts the two always differ, and an account id passed to an organization filter silently matches nothing.

### If they don't: build a Dream 100

The 100 companies the business would most like to have as customers. **Chosen, not scraped.**

1. **Start from the ICP.** Read `profile.yaml` and `brief.md` (see `apollo-icp-builder` and `business-brief`).
2. **Pull candidates for free.** `apollo_organizations_lookup` takes the same firmographic filters as the ICP: location, employee range, NAICS and SIC codes, keyword tags, funding, active job titles, technologies, and website visitors. Verified 2026-09-14: 25 companies per page, no overlap between pages, zero credits. Rows are shallow (id, name, domain, website) and there is no total count, so page until you have a few hundred candidates.
3. **Cut to 100 with the operator.** Fit, likely deal size, logos they want, relationships they already have. Claude can read each company's site and score it against a research signal (`apollo-list-builder`, step 6).
4. **Suppress** customers, open deals, competitors, and anyone on the do-not-contact list before going further.
5. **Save it where the team can see it.** Create the accounts with `apollo_accounts_bulk_create` and `run_dedupe: true`, then put them on a list with `apollo_labels_add_entity_ids_to_label_names` and `modality: "accounts"`. Both verified 2026-09-14: resubmitting an already-saved company with `run_dedupe` returned it under `existing_accounts` and created nothing, and the list call put the account on a new list. **Confirm with `apollo accounts search --label-ids`**, not the list's `cached_count`, which still read 0 right after the add. Keep both the account id and the `organization_id` in the local file.

Need full firmographics rather than shallow rows? `apollo_mixed_companies_search` has them, at 1 credit per call.

## 2. Map the people at each account

People search is free. `apollo_mixed_people_api_search` with `organization_ids` (or `q_organization_domains_list`), plus `person_seniorities` and titles for the roles you want. Verified: 1,709 people at one company, 147 of them at C-suite, VP, or director level. Rows carry `title` and `has_email` but **no seniority field**, so filter seniority in the search, not afterwards.

### The default: 2 to 3 people per account, and why

Explain this to the operator before building, then let them choose:

- **Who:** one economic buyer above the line and one or two champions or users below it (see `atl-btl.md`).
- **Why not more:** colleagues compare notes. Five similar emails landing in one company in the same week reads as a blast, gets forwarded to IT, and costs you the account and some domain reputation. Two or three distinct conversations reads as someone who did their homework.
- **Why not one:** a single contact is one out-of-office, one job change, or one wrong person away from zero.

### The option: everyone at the account

When the operator wants the whole picture, offer both versions and state the cost first:

- **Free:** every person from search. Names, titles, and whether an email exists; no emails, last names masked. Good for mapping the organisation and choosing who to contact.
- **Fully enriched:** enrich all of them. **1 lead credit per person attempted**, whether or not an email comes back, so filter to `has_email: true` first. At 1,709 people that is up to 1,709 credits for one company, which is why it is an option and not the default. Measure it against the remaining balance, not the plan limit (see `operator-voice.md`).

## 3. Sequences

- **One sequence per persona, not one for the buying committee.** The executive and the champion get different copy and different subject lines, so the account sees separate conversations rather than one blast.
- **Do not rely on Apollo to stop a second person from the same company.** The enrollment flag `sequence_same_company_in_same_campaign` (CLI `--same-company`) is documented as blocking it by default, but on 2026-09-14 two people from the same company, same `organization_id` and same account, both enrolled into an inactive sequence with the flag off and nothing skipped. So the per-account cap lives in your list file, not in Apollo. Separate persona sequences are still the cleaner build.
- **The job-change guard does work.** A contact whose job has changed was skipped with the reason `contacts_with_job_change` (flag `sequence_job_change`, off by default). In ABM that is usually right: the person has left the account you are targeting.
- **Timing is the operator's call, per campaign.** Staggered means champions first and the executive a few days later, so the conversation builds. Simultaneous is faster, with more risk of colleagues comparing similar emails the same morning. Ask, write the choice into `profile.yaml`, and time the second enrollment to match.
- **Launch as usual** through `apollo-go-live`: enroll into inactive sequences, read them back, a human approves.

## 4. The stop rule: a positive reply stops the account

**When anyone at an account replies positively, stop everyone else at that account, in every ABM sequence.** A negative reply ("not me", "not interested") stops only the person who sent it, and the others continue.

Apollo will not do this for you. It finishes only the person who replied, and contact search has no account filter on MCP or CLI. So:

1. **Keep `organization_id` for every enrolled contact** in the campaign's local file. It is the only reliable way to find siblings later.
2. **Find positive replies.** `apollo emails search --stats replied` returns each reply with a `reply_class`; `willing_to_meet` and `person_referral` are the likely positives. Treat the class as a first pass and confirm by reading the reply (`positive-reply-scoring`).
3. **Stop the siblings.** Look them up by `organization_id` in the file, then `apollo_emailer_campaigns_remove_or_stop_contact_ids` with `mode: "stop"`, their contact ids, every ABM sequence id, and a stop reason that names the reply.
4. **A referral to a named colleague:** stop the others, then write to the named person with the referral as the opening line.

## 5. Measure by account

The honest ABM number is **accounts with a positive conversation, out of accounts worked**, not reply rate per email. `campaign-reporting` can group by `account_id` or `account_label_ids`, and saved account rows carry `num_contacts` and a campaign status tally for coverage.

## Everything else still applies

- **Capacity:** every person is a send. 100 accounts, 3 people, 3 steps is 900 emails, and they queue on the same mailboxes as everything else.
- **Verification before copy, suppression, and the per-company cap across campaigns** (`apollo-list-builder`).
- **Compliance** (`compliance.md`).

## Common mistakes

- **Passing an account `id` into an organization filter.** It matches nothing, silently.
- **Trusting a fuzzy lookup match** without checking the company and its domain.
- **One sequence for the whole buying committee.**
- **Turning on same-company enrollment in a normal volume campaign.**
- **Enriching everyone at a large account by default.**
- **Letting the others keep sending after someone said yes.**
- **Scraping a Dream 100 instead of choosing it.**
