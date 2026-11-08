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
| **1. MCP** (library default) | Apollo appears as tools inside the model's context, at `https://mcp.apollo.io/mcp`. The full catalog loads up front (96 tools on 2026-11-08, and growing) | OAuth 2.0 in a connector, **or a master API key** (`X-Api-Key`) for unattended runs | **Through context, always** | Guardrailed, human-gated, in-conversation steps on small record counts |
| **2. CLI binary** | A normal terminal command an agent shells out to, or a human, script, or cron runs | OAuth 2.0 (`apollo auth login`) | **To disk** | Bulk search and enrich with common filters, composable pipelines, reproducible jobs |
| **3. REST + the CLI's own token** | `curl` against the same endpoint the MCP uses, authenticated with the token `apollo auth login` already stored | OAuth 2.0, **reuses the CLI's token, no API key** | **To disk** | **Bulk work that needs advanced filters.** The full MCP filter surface with the CLI's disk output. |
| **4. REST + an API key** | You build your own client against the HTTP endpoints | API key (`x-api-key`, `APOLLO_API_KEY`). A **master** key also authenticates the MCP | Wherever you send them | Back-end services and integrations that run without a logged-in user |

Apollo explicitly warns not to confuse API access with MCP. They are different lanes to the same place.

**Lane 3 is the one most people miss**, and it is the most useful discovery in this library. See "The CLI binary is not a superset, but its token is" below.

### The one-paragraph version

**MCP** puts Apollo *inside* the model as tools: safest, most expensive in context, and results always land in the conversation. **The CLI** puts Apollo *under* the agent as a composable command whose output can go straight to a file. **Lane 3** is the CLI's cheap, disk-bound access combined with the MCP's complete filter surface, and it needs nothing you do not already have after `apollo auth login`. **Lane 4** is for building software, not for running a motion.

## MCP vs CLI (the choice that actually comes up)

For an agent running Operator, the real decision is MCP or CLI, because both can be driven from inside a Claude Code session.

- **MCP** (`/mcp`) loads every tool schema into the model's context whether you use them or not: 74 tools and 344 KB on 2026-09-14, 85 tools and 485 KB on 2026-09-25, **96 tools and about 526 KB on 2026-11-08**. The v2 router that loaded four tools instead is retired (see below). In return you get typed, structured, in-conversation calls the model cannot fat-finger.
- **CLI** loads nothing upfront. The agent runs `apollo --help` on demand, then composes commands. It is deterministic (same command, same result), scriptable, version-controllable, and it runs with or without an AI in the loop.

The one-liner: **MCP puts Apollo *inside* the model as tools. The CLI puts Apollo *under* the agent as a composable command it can pipe, script, and re-run.**

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
`references/cli-recipes.md`.

### MCP v2 is retired: one endpoint again

In September 2026 Apollo added a second endpoint, `https://mcp.apollo.io/mcp-v2`, that exposed four router tools (`apollo_find_tools`, `apollo_read`, `apollo_write`, `apollo_write_destructive`) instead of the full catalog, for about 1% of the schema preload. **It is gone.** On 2026-11-08 `/mcp-v2` returns 404, and Apollo's MCP docs (updated 2026-11-04) list only `/mcp`. If a connector or a config still points at `/mcp-v2`, repoint it at `/mcp`.

What that changes:

- **The schema preload is back.** The full catalog loads into context whatever you use: 96 tools and about 526 KB on 2026-11-08. That makes "search wide off-context" (below) matter more, not less.
- **The cost signal survived.** The router's read / write / destructive split now lives on each tool as `readOnlyHint` and `destructiveHint` annotations, and it still follows credits rather than verbs: people search and organization lookup are read-only, company search, enrichment, and email validation are destructive. Treat destructive as "quote the cost and ask first".
- **The live tool list is the authority.** There is no `find_tools` to ask any more, so check what the connection actually lists before calling something, and before telling anyone Apollo cannot do it.

