# Measured observations from the tagged source

These are Creatop's historical observations, not current account balances or promised outcomes. Dates and figures are retained from the source. Use the runtime capability check before acting on any local-command example.

### What the context saving actually is (measured)

Be precise about this, because the obvious guess is wrong.

| | Measured |
|---|---|
| One MCP v1 tool schema (`apollo_mixed_people_api_search`) | ~19 KB, roughly 5,000 tokens (2026-09-25; it was ~29 KB in July). It is 1 of 85. |
| MCP v2 full tool list (the router) | 6.0 KB for all four tools, 1.2% of v1's 485 KB (2026-09-25; 5.6 KB against 344 KB, 1.6%, on 2026-09-14) |
| CLI help, loaded on demand | `apollo --help` 1.9 KB · `people search --help` 3.0 KB · `sequences create --help` 0.9 KB |
| **Response payload, same search both lanes** | **Byte-identical.** Same totals, same ids, same obfuscated preview shape. |

So the CLI does **not** return leaner data. Its advantage is two other things:

1. **Schema overhead.** On MCP v1 you pay for every tool definition whether or not you use them, 85 of them as of 2026-09-25. **MCP v2
   removes almost all of it**, so on v2 the CLI's real advantage is the second point.
2. **Results can bypass context entirely.** `> leads.json` means the payload never reaches the
   model. MCP results always pass through context. In a real run, 3,238 people came down as 4 MB
   on disk with nothing in the window. That pull does not fit in a context window over MCP, so
   this is not an optimization, it is the difference between possible and impossible.

The composability is the other standout: search, filter, dedupe, and save chain in one line, which
is exactly the shape of List list work. MCP tool calls do not compose like that. Just note that
`-f csv` is broken on nested responses, so the pipeline ends in `jq`, not in `-f csv`. See
[CLI recipes](cli-recipes.md).

Measured against v1 twice, eleven days apart:

| | v1 `/mcp` (2026-09-14) | v2 (2026-09-14) | v1 `/mcp` (2026-09-25) | v2 (2026-09-25) |
|---|---:|---:|---:|---:|
| Tools listed | 74 | 4 | 85 | 4 |
| Tool list size | 344 KB | 5.6 KB | 485 KB | 6.0 KB |
| Same NAICS and headcount-growth search | 7,792 | 7,792 | 7,829 | 7,829 |

The v1 catalog grew by 11 tools in those eleven days and v2 stayed at four, which is the whole argument for the router in one row. Apollo's own count for the catalog is about 100; what one account is served is smaller and depends on plan and rollout.

Verified live 2026-08-27: the NAICS filter bites on this lane, narrowing the same founder/CEO query
from 914,768 to 144,734, while writing to disk and costing nothing in context. **No API key needed**,
the CLI's OAuth token is enough. (The original v1.2 verification used department headcount and
matched MCP's total exactly at 2,837; that filter has since landed in the CLI.)

**The router runs ahead of its dispatcher, and the dispatcher catches up.** On 2026-09-14 `apollo_find_tools` documented 21 tools in no published list, and v2 refused to dispatch any of them. **By 2026-09-25, 11 of those had been published**: record collections (`apollo_custom_objects_create` / `show`, `apollo_custom_object_records_search`), fields (`apollo_fields_create` / `update`), dynamic AI enrichment (`apollo_dynamic_field_enrichment_enrich` and its `ongoing_enrichment_requests`), data sources (`apollo_data_sources_create`, `apollo_data_source_imports_create`), and CSV exports (`apollo_csv_exports_export_view` / `show`). They are in v1's tool list, in v2's dispatch enums, and in Apollo's public docs, and v2 dispatches them. Record collections still carry a plan gate: on our account the read returns `You don't have access to Sheets` (it said `AI Studio` on 2026-09-14; same lock, new name, still not a plan option or a documented product). No profile or usage call exposes the gate, so the only way to know is to try, and trying a read costs nothing.

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
