# Apollo search filter reference

> What Apollo can actually filter on, via `apollo_mixed_people_api_search` (find people) and `apollo_mixed_companies_search` (find companies). Use real fields. Do not invent filters.

## People search: person-level filters

| ICP language | Apollo field | Notes |
|---|---|---|
| Job titles | `person_titles` (array) | Pair with `include_similar_titles: false` for strict matches. |
| Seniority | `person_seniorities` | Values: `senior`, `manager`, `director`, `vp`, `c_suite`, and more. |
| Location | `person_locations` | e.g. `['United States', 'London']`. |
| Total experience | `person_total_yoe_range` | `{min, max}` years across whole career. |
| Time in current role | `person_days_in_current_title_range` | Days. 90 = new in seat, a job-change signal. |
| Free-text | `q_keywords` | Names, titles, employers, emails. |
| Email status | `contact_email_status` | `verified`, `likely to engage`, `unverified`, `unavailable`. |
| Specific people | `person_linkedin_urls` | Full profile URLs. |

## People + company search: company-level filters

These apply to both endpoints (on people search, they filter by the person's current employer).

| ICP language | Apollo field | Notes |
|---|---|---|
| Company size | `organization_num_employees_ranges` | e.g. `['1,10','11,50','51,200']`. |
| Industry (keyword) | `q_organization_keyword_tags` | e.g. `['SaaS','fintech']`. Easiest industry filter. |
| Industry (code) | `organization_naics_codes` / `organization_sic_codes` | Prefix match on NAICS, so shorter is broader. |
| Revenue | `revenue_range` | `{min, max}`, raw integers, no symbols or commas. |
| Location (HQ) | `organization_locations` / `organization_not_locations` | Company search can exclude with `not`. |
| Tech stack | `currently_using_any_of_technology_uids` / `all_of` | Underscores for spaces, e.g. `google_analytics`. People search can also exclude with `currently_not_using_any_of`. |
| Market segment | `market_segments` | e.g. `['B2B','Enterprise']`. |
| Founded year | `organization_founded_year_range` | `{min, max}`. |
| Domains | `q_organization_domains_list` | Target or exclude named companies. |
| Company name | `q_organization_name` | Company search only. |

## Signal-type filters (timing and intent)

These are how you express many "signals" as native filters instead of guesswork.

| Signal | Apollo field | Endpoint |
|---|---|---|
| Department size (e.g. sales team >= 5) | `organization_department_or_subdepartment_counts: {master_sales: {min: 5}}` | both |
| Headcount growth | `organization_headcount_growth_range` + `organization_headcount_growth_past_n_months` | both |
| Actively hiring (roles) | `q_organization_job_titles`, `organization_num_jobs_range`, `organization_job_locations`, `organization_job_posted_at_range` | both |
| Recently funded | `latest_funding_date_range`, `latest_funding_amount_range`, `total_funding_range` | company search only |

**Valid department keys** (for the department-count filter): `c_suite`, `product_management`, `master_engineering_technical`, `design`, `education`, `master_finance`, `master_human_resources`, `master_information_technology`, `master_legal`, `master_marketing`, `medical_health`, `master_operations`, `master_sales`, `consulting`. An unknown key returns zero matches silently, so use these exact keys.

## Gotchas

- **Search returns no emails.** People search is for finding, not contacting. Emails (and unmasking last names) come from enrichment at List (`apollo_people_bulk_match`). Masked last names in search are expected and do not block enrichment.
- **50,000-record display ceiling** (100 per page, 500 pages). If your search exceeds this, it is too broad regardless. Add filters.
- **Company search costs 1 credit** per call that returns results (0 if no matches). Confirm first, verbatim: "This will consume 1 credit. Do you want to proceed?" People search is for prospecting net-new people.
- **Some filters are plan-gated** (e.g. department counts, founded-year, headcount growth). Free plans get an upgrade-required error; note it and move on.
- **`include_similar_titles` defaults to true, and it is a volume multiplier.** Left on with broad titles ("events manager", "marketing manager") it can inflate a search 100x. Real example: strict trade-show titles = 679 in the US; the same intent with broad event/marketing titles and similar-titles on = 91,000. Always sanity-check by running once with `include_similar_titles: false` and only the strict core titles to see the true base, then widen deliberately. Turn it off when you need exact titles only.
- **The match count is the `total_entries` field** in the response. Read it, report it, and never trust a single inflated number without checking what is driving it.