**The catalog moves in both directions.** Between 2026-09-25 and 2026-11-08 it went from 85 to 96 tools, and the churn was larger than the net. Published since: email and phone validation, duplicate finding and merging, per-contact step content, deals, the website-visitor tracker install tools, Context Center product reads, task reads, and the agent tools below. **Gone from the list:** record collections (`apollo_custom_objects_*`, `apollo_custom_object_records_search`), field create and update, dynamic AI enrichment, data sources and imports, CSV exports, domain authentication diagnosis, and domain purchase. Several of those were published as recently as 2026-09-25. **Unlisted is not always unreachable**: `apollo_email_domain_diagnosis_authentication_status` still answered a direct call with a master key on 2026-11-08 although it is no longer listed. Do not build a workflow on an unlisted tool, and do not assume a listed one from last month is still there.

| | v1 `/mcp` (2026-09-14) | v1 (2026-09-25) | v1 (2026-11-08) |
|---|---:|---:|---:|
| Tools listed (master key) | 74 | 85 | 96 |
| Tool list size | 344 KB | 485 KB | 526 KB |

Apollo's own count for the catalog is about 100; what one account is served is smaller and depends on plan and rollout.

### Apollo's own AI agent: twelve tools, one agent

The MCP lists twelve `apollo_agent_*` tools and a poll tool on 2026-11-08. **They are twelve doors to the same underlying Apollo AI Agent.**

| Tool | What it does | Closest skill here |
|---|---|---|
| `apollo_agent_find_prospects` | Natural language to an Apollo search, escalating to semantic search and AI research | `apollo-icp-builder` |
| `apollo_agent_plan_gtm_campaign` | Campaign strategies from the Context Center, then executes the chosen one on confirmation | `apollo-icp-builder`, `apollo-signals` |
| `apollo_agent_bulk_enrich` | Enriches a search's results or more than 10 named people, quotes the cost, waits for approval, saves them as contacts | `apollo-list-builder` step 5 |
| `apollo_agent_research_and_analyze` | AI research columns on existing contacts or accounts (qualify, summarize, personalize), with a preview and a cost quote. Apollo's route for what record collections used to do | `apollo-list-builder`, `apollo-sequence-builder` |
| `apollo_agent_draft_sequence_copy` | AI drafting, review, and optimisation of sequence copy | `apollo-sequence-builder`, `sequence-reviewer` |
| `apollo_agent_optimize_sequences` | Finds why a sequence underperforms, proposes one change, applies it on approval (overwrites live copy or settings) | `experiment-design`, `sequence-reviewer` |
| `apollo_agent_analyze_performance` | Open-ended performance analysis and dashboards across emails, calls, meetings, and pipeline | `campaign-reporting` |
| `apollo_agent_fix_email_deliverability` | Bounce and spam analysis, SPF/DKIM/DMARC, sending limits, not-sent remediation | `apollo-deliverability` |
| `apollo_agent_workflow_automation` | Builds and edits Plays (trigger and action automations). The only route to workflow CRUD. | nothing here, Plays are out of scope |
| `apollo_agent_connect_mailbox` | Mailbox connection | `sending-infrastructure` |
| `apollo_agent_summarize_calls_meetings` | Call and meeting summaries | `business-brief` inputs |
| `apollo_agent_explain_howto` | Product how-to | the KB map in `operator-context` |

`apollo_agent_build_lead_scores` and `apollo_agent_manage_billing`, listed in September, are not in the 2026-11-08 list.

**Lifecycle, verified 2026-11-08.** CREATE: pass an `instruction` and no `task_id`; you get a `task_id`, status `running`, and `poll_after_ms` (10,000). POLL: call `apollo_agent_poll_task` with the `task_id` and the `cursor` you were given, and pass the newest cursor each time so only new messages come back. An empty `messages` list means nothing changed yet, not that the task is done: read `status`. `awaiting_input` means the agent asked a question, so put it to the operator. (In September the CREATE response pointed at an `apollo_agent_task` tool that did not exist; the dedicated poll tool fixes that.) The CLI has the same pair as `apollo agent run` and `apollo agent poll` since v2.2.0.

**Reachable everywhere now.** Published on `/mcp`, listed in the Claude connector, and callable with a master API key (2026-11-08). In September they ran only on v1 with a master key.

**Planning is free; executing is not.** A full `plan_gtm_campaign` run, including four live audience sizings, moved no credit pool (2026-09-16), and an `explain_howto` answer in 30 seconds moved none (2026-11-08). Execution is a separate confirmed step, and `bulk_enrich` and `research_and_analyze` quote credits before they run.

**Destructive is a stop sign, not a description.** Every agent tool is marked `destructiveHint`, including the one that only answers how-to questions. Read what the tool actually does before you quote a cost.

