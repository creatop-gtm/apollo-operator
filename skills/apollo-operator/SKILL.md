---
name: apollo-operator
description: "The four ways to run Apollo headless and how to choose: MCP (typed tools, results always through context), the apollo CLI binary (disk output, common filters), REST authenticated with the CLI's own OAuth token (full MCP filter surface WITH disk output, no API key), and REST with an API key. Includes measured context costs, credit costs per lane, verified command recipes, and the kill switch for stopping a live sequence. Use to decide which lane a task should run on."
---

# Apollo Operator: the four lanes (Foundations)

Apollo can be driven four ways, and picking the right one per task decides how much context you burn, how many credits you spend, and whether a job is even possible. This library is MCP-native by default, so every other skill assumes Apollo shows up as typed tools inside the model. But there are three other routes, and for some jobs they are strictly better. This skill is the map.

## The four lanes (same API underneath)

All four hit the same Apollo API: the same data, enrichment, sequences, and CRM. What differs is **how the work is invoked**, and critically, **whether the results have to pass through the model's context.**

| Lane | How work is invoked | Auth | Results go | Best for |
|---|---|---|---|---|
| **1. MCP** (library default) | Apollo appears as tools inside the model's context. **Prefer v2** (`https://mcp.apollo.io/mcp-v2`): four router tools instead of the full catalog (85 on 2026-09-25, and growing) | OAuth 2.0 in a connector, **or a master API key** (`X-Api-Key`) for unattended runs | **Through context, always** | Guardrailed, human-gated, in-conversation steps on small record counts |
| **2. CLI binary** | A normal terminal command an agent shells out to, or a human, script, or cron runs | OAuth 2.0 (`apollo auth login`) | **To disk** | Bulk search and enrich with common filters, composable pipelines, reproducible jobs |
| **3. REST + the CLI's own token** | `curl` against the same endpoint the MCP uses, authenticated with the token `apollo auth login` already stored | OAuth 2.0, **reuses the CLI's token, no API key** | **To disk** | **Bulk work that needs advanced filters.** The full MCP filter surface with the CLI's disk output. |
| **4. REST + an API key** | You build your own client against the HTTP endpoints | API key (`x-api-key`, `APOLLO_API_KEY`). A **master** key also authenticates the MCP | Wherever you send them | Back-end services and integrations that run without a logged-in user |

Apollo explicitly warns not to confuse API access with MCP. They are different lanes to the same place.

**Lane 3 is the one most people miss**, and it is the most useful discovery in this library. See "The CLI binary is not a superset, but its token is" below.

### The one-paragraph version

**MCP** puts Apollo *inside* the model as tools: safest, most expensive in context, and results always land in the conversation. **The CLI** puts Apollo *under* the agent as a composable command whose output can go straight to a file. **Lane 3** is the CLI's cheap, disk-bound access combined with the MCP's complete filter surface, and it needs nothing you do not already have after `apollo auth login`. **Lane 4** is for building software, not for running a motion.

## MCP vs CLI (the choice that actually comes up)

For an agent running Operator, the real decision is MCP or CLI, because both can be driven from inside a Claude Code session.

- **MCP v1** (`/mcp`) loads every tool schema into the model's context whether you use them or not: 74 tools and 344 KB on 2026-09-14, **85 tools and about 485 KB on 2026-09-25**, because the catalog grows every week. **MCP v2** (`/mcp-v2`) loads four router tools, about 6 KB, and looks up the rest on demand. Either way you get typed, structured, in-conversation calls the model cannot fat-finger.
- **CLI** loads nothing upfront. The agent runs `apollo --help` on demand, then composes commands. It is deterministic (same command, same result), scriptable, version-controllable, and it runs with or without an AI in the loop.

The one-liner: **MCP puts Apollo *inside* the model as tools. The CLI puts Apollo *under* the agent as a composable command it can pipe, script, and re-run.**

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
`references/cli-recipes.md`.

### MCP v2: a router, not a bigger catalog

