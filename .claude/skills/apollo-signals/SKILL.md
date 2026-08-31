---
name: apollo-signals
description: "Find the reason to reach out now. Offer-fit signal discovery, not a universal heat score. Use to identify and act on buying signals for a specific offer on Apollo."
---

# Apollo Signals (cross-cutting)

Find the reason to reach out now. This skill is deliberately not a universal list of "buying signals" with a heat score. That approach treats every signal as equally valuable, and it is not.

## The stance: a signal is only as good as its offer fit

A signal is worth acting on to the degree it connects to what you sell. Someone attending a trade show is a strong signal if you sell trade-show services, because they have literally shown up in the room you serve. "They are hiring for a role" is a much weaker reason if it does not connect to your offer. So we do not hand out a generic heat score. We ask, for this specific business:

1. What does the buyer's world look like right *before* they buy from someone like you?
2. What observable fact marks that moment?
3. Can you actually detect it (a filter, a research check, a tool)?

Find that fact, and you have your signal. The best ones are usually discovered per business, over time, not pulled off a shelf.

## Discover the signal (the useful work)

- Look at the last few real deals. What was true about those companies right before they said yes? A tool they adopted, a page they built, a role they added, an event they joined.
- Write it as a rule you can check: "sells to X," "runs paid ads," "has a public pricing page," "exhibits at Y."
- Then decide how to detect it: a native Apollo filter, a research check Claude runs per company, or a tool.

## The signal menu the MCP can detect (all live-tested)

Every signal below works as a **search filter** in people or company search, which is how you turn a signal into a list. All were validated live on a real Apollo account. Keep the offer-fit rule: a signal is worth using only if it maps to what you sell.

- **Hiring for a role:** `q_organization_job_titles`, `organization_num_jobs_range`, `organization_job_posted_at_range`, `organization_job_locations`. To inspect one specific company's roles, use `apollo_organizations_job_postings` (1 credit per org). Caveat from testing: small and private companies often return zero postings even when active, so this signal fits larger or tech-forward targets far better than small firms. If it keeps coming back empty for your ICP, the signal does not fit your offer, the company is not necessarily quiet.
- **Headcount growth:** `organization_headcount_growth_range` + `organization_headcount_growth_past_n_months`. Every company-search result also carries `organization_headcount_*_month_growth` inline, so you get the growth read for free.
- **Recently funded:** `latest_funding_date_range`, `latest_funding_amount_range`, `total_funding_range`.
- **Technology adoption:** `currently_using_any_of_technology_uids` (also `_all_of_`, and the `not_` exclusion). "Uses or added tool X."
- **New in role (tenure):** `person_days_in_current_title_range` in people search. A fresh decision-maker in the seat.
- **Job change:** `contact_job_changed` + `job_change_new_organization_ids` in contacts search. Returns a full change event (old title, old company, new company), so you know exactly who moved where. One of the strongest B2B signals: your champion just landed somewhere new.
- **Company maturity:** `organization_founded_year_range`.
- **Department size or growth:** `organization_department_or_subdepartment_counts` (for example, a marketing team of 5 or more).

Some of these are labeled advanced filters and can return an upgrade-required error on free Apollo plans (all worked on the paid plan we tested). If one errors, it is gated on your plan, fall back to another signal.

### Website visitors (moved out of the UI, now filterable)

**This used to be UI-only and no longer is.** If your team has the Apollo tracking script installed and Website Visitors access, `apollo_mixed_companies_search` will filter to companies that visited *your* site:

- `website_visitors_from_domains` — your own tracked domains.
- `website_visitors_from_past` — 1, 7, 15, 30, 60, or 90 days.
- `website_visitors_domain_pages` / `..._exact_pages` — filter to who hit a specific path, for example `/pricing`.
- `website_visitors_intent` — Apollo's computed low / medium / high from visit behaviour.
- `web_page_view_counts` — **requires `website_visitors_from_past`**, because page views are only stored as time-windowed aggregates. Without a window it is silently ignored.
- `show_new_companies_only` — first-time visitors only.

`apollo_organizations_enrich` also returns `website_intent`, `website_last_visit`, `website_unique_visitors`, and `website_total_visits` for teams with access.

