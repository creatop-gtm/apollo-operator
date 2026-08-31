# CLI recipes (verified)

Every command here was run against a live Apollo account on CLI **v2.0.0** and produced the
result described. Where the CLI misbehaves, the workaround is given rather than the ideal.

---

## Lane 3: REST with the CLI's own token

The `apollo` binary has no flag for NAICS, SIC, founded year, headcount growth, tenure, years of
experience, market segments, LinkedIn URL lookup, or person-level website visitors. (Department
headcount was on this list until CLI v2.1.0.) You do not have to fall back to MCP for those. The token `apollo auth login` already stored authenticates against the same endpoint the MCP
uses, so you get the **full filter surface with disk output and no API key**.

```bash
TOK=$(python3 -c "import json,os;print(json.load(open(os.path.expanduser('~/.config/apollo/credentials')))['access_token'])")

curl -s -X POST "https://api.apollo.io/api/v1/mixed_people/api_search" \
  -H "Authorization: Bearer $TOK" \
  -H "Content-Type: application/json" \
  -d '{"person_titles":["founder","ceo"],
       "organization_num_employees_ranges":["11,50","51,200"],
       "organization_locations":["United States"],
       "contact_email_status":["verified"],
       "organization_department_or_subdepartment_counts":{"master_sales":{"min":0,"max":2}},
       "per_page":100,"page":1}' > pages/p1.json
```

Results are under `.people`, same shape as the CLI. Paginate exactly as above.

- **Use `mixed_people/api_search`.** The older `mixed_people/search` returns `422` with
  "This endpoint is deprecated for API callers."
- Verified against MCP on the same query: identical `total_entries` (2,837), 3 MB to disk, nothing
  through context.
- Credentials live at `~/.config/apollo/credentials` (`access_token`, `refresh_token`, `expires_at`).
  Read the token at run time rather than copying it anywhere.

# Preflight (zero credits)

```bash
apollo auth whoami                      # who am I, or "Not logged in."
apollo usage credits -f json | jq '.credit_usage_stats.lead_credit'
```

`lead_credit` returns `{limit, consumed, left_over}`. Check it before any enrichment, and again
after, so you report the true spend rather than an estimate.

## Rule 1: always `-f json`, shape with `jq`

`-f csv` is unsafe on any nested response and `-f table` is unreadable on wide objects.

**`people search -f csv` is broken.** It emits two columns with the entire people array stuffed
into one quoted cell:

```
total_entries,people
3243,"[{""id"":""66f14…"",""first_name"":""Emmanuel"",…}, …]"
```

Do this instead:

```bash
apollo people search --title "founder" "ceo" --seniority c_suite founder \
  --employees "11,200" --company-location "United States" \
  --per-page 100 -f json > raw.json

jq -r '["id","first_name","title","company","has_email"],
       (.people[]|[.id,.first_name,.title,.organization.name,.has_email]) | @csv' \
  raw.json > leads.csv
```

`email-accounts list -f table` renders 100+ columns across the terminal. Use
`-f json | jq` and select the fields you actually want:

```bash
apollo email-accounts list -f json \
  | jq -r '.email_accounts[] | {email, active, revoked_at, email_daily_threshold,
                                mailwarming_status, true_warmup_status, active_campaigns_count}'
```

## Rule 2: response shapes are inconsistent between commands

There is no single envelope. Check the shape before writing a `jq` filter against it.

| Command | Where the results are | Total count at |
|---|---|---|
| `people search` | `.people` | `.total_entries` (flat, top level) |
| `companies search` | **`.accounts` AND `.organizations`** | `.pagination.total_entries` |

**`companies search` splits results across two arrays.** Records already in your team's Apollo
account land in `.accounts`; everything else lands in `.organizations`. The split shifts by page:
in a real run page 1 was 97 accounts and 3 organizations, while page 20 was 1 and 99. Reading
only `.accounts` silently lost 1,241 of 2,100 companies. Always read both:

```bash
jq -s '[.[] | (.accounts[]?, .organizations[]?)]' co_p*.json > companies_all.json
```

## Rule 3: people search is free, company search is not

Measured with a controlled before-and-after on `lead_credit.consumed`:

| Call | Credits |
|---|---|
| `people search`, any page depth | **0** |
| `companies search` | **1 per call** |
| `people bulk-enrich` | 1 per record *requested* |

Search returns an obfuscated preview (`last_name_obfuscated`, `has_email: true` booleans, no
address), never real contact data. So paginate people as wide as you like. A 21-page company pull
costs 21 credits, which is cheap but is not nothing, and it is easy to miss because the obvious
assumption is that all search is free.

## Rule 4: Apollo's pagination overlaps, so dedupe by id

A real 33-page people pull returned **3,238 rows containing 2,933 unique people**, a 9% duplicate
rate. Deduping by company, or applying a per-company cap, does not remove them, and every survivor
costs a credit at enrichment.

```bash
jq -s '[.[].people[]] | unique_by(.id)' people_p*.json > people_all.json
```

Verify rather than assume: `jq 'length'` and `jq '[.[].id]|unique|length'` must match.

## Rule 5: filter to verified emails in the search, not after

`--email-status verified` costs nothing and shrinks what you later pay to enrich. In a real run it
cut a 3,240-person universe to 2,742, avoiding roughly 500 wasted enrichment credits.

```bash
apollo people search <filters> --email-status verified --per-page 100 --page 1 -f json
```

## Bulk enrichment (verified at 848 records)

