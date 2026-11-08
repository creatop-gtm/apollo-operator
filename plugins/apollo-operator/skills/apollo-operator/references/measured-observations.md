# Measured observations from the tagged source

These are Creatop's historical observations, not current account balances or promised outcomes. Dates and figures are retained from the source. Use the runtime capability check before acting on any local-command example.

### What the context saving actually is (measured)

Be precise about this, because the obvious guess is wrong.

| | Measured |
|---|---|
| One MCP tool schema (`apollo_mixed_people_api_search`) | ~20 KB, roughly 5,000 tokens (2026-11-08; it was ~29 KB in July). It is 1 of 96. |
| The whole MCP tool list | ~526 KB (2026-11-08). The v2 router cut this to 6 KB until it was retired. |
| CLI help, loaded on demand | `apollo --help` 1.9 KB · `people search --help` 3.0 KB · `sequences create --help` 0.9 KB |
| **Response payload, same search both lanes** | **Byte-identical.** Same totals, same ids, same obfuscated preview shape. |

So the CLI does **not** return leaner data. Its advantage is two other things:

1. **Schema overhead.** On MCP you pay for every tool definition whether or not you use them, 96 of them as of 2026-11-08. The v2 router removed almost all of it while it lasted; with it retired, this advantage is back.
2. **Results can bypass context entirely.** `> leads.json` means the payload never reaches the
   model. MCP results always pass through context. In a real run, 3,238 people came down as 4 MB
   on disk with nothing in the window. That pull does not fit in a context window over MCP, so
   this is not an optimization, it is the difference between possible and impossible.

The composability is the other standout: search, filter, dedupe, and save chain in one line, which
is exactly the shape of List list work. MCP tool calls do not compose like that. Just note that
`-f csv` is broken on nested responses, so the pipeline ends in `jq`, not in `-f csv`. See
[CLI recipes](cli-recipes.md).

| | v1 `/mcp` (2026-09-14) | v1 (2026-09-25) | v1 (2026-11-08) |
|---|---:|---:|---:|
| Tools listed (master key) | 74 | 85 | 96 |
| Tool list size | 344 KB | 485 KB | 526 KB |

Verified live 2026-08-27: the NAICS filter bites on this lane, narrowing the same founder/CEO query
from 914,768 to 144,734, while writing to disk and costing nothing in context. **No API key needed**,
the CLI's OAuth token is enough. (Both filters this lane was first proven with, department headcount in v1.2 and NAICS here, have since landed in the CLI. The lane still matters for the filters listed above.)

**The catalog moves in both directions.** Between 2026-09-25 and 2026-11-08 it went from 85 to 96 tools, and the churn was larger than the net. Published since: email and phone validation, duplicate finding and merging, per-contact step content, deals, the website-visitor tracker install tools, Context Center product reads, task reads, and the agent tools below. **Gone from the list:** record collections (`apollo_custom_objects_*`, `apollo_custom_object_records_search`), field create and update, dynamic AI enrichment, data sources and imports, CSV exports, domain authentication diagnosis, and domain purchase. Several of those were published as recently as 2026-09-25. **Unlisted is not always unreachable**: `apollo_email_domain_diagnosis_authentication_status` still answered a direct call with a master key on 2026-11-08 although it is no longer listed. Do not build a workflow on an unlisted tool, and do not assume a listed one from last month is still there.

## Rate limits and usage

Credits are not the only ceiling. `POST /usage_stats/api_usage_stats` (REST) returns **usage and rate limits together**, per endpoint, and it is the only place to see them.

Check it before scripting a long run, because the failure mode is silent: a burst gets throttled, and if your script treats a non-200 as a result rather than a retry, you record garbage. That is exactly how a verification run in testing recorded 25 of 30 responses as valid when they were rejections.

Practical defaults that have held up in real runs: a **0.2 to 0.4 second sleep between paginated calls**, a small worker pool rather than a burst for anything parallel, retry on anything that is not an explicit success, and never treat a `200` as proof (several APIs, Apollo included, return `200` with a failure flag in the body).

## Credits are eleven separate pools, not one balance

**The single most useful thing to know before budgeting anything.** `apollo usage credits` returns eleven independent pools, and spending one does not touch the others. A real paid account on 2026-08-31:

| Pool | Limit | What draws on it |
|---|---|---|
| `lead_credit` | 4,000 | Contact data: enrichment, email reveal, company search. **The scarce one.** |
| `ai_credit` | **800,000** | Apollo's own AI generation, including dynamic prompt-execution fields |
| `broadcast_credit` | 50,000 | Sending volume |
| `conversation_credit` | 4,000 | Call and meeting transcripts |
| `direct_dial_credit` | 4,000 | Phone number reveal |
| `dialer` | 390 | Dialer minutes |
| `web_search_record_credit` | 200 | Agentic web-search lookups |
| `inbound_website_visitor_credit` | 100 | Website visitor identification |
| `export_credit`, `power_up_credit`, `contact_website_visitor_credit` | 0 here | Plan-gated, zero on this account |

**Read the ratio, because it drives real decisions.** `ai_credit` is 200 times larger than `lead_credit` on the same plan. Apollo prices its own AI generation as near-free and prices **data** as the scarce resource. So the expensive thing about a list is finding and verifying the people, not writing to them.

Two consequences worth acting on:

- **Never say "credits" without saying which pool.** "We have 3,000 credits left" is meaningless on its own, and a skill that warns about credit cost without naming the pool sends operators to conserve the wrong thing.
- **Generation inside Apollo is cheap; generation outside it is not.** This is the whole basis of the two-path personalization choice in `apollo-sequence-builder`.

**Observed 2026-08-31.** MCP writes succeeded at 10:38. Minutes later the same connection returned `This connector requires authentication.` In the same minute, `apollo auth whoami` was fine and a lane 3 REST call returned 200. The MCP session had expired; the CLI's stored OAuth token had not.