**The stance this library takes.** Apollo's agent is strong at the front of the motion and now offers a diagnosis at the back of it too. What it does not do: verify addresses independently of Apollo, grade list composition, suppress against prior campaigns, respect a warmup clock, or judge whether a sized audience is worth sending to. Where the two overlap, say so and let the operator choose. Then run whatever comes back through List and Infrastructure before anything sends.

### The CLI binary is not a superset of the MCP, but the CLI's *token* is

The `apollo` binary covers the common filters, and since **v2.2.0 (2026-10-28)** many of the advanced ones: NAICS and SIC codes with their exclusions (`--naics`, `--not-naics`, `--sic`, `--not-sic`), headcount growth (`--headcount-growth` with `--headcount-growth-months` of 6, 12, or 24), lookalikes (`--lookalike-ids`), industry names, and title, domain, and location exclusions. Read from the v2.2.0 source on 2026-11-08. **Department headcount landed earlier, in v2.1.0** (2026-08-07).

**Still no CLI flag on 2026-11-08:**

founded-year range · days in current title (tenure) · total years of experience · market segments · LinkedIn URL lookup in search · and the seven person-level website-visitor filters.

Expect this list to keep shrinking: check `--help` against it rather than trusting it. **After upgrading, run `apollo auth login` again**: v2.2.0's new commands need new token scopes, and until you sign in again they return `403`.

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
the CLI's OAuth token is enough. (Both filters this lane was first proven with, department headcount in v1.2 and NAICS here, have since landed in the CLI. The lane still matters for the filters listed above.)

Two gotchas worth knowing:

- Use `mixed_people/api_search`. The older `mixed_people/search` now returns a 422 telling API
  callers to move to the new endpoint.
- This is the best of both lanes, so it is the right default for **large searches with advanced
  filters.** Reach for MCP when you want the typed, guardrailed surface on a small number of
  records, not because the CLI cannot express the query.

## When Operator uses which

Decide in this order. The first question is capability, not volume.