`{"id": "<apollo person id>"}` is a valid match record, so a list staged on disk enriches directly:

```bash
split -l 10 enrich_ids.txt batches/batch_
for b in batches/batch_*; do
  jq -R '{id:.}' "$b" | jq -s '.' > "$b.json"
  apollo people bulk-enrich --file "$b.json" -f json > "$b.out.json"
  sleep 0.25
done
jq -s '[.[].matches[]?] | unique_by(.id)' batches/*.out.json > enriched_full.json
```

Measured on a real run: **848 records, 85 batches, 0 failures, exactly 848 credits, ~100 seconds,
33 fields per person.** Each response carries `credits_consumed`, `total_requested_enrichments`,
`matches`, and `missing_records`. Sum the real `credits_consumed` rather than estimating.

Enrichment is where `email`, `email_status`, `email_domain_catchall_verdict`, the full
`organization` object (including `industry` and `estimated_num_employees`, both null pre-enrichment),
`employment_history`, `seniority`, and `departments` finally appear.

## Pagination to disk, without touching context

The point of the CLI path. Results are redirected to files, so the payload never enters the
model's context:

```bash
mkdir -p lists/<campaign>/pages
for p in $(seq 1 33); do
  apollo people search <filters> --per-page 100 --page $p -f json \
    > lists/<campaign>/pages/p$p.json
  n=$(jq '.people|length' lists/<campaign>/pages/p$p.json)
  [ "$n" = "0" ] && break
  sleep 0.4
done
jq -s '[.[].people[]] | unique_by(.id)' lists/<campaign>/pages/*.json \
  > lists/<campaign>/people_all.json
```

**Write pages into their own subdirectory.** If the merged output sits beside the pages, the next
command that globs `p*.json` or `*.json` will pick up the merged file too and fail with a type
error, or worse, silently double-count. A `pages/` subdirectory makes the glob unambiguous.

**Two shell traps in this pattern:**

- **zsh does not word-split unquoted variables.** A loop like `for c in "people search" …; do apollo $c --help; done` passes `"people search"` as one argument, and the CLI silently prints root help instead of erroring. Use `apollo ${=c}` in zsh, or arrays.
- **Never build filenames from email addresses.** `xargs -I{}` with an address containing `@` and dots produces "command line cannot be assembled" or unusable paths. Number the outputs instead.

A real run pulled 3,238 people into 4 MB on disk with nothing through context. The same pull over
MCP would not fit in a context window at all.

Keep the `sleep`. It is politeness toward the rate limiter, and it costs nothing on a free call.

## Free ICP composition check (no credits)

`industry` and `estimated_num_employees` come back **null** in company search results, they only
populate on enrichment. `naics_codes` and `sic_codes` **are** populated. So you can grade the
composition of a target universe for free by joining people to companies on the company name:

```bash
# 1. pull both universes to disk (free)
# 2. build a name -> naics map and tag every person
jq -n --slurpfile ppl people_all.json --slurpfile co companies_all.json '
  ($co[0] | map({key:(.name|ascii_downcase), value:{naics:.naics, domain:.domain}}) | from_entries) as $cmap
  | $ppl[0]
  | map(. + { company: (.organization.name // ""),
              naics: ($cmap[(.organization.name // ""|ascii_downcase)].naics // "") })
  | map(. + { sector: (.naics|.[0:2]), n4: (.naics|.[0:4]) })
' > people_tagged.json

# 3. see the sector mix
jq -r '.[]|.sector' people_tagged.json | sort | uniq -c | sort -rn
```

Useful 4-digit NAICS when the buyer is a B2B seller: `5415` IT services, `5132` software
publishers, `5416` management consulting, `5418` advertising, `5613` employment and staffing
(usually drift, they have in-house SDRs). Two-digit `61` is education and `62` is health care,
both usually drift for a B2B offer.

Name-based joining is fuzzy. In a real run it matched 2,861 of 3,238 people (88%); treat the
unmatched remainder as its own bucket rather than dropping it.

## Enrollment guard flags

`sequences add-contacts` sends real emails. Confirm before running it, always.

The guard flags are opt-**in**, which means Apollo blocks each of these by default. Leave them off
unless you have a stated reason:

| Flag | What turning it on does |
|---|---|
| `--active-in-other` | Enrolls people already active in another sequence (double-touch) |
| `--finished-in-other` | Enrolls people who already finished another sequence |
| `--same-company` | Enrolls a second person at a company already in the sequence |
| `--unverified-email` | Enrolls addresses that failed verification (bounce risk) |
| `--no-email` | Enrolls contacts with no email at all |
| `--skip-verification` | Skips the verification step entirely |

A clean list still produces a bounce spike if you pass `--skip-verification` or
`--unverified-email` at enrollment. See `apollo-list-builder`
for the list-side half of suppression.

## Stopping a live send

The CLI has a kill switch the MCP does not expose:

```bash
apollo sequences abort --id <sequence_id>     # deactivate, stops sending
apollo sequences archive --id <sequence_id>   # retire a finished sequence
```

Reach for `abort` on a deliverability incident, a bounce spike, or a copy error caught after
activation. See `apollo-deliverability`.

## Other useful commands

```bash
apollo sequences schedules -f json                    # schedule ids for create/update
apollo people employees --domain acme.com             # everyone at one company
apollo contacts create --dedupe --label "My List" …   # native dedupe + list on create
apollo labels add --ids <ids…> --names "My List" --modality contacts --async
```