**Person-level visitor filtering is newer still, and it is MCP-only.** The company-level filters above tell you *which company* came to your site. A parallel set on `apollo_mixed_people_api_search` and `apollo_contacts_search` tells you *which person*, which is the difference between an account signal and a name to write to:

- `website_visitors_people_from_domains` / `..._from_past` — same shape as the company filters, but the window starts at 7 days, not 1.
- `website_visitors_people_pages` / `..._exact_pages` — path fragments, or exact stored `domain/path` values.
- `website_visitors_people_intent` — low / medium / high.
- `website_visitors_people_confidence_tier` — how confident Apollo is that it identified *this person*, not just the company. Filter to `high` before you write anything personal about the visit.
- `website_visitors_people_page_view_counts` — defaults to a 90-day window if you omit the time filter, rather than being ignored. This differs from the company-level `web_page_view_counts`, which is silently dropped without a window.
- `sort_by_field: last_visited_at` — most recent visitors first, which is the ordering you actually want for a timing signal.

Three limits to state plainly before building an angle on this, because two of them are gates rather than caveats.

**Person-level tracking is a paid add-on.** In the domain settings, tracking is a choice between "Company: identify companies only" and "Company & person", and the second is labelled **Inbound add-on**. Company-level filtering comes with the plan; every person-level filter in the list above needs that add-on bought first. Verified in the UI 2026-08-31. Check this before designing an angle on it, because the API will simply return nothing rather than telling you the feature is not on the plan.

**Person-level identification is United States only**, so a non-US list comes back thin or empty no matter how much traffic the site has. The same is not true of company-level, which is global.

**The script has to be live and verified.** One pixel covers both tracking modes, so there is nothing to change later when person-level is enabled. But the domain sits `Inactive` with "Failed to connect: check your script" until Apollo can actually see it, and an inactive domain yields no data while still counting against the domain limit (three on a standard plan). Read tracked domains and intent paths from `apollo_website_visitor_domain_tracker_index` rather than typing them from memory, and confirm status is active before trusting an empty result. There is a four-tool tracker group for this (`index`, `install_script`, `send_install_email`, `update`) plus `apollo_website_visitors_domain_aggregates` for the rollup.

None of it has a CLI flag, so a large visitor-based pull belongs on lane 3 with the full payload.

This is the strongest offer-fit signal in the menu, because someone reading your pricing page has already told you what they want. It also needs no guessing: it is your own data. Gated on having the tracking script installed, so ask before promising it.

## What still lives in Apollo's UI only

Worth knowing, but the MCP cannot execute these (see `../operator-context/references/apollo-kb-map.md`):

- **Search alerts:** save a search and get alerted when new people match it. A standing signal feed.
- **Workflows:** Apollo automates signal-to-action in the UI (Hit New Hires, Hit Recently Funded, Reach Website Visitors, Spot Companies Hiring for Specific Roles). Point the operator here; the MCP does not build workflows.
- **Buyer-intent topics:** company results carry `show_intent` and `intent_strength` inline, but filtering by intent topic is UI or plan gated.
- **Custom enrichment:** Clay integrates with Apollo and is a strong option for building custom signal workflows. Do not pretend it does not exist.

## How signals feed the rest

- **Into targeting (Targeting and 2):** a good signal becomes a scoring rule or a filter, so it sharpens the list.
- **Into prioritization (Iterate):** signal-matched leads are your High-priority tier, worked first.
- **Into copy (Message):** the signal is often the opener. "Saw you're exhibiting at X" is a better first line than anything generic.

## No benchmarks

You will see "signal-based outreach gets 3x replies" numbers everywhere. Ignore them. Whether a signal works depends on your offer and your market, and the only honest measure is your own positive reply rate on signal-matched leads versus cold. Test it, do not quote it.

## Common mistakes

- Treating every signal as equally valuable. Fit to the offer is everything.
- Adopting a generic heat score. It scores things that do not predict a sale for your business.
- Gating on a timing trigger. A signal is a reason to prioritize, not a hard filter. Gating on "funded in the last 6 months" shrinks the list 20x and buries you among everyone else pitching them.