**1. Does the search need an advanced filter?** Founded year, tenure, years of experience, market segments, LinkedIn URL lookup, or person-level website visitors. The `apollo` binary has no flag for any of them (NAICS, SIC, and headcount growth moved into the CLI in v2.2.0). Under ~100 records use **lane 1 (MCP)**; above that use **lane 3 (REST with the CLI's
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

- **Connect it:** `claude mcp add --transport http apollo https://mcp.apollo.io/mcp`, then `/mcp` inside Claude Code to sign in. `/mcp` is the only endpoint; `/mcp-v2` was retired (see "MCP v2 is retired" above).
- **Or install Apollo's plugin:** `/plugin marketplace add apolloio/apollo-mcp-plugin`, then `/plugin install apollo@apollo-plugin-marketplace`, and restart. It connects the same MCP (`/mcp`) and adds five skills (`/apollo:prospect`, `/apollo:enrich-lead`, `/apollo:sequence-load`, `/apollo:analytics`, and `/apollo:gtm-strategist`, added 2026-09-16). The first four are Apollo's quick paths and this library is the method around them. The fifth overlaps this library's Targeting phase directly: it is Apollo's own doctrine for forming a play, building an audience, and executing it, read in full on 2026-09-28 and unchanged on 2026-11-08. The complement-not-compete rule in `operator-context` applies to it as much as to the agent tools, and the detailed position, including where it hands off to this library, is in `apollo-icp-builder` 6c.
- **Unattended runs:** send a **master** API key in an `X-Api-Key` header instead of signing in. Scoped keys do not reach the MCP.
- **Preflight:** `apollo_users_api_profile` confirms the connection, the same check `operator-context` uses.

## Setup (CLI)

Per Apollo's docs (https://docs.apollo.io/docs/apollo-cli-overview):

- **Install:** Homebrew (`brew install apolloio/apollo-io-cli/apollo-io-cli`), a prebuilt binary for macOS, Linux, or Windows (no Node needed), or from source (Node 18+).
- **Auth:** `apollo auth login` once, a browser OAuth flow, not an API key. Credentials store and auto-refresh locally. `apollo auth whoami` confirms the session, and there is no `auth status`.
- **Output:** nominally JSON, JSONL, CSV, YAML, or table. **In practice, use `-f json` and shape with `jq`.** `-f csv` is broken on nested responses (`people search -f csv` returns the whole result array inside one cell) and `-f table` is unreadable on wide objects.
- **Covers:** people search and enrich, company operations, contact, account, and deal management, sequence create, enroll, approve, abort, and archive, call logging, task creation, news, and credit usage. Since v2.2.0 also `agent run` and `agent poll` (Apollo's AI agent from a terminal), `sequences activity`, `emails content`, and `fields update`, plus waterfall and `--webhook-url` flags on enrichment.
- **After any upgrade, sign in again.** New commands can need new token scopes, and an old token answers them with `403`. `apollo auth login` fixes it.

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

**Not yet verified:** that Apollo's AI research columns decrement `ai_credit` specifically. The pool structure and the 800,000 limit make it the obvious candidate, but the dynamic-enrichment and record-collection tools we planned to measure it with left the MCP list before we could, and their replacement, `apollo_agent_research_and_analyze`, quotes its own cost before running. Read that quote rather than assuming a pool. Treat as strongly indicated, not measured.

## Preflight (zero credit)

Before real work on any lane, confirm access. The CLI and API expose `GET /v1/auth/health`, which returns `{ "healthy": true, "is_logged_in": true }` and spends **no credits**. It is the cleanest way to verify a session or key works before doing anything that costs money, the same role `apollo_users_api_profile` plays on the MCP preflight in `operator-context`.

## Lane 1 can disappear mid-task. The others do not.

**Observed 2026-08-31.** MCP writes succeeded at 10:38. Minutes later the same connection returned `This connector requires authentication.` In the same minute, `apollo auth whoami` was fine and a lane 3 REST call returned 200. The MCP session had expired; the CLI's stored OAuth token had not.

This is an **availability** difference, not a capability one, and it is the part of the four-lane model that is easiest to miss. Lane 1 is a connector session that can lapse without warning in the middle of a job. Lanes 2, 3, and 4 share a credential on disk with a much longer life.

Two practical consequences:

- **For anything long-running or unattended, prefer lanes 2 to 4, or lane 1 on a master API key.** A batch that takes 20 minutes should not depend on a session that can expire at minute 12.
- **A sudden authentication error on MCP does not mean the account is broken.** Check `apollo auth whoami` before you start debugging credentials. If the CLI is fine, it is the connector that needs reconnecting, and only the user can do that.

**The fix for unattended MCP is a master API key.** The MCP accepts an API key in an `X-Api-Key` header instead of a connector session, so there is no consent prompt and no session to lapse. Apollo's MCP docs now describe this headless route themselves. It must be a **master** key: a scoped key is rejected with `403` and `API_INACCESSIBLE`. Verified 2026-09-14: the same tool list as OAuth (73 of 74, missing only `apollo_survey_submit`) and identical search totals; 96 tools on the master key on 2026-11-08. Treat the key like a password, because a master key is not limited to any set of endpoints. **What is safe to run that way, and what must stay human:** `weekly-rhythm`, "Running checks unattended".

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
- **Assuming the CLI can run any search the MCP can.** It cannot. Tenure, years of experience, founded year, and market segments have no CLI flag on v2.2.0. They work on MCP and on lane 3 (REST with the CLI's token). Check the filter surface before picking the lane.
- **Running an old CLI against new docs.** NAICS, SIC, and headcount growth need v2.2.0 or later, and a fresh `apollo auth login` after upgrading.
- **Trusting `-f csv`.** It silently produces a one-cell dump on nested responses. Use `-f json | jq … | @csv`.
- **Reading only `.accounts` from `companies search`.** Results are split across `.accounts` and `.organizations`, and the split shifts by page. Reading one array quietly loses most of the result set.
- **Expecting `industry` from a search.** It comes back null on both lanes. Only `naics_codes` and `sic_codes` are populated pre-enrichment, and they are enough to grade composition for free.
- **Calling `apollo_sequences_update` with `active: false` and no steps.** The update is declarative, so every step you leave out is deleted. Read the sequence with `apollo_emailer_campaigns_show` and echo everything back, or use `apollo sequences abort`.
- **Pointing a connector at `/mcp-v2`.** It returns 404 since the router was retired. Use `https://mcp.apollo.io/mcp`, and check the tool names it actually lists rather than names from an older guide.