`https://mcp.apollo.io/mcp-v2` exposes four tools: `apollo_find_tools` (natural-language search over the catalog, returning each tool's schema and the dispatcher that runs it), `apollo_read`, `apollo_write`, and `apollo_write_destructive`. The dispatcher split is Apollo's own safety classification, and it follows credits rather than verbs: `apollo_mixed_companies_search`, `apollo_people_match`, and `apollo_organizations_enrich` are destructive because they spend `lead_credit`, while `apollo_mixed_people_api_search` is a read because it is free.

Measured against v1 twice, eleven days apart:

| | v1 `/mcp` (2026-09-14) | v2 (2026-09-14) | v1 `/mcp` (2026-09-25) | v2 (2026-09-25) |
|---|---:|---:|---:|---:|
| Tools listed | 74 | 4 | 85 | 4 |
| Tool list size | 344 KB | 5.6 KB | 485 KB | 6.0 KB |
| Same NAICS and headcount-growth search | 7,792 | 7,792 | 7,829 | 7,829 |

The v1 catalog grew by 11 tools in those eleven days and v2 stayed at four, which is the whole argument for the router in one row. Apollo's own count for the catalog is about 100; what one account is served is smaller and depends on plan and rollout.

**How to call a dispatcher.** `apollo_read`, `apollo_write`, and `apollo_write_destructive` take `action` plus the action's own parameters **flattened at the top level** next to it (`additionalProperties: true`), not nested under a `parameters` or `arguments` key. Nesting them returns `Invalid params` with a suggestion to re-read the schema, and the schema will not tell you this, because the action's parameters are not in it. Example: `{"action": "apollo_mixed_people_api_search", "person_titles": ["founder"], "per_page": 25}`.

Every v1 tool is dispatchable on v2, and the full filter surface passes through and is still validated. **Call `apollo_find_tools` first and use the action name it returns; never guess one.** What v2 does not change: results still come back through context, so bulk pulls still belong on lanes 2 and 3.

**The router runs ahead of its dispatcher, and the dispatcher catches up.** On 2026-09-14 `apollo_find_tools` documented 21 tools in no published list, and v2 refused to dispatch any of them. **By 2026-09-25, 11 of those had been published**: record collections (`apollo_custom_objects_create` / `show`, `apollo_custom_object_records_search`), fields (`apollo_fields_create` / `update`), dynamic AI enrichment (`apollo_dynamic_field_enrichment_enrich` and its `ongoing_enrichment_requests`), data sources (`apollo_data_sources_create`, `apollo_data_source_imports_create`), and CSV exports (`apollo_csv_exports_export_view` / `show`). They are in v1's tool list, in v2's dispatch enums, and in Apollo's public docs, and v2 dispatches them. Record collections still carry a plan gate: on our account the read returns `You don't have access to Sheets` (it said `AI Studio` on 2026-09-14; same lock, new name, still not a plan option or a documented product). No profile or usage call exposes the gate, so the only way to know is to try, and trying a read costs nothing.

**Still unpublished on 2026-09-25, and still reachable only on v1 with a master API key:** the nine `apollo_agent_*` tools, `apollo_email_domain_diagnosis_authentication_status`, `apollo_domain_purchase_create`, and `apollo_prompts_suggest`. The pattern to expect: a tool appears in `find_tools` first, dispatches on v1 with a master key next, and lands in the v2 enums and the docs last. The CLI's OAuth token is refused with `insufficient scope` at every stage before the last. When v2 returns `not_implemented`, the message names the dispatcher (`'apollo_read' isn't available for your account yet`) rather than the action, so do not read it as every read being down.

### Apollo's own AI agent: nine tools, one agent, one route

The router advertises nine `apollo_agent_*` tools. **They are nine doors to the same underlying Apollo AI Agent**, and a `task_id` created through one can be polled or continued through any other.

| Tool | What it does | Closest skill here |
|---|---|---|
| `apollo_agent_find_prospects` | Natural language to an Apollo search, escalating to semantic search and AI research | `apollo-icp-builder` |
| `apollo_agent_plan_gtm_campaign` | Campaign strategies from the Context Center, then executes the chosen one on confirmation | `apollo-icp-builder`, `apollo-signals` |
| `apollo_agent_draft_sequence_copy` | AI drafting, review, and optimisation of sequence copy | `apollo-sequence-builder`, `sequence-reviewer` |
| `apollo_agent_build_lead_scores` | Lead and account scoring models grounded in the ICP | `apollo-icp-builder` scoring signals |
| `apollo_agent_workflow_automation` | Builds and edits Plays (trigger and action automations). The only route to workflow CRUD. | nothing here, Plays are out of scope |
| `apollo_agent_connect_mailbox` | Mailbox connection | `sending-infrastructure` |
| `apollo_agent_summarize_calls_meetings` | Call and meeting summaries | `business-brief` inputs |
| `apollo_agent_explain_howto` | Product how-to | the KB map in `operator-context` |
| `apollo_agent_manage_billing` | Billing | none |

**Lifecycle.** CREATE: pass an `instruction` and no `task_id`, get a `task_id` with status `running`, executing asynchronously in 30 to 90 seconds. POLL: pass the `task_id` alone. CONTINUE: pass both. **Poll with the same agent tool you created with.** The CREATE response tells you to poll `apollo_agent_task`; that tool does not exist and returns `TOOL_NOT_FOUND`.

**One route only, as of 2026-09-16, re-verified 2026-09-25.**

| Route | Result |
|---|---|
| v2 dispatcher (`apollo_write`) | `not_implemented`, and none of the three action enums contains an agent action (checked again 2026-09-25) |
| Claude Code connector | Not in its tool list (2026-09-16; not re-checked since) |
| **v1 `/mcp` direct, master API key** | **Runs** (2026-09-25: `explain_howto` answered a product question from Apollo's knowledge base in under ten seconds, free) |

**Planning is free.** A full `plan_gtm_campaign` run, including four live audience sizings, moved no credit pool. Execution is a separate confirmed step and is not free, because it builds a list.

**`apollo_write` is not a safety classification.** All nine agents are advertised as `apollo_write`, and one of them will build a list and draft a sequence once confirmed. `apollo_write_destructive` reliably means treat it as spending. `apollo_write` means read what the tool actually does.

**The stance this library takes.** Apollo's agent is strong at the front of the motion and absent from the back of it: no deliverability check, no independent verification, no composition grading, no suppression, no warmup clock, no judgment about whether a sized audience is worth sending to. Where the two overlap, say so and let the operator choose. Then run whatever comes back through List and Infrastructure before anything sends.

### The CLI binary is not a superset of the MCP, but the CLI's *token* is

The `apollo` binary covers the common filters. Several **advanced filters have no CLI flag at all**:

NAICS codes and their exclusions · SIC codes and their exclusions · founded-year range ·
headcount-growth window and range · days in current title (tenure) · total years of experience ·
market segments · LinkedIn URL lookup · and the seven person-level website-visitor filters.

**Department headcount used to be on this list and landed in CLI v2.1.0** (2026-08-07). Expect more
to follow: check `--help` against this list rather than trusting it.

**But you are not stuck choosing between them.** `apollo auth login` stores an OAuth access token
at `~/.config/apollo/credentials`, and that token authenticates directly against the REST endpoint
the MCP itself uses. So you can send the **full MCP filter payload** and still redirect the
response to disk:

```bash
TOK=$(python3 -c "import json,os;print(json.load(open(os.path.expanduser('~/.config/apollo/credentials')))['access_token'])")

curl -s -X POST "https://api.apollo.io/api/v1/mixed_people/api_search" \
  -H "Authorization: Bearer $TOK" -H "Content-Type: application/json" \
  -d '{"person_titles":["founder","ceo"],
       "organization_num_employees_ranges":["11,50","51,200"],
       "organization_naics_codes":["5415"],
       "per_page":100,"page":1}' > page1.json
```

Verified live 2026-08-27: the NAICS filter bites on this lane, narrowing the same founder/CEO query
from 914,768 to 144,734, while writing to disk and costing nothing in context. **No API key needed**,
the CLI's OAuth token is enough. (The original v1.2 verification used department headcount and
matched MCP's total exactly at 2,837; that filter has since landed in the CLI.)

Two gotchas worth knowing:

- Use `mixed_people/api_search`. The older `mixed_people/search` now returns a 422 telling API
  callers to move to the new endpoint.
- This is the best of both lanes, so it is the right default for **large searches with advanced
  filters.** Reach for MCP when you want the typed, guardrailed surface on a small number of
  records, not because the CLI cannot express the query.

## When Operator uses which

Decide in this order. The first question is capability, not volume.

**1. Does the search need an advanced filter?** NAICS, SIC, founded year, headcount growth, tenure,
years of experience, market segments, LinkedIn URL lookup, or person-level website visitors. The
`apollo` binary has no flag for any of them. Under ~100 records use **lane 1 (MCP)**; above that use **lane 3 (REST with the CLI's
token)**, which gets the full filter surface *and* disk output.

**2. Is the step human-gated or reputation-sensitive?** Enrollment and activation
(`apollo-go-live`) stay on **MCP**, where you get typed
calls and no shell improvisation. Stopping a live send also works on MCP, but it has a trap that makes
the CLI the better tool in an incident (see below).

**3. Is this bulk work whose output belongs on disk?** Then **lane 2 (CLI binary)** if the common
filters cover it, **lane 3** if they do not. Paginating a few thousand people, writing every stage
to a file, and re-running the job later are all things these lanes do natively and MCP cannot do
without flooding the window. See List,
`apollo-list-builder`.

**4. Otherwise, default to MCP.** For a handful of records in conversation, the typed surface is
easier to get right, and the schemas are already loaded.

**Lane 4 (API key)** does not appear in this decision at all. It is for building a service that runs
without a logged-in user. If you are running a motion, you want one of the first three. The one
crossover: a **master** API key also authenticates lane 1, which is how to run MCP unattended (see
"Lane 1 can disappear mid-task" below).

### The rule the whole thing collapses to

**Search wide off-context, act narrow on MCP.** Pull, grade, dedupe, and suppress thousands of rows
to disk on lane 2 or 3, then hand the short final list to the guardrailed MCP path for enrollment
and activation. Every skill in this library is written against that shape.

| Job | Lane |
|---|---|
| Bulk search, common filters | 2 (CLI) |
| Bulk search, advanced filters | **3 (REST + CLI token)** |
| Bulk enrichment | 2 (`apollo people bulk-enrich --file`) |
| Small search or enrich, in conversation | 1 (MCP) |
| Enroll, approve, go live | 1 (MCP) |
| **Stop a live send** | **2** (`sequences abort`) first; 1, 3, and 4 also work |
| Building a separate service | 4 (API key) |

### The kill switch: every lane, but MCP has a trap

`apollo sequences abort --id <id>` deactivates a live sequence and stops sending, and
`sequences archive` retires a finished one. Both also exist on REST as
`POST /emailer_campaigns/{sequence_id}/abort` and `/archive`, so lanes 2, 3, and 4 can all stop a
send with one call.

**MCP can do it too, verified live 2026-09-14.** `apollo_sequences_update` takes a required `active`
flag, and `active: false` flips a running sequence to `status_reason: manual_pause`. The trap: the
update is **declarative**. The `emailer_steps` array you send is the full intended state, so any step,
touch, or template whose id you leave out is **deleted**. A quick "just set active to false" call
wipes the sequence. The safe MCP kill switch is two calls: `apollo_emailer_campaigns_show` for every
step, touch, and template id with its content, then `apollo_sequences_update` with `active: false`
and all of it echoed back unchanged.

That is too much to get right mid-incident, so **the CLI's `abort` stays the recommended kill
switch.** Have it installed before you activate anything. MCP is the fallback when the CLI is not
available, not the plan. Earlier versions of this library said MCP could not stop a sequence at all;
that is no longer true.

**Status:** the library is MCP-native by default and stays that way for anything guardrailed. The
CLI-backed variants of the bulk search and grade steps are live in
`apollo-list-builder` as of v1.2, with verified
commands in `references/cli-recipes.md`.

## Setup (MCP)

- **Connect it:** `claude mcp add --transport http apollo https://mcp.apollo.io/mcp`, then `/mcp` inside Claude Code to sign in. For the router version, use `https://mcp.apollo.io/mcp-v2` instead (see "MCP v2" above).
- **Or install Apollo's plugin:** `/plugin marketplace add apolloio/apollo-mcp-plugin`, then `/plugin install apollo@apollo-plugin-marketplace`, and restart. It connects the same MCP (v1, `/mcp`, as of 2026-09-25) and adds five skills (`/apollo:prospect`, `/apollo:enrich-lead`, `/apollo:sequence-load`, `/apollo:analytics`, and `/apollo:gtm-strategist`, added 2026-09-16). The first four are Apollo's quick paths and this library is the method around them. The fifth overlaps this library's Targeting phase directly, so read it before assuming a job is ours: it is Apollo's own doctrine for audience building and execution, and the complement-not-compete rule in `operator-context` applies to it as much as to the agent tools.
- **Unattended runs:** send a **master** API key in an `X-Api-Key` header instead of signing in. Scoped keys do not reach the MCP.
- **Preflight:** `apollo_users_api_profile` confirms the connection, the same check `operator-context` uses.

## Setup (CLI)

Per Apollo's docs (https://docs.apollo.io/docs/apollo-cli-overview):

- **Install:** Homebrew (`brew install apolloio/apollo-io-cli/apollo-io-cli`), a prebuilt binary for macOS, Linux, or Windows (no Node needed), or from source (Node 18+).
- **Auth:** `apollo auth login` once, a browser OAuth flow, not an API key. Credentials store and auto-refresh locally. `apollo auth whoami` confirms the session, and there is no `auth status`.
- **Output:** nominally JSON, JSONL, CSV, YAML, or table. **In practice, use `-f json` and shape with `jq`.** `-f csv` is broken on nested responses (`people search -f csv` returns the whole result array inside one cell) and `-f table` is unreadable on wide objects.
- **Covers:** people search and enrich, company operations, contact, account, and deal management, sequence create, enroll, approve, abort, and archive, call logging, task creation, news, and credit usage.

### Command reference: use Apollo's own skill

Apollo maintains an official Claude Code skill for the CLI, shipped in the CLI repo:

```
https://raw.githubusercontent.com/apolloio/apollo-io-cli/main/.claude/skills/apollo-cli/SKILL.md
```

**Use it as the command reference, and treat it as more current than anything written here.** Apollo shipped 30+ launches in twelve weeks, so a copied command list goes stale quickly, and a stale command reference is worse than none. This skill deliberately does not duplicate it.

What this skill adds on top: **which lane to use and why**, the measured context and credit costs, and the failure modes we hit running it for real. That is a different job from documenting commands, and it is the part that does not go stale.

Command groups, for orientation only: people (search, enrich, bulk-enrich, email, employees) · companies (search, enrich, bulk-enrich, get, jobs) · contacts · accounts · deals · sequences · tasks · calls · news · conversations · users and credits · analytics. Pagination flags include `--per-page`, `--page`, `--sort-by`, and `--sort-asc`.

Our **verified recipes**, response-shape gotchas, and the enrollment guard flags, none of which are in Apollo's docs, live in
`references/cli-recipes.md`.

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

**Not yet verified:** that running a dynamic prompt-execution field decrements `ai_credit` specifically. The pool structure and the 800,000 limit make it the obvious candidate, but the enrichment tool that runs those fields is reachable only on the v1 MCP with a master API key, and record collections additionally need an unreleased "AI Studio" access flag, so it has not been measured from a script yet. Treat as strongly indicated, not measured.

## Preflight (zero credit)

Before real work on any lane, confirm access. The CLI and API expose `GET /v1/auth/health`, which returns `{ "healthy": true, "is_logged_in": true }` and spends **no credits**. It is the cleanest way to verify a session or key works before doing anything that costs money, the same role `apollo_users_api_profile` plays on the MCP preflight in `operator-context`.

## Lane 1 can disappear mid-task. The others do not.

**Observed 2026-08-31.** MCP writes succeeded at 10:38. Minutes later the same connection returned `This connector requires authentication.` In the same minute, `apollo auth whoami` was fine and a lane 3 REST call returned 200. The MCP session had expired; the CLI's stored OAuth token had not.

This is an **availability** difference, not a capability one, and it is the part of the four-lane model that is easiest to miss. Lane 1 is a connector session that can lapse without warning in the middle of a job. Lanes 2, 3, and 4 share a credential on disk with a much longer life.

Two practical consequences:

- **For anything long-running or unattended, prefer lanes 2 to 4, or lane 1 on a master API key.** A batch that takes 20 minutes should not depend on a session that can expire at minute 12.
- **A sudden authentication error on MCP does not mean the account is broken.** Check `apollo auth whoami` before you start debugging credentials. If the CLI is fine, it is the connector that needs reconnecting, and only the user can do that.

**The fix for unattended MCP is a master API key.** The MCP accepts an API key in an `X-Api-Key` header instead of a connector session, on both `/mcp` and `/mcp-v2`, so there is no consent prompt and no session to lapse. It must be a **master** key: Apollo's docs say a scoped key cannot reach the MCP. Verified 2026-09-14: the same tool list as OAuth (73 of 74, missing only `apollo_survey_submit`) and identical search totals. Treat the key like a password, because a master key is not limited to any set of endpoints.

## Hand back a URL whenever a human has to look at something

Headless work still ends in a person's browser more often than the lanes suggest. **Sequences are readable now:** `apollo_emailer_campaigns_show` returns every step, touch, subject, and body (verified 2026-09-14; v1.3 found every read path empty), so an agent can check what a sequence will send. Over REST, `GET /emailer_campaigns/{id}` returns the same content with a master API key, while the CLI's OAuth token still gets a 403. **A human still opens the sequence before activation**, because approval is a policy, not a missing read. Dynamic AI field prompts are still not readable. Whenever you create or change something a person has to approve, print the link rather than the id.

| Object | URL |
|---|---|
| Sequence | `https://app.apollo.io/#/sequences/<id>` |
| List (label) | `https://app.apollo.io/#/lists/<id>` |
| Contact | `https://app.apollo.io/#/contacts/<id>` |

An id makes the operator go hunting; a link makes it one click. A verification step someone has to hunt for is a verification step they skip, and on sequences that means real emails going out unread. Say what the link is *for*, not just that it exists.

## Common mistakes

- **Confusing API access with MCP.** Different lanes, different auth. Apollo warns about this directly.
- **Loading the full MCP tool surface for a bulk search** a single CLI command plus `jq` would do more cheaply. Match the lane to the job.
- **Using the CLI for human-gated activation.** Enrollment and go-live want the MCP's typed, guardrailed calls, not a shell command the model composed.
- **Assuming the CLI needs an API key.** It is OAuth (`apollo auth login`). The API key is lane 4, for building your own client, and a master key is also how to run the MCP unattended.
- **Assuming the CLI can run any search the MCP can.** It cannot. NAICS, SIC, tenure, headcount growth, founded year, and market segments have no CLI flag. They work on MCP and on lane 3 (REST with the CLI's token). Check the filter surface before picking the lane.
- **Trusting `-f csv`.** It silently produces a one-cell dump on nested responses. Use `-f json | jq … | @csv`.
- **Reading only `.accounts` from `companies search`.** Results are split across `.accounts` and `.organizations`, and the split shifts by page. Reading one array quietly loses most of the result set.
- **Expecting `industry` from a search.** It comes back null on both lanes. Only `naics_codes` and `sic_codes` are populated pre-enrichment, and they are enough to grade composition for free.
- **Calling `apollo_sequences_update` with `active: false` and no steps.** The update is declarative, so every step you leave out is deleted. Read the sequence with `apollo_emailer_campaigns_show` and echo everything back, or use `apollo sequences abort`.
- **Guessing an action name on MCP v2.** Call `apollo_find_tools` and use the name and dispatcher it returns. An invented name fails, and a name from the router can still be undispatchable.
