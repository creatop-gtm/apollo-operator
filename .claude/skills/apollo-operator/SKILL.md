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
| **1. MCP** (library default) | Apollo appears as typed tools inside the model's context; the model picks a tool and fills a JSON schema | OAuth 2.0 (`https://mcp.apollo.io/mcp`) | **Through context, always** | Guardrailed, human-gated, in-conversation steps on small record counts |
| **2. CLI binary** | A normal terminal command an agent shells out to, or a human, script, or cron runs | OAuth 2.0 (`apollo auth login`) | **To disk** | Bulk search and enrich with common filters, composable pipelines, reproducible jobs |
| **3. REST + the CLI's own token** | `curl` against the same endpoint the MCP uses, authenticated with the token `apollo auth login` already stored | OAuth 2.0, **reuses the CLI's token, no API key** | **To disk** | **Bulk work that needs advanced filters.** The full MCP filter surface with the CLI's disk output. |
| **4. REST + an API key** | You build your own client against the HTTP endpoints | API key (`x-api-key`, `APOLLO_API_KEY`) | Wherever you send them | Back-end services and integrations that run without a logged-in user |

Apollo explicitly warns not to confuse API access with MCP. They are different lanes to the same place.

**Lane 3 is the one most people miss**, and it is the most useful discovery in this library. See "The CLI binary is not a superset, but its token is" below.

### The one-paragraph version

**MCP** puts Apollo *inside* the model as tools: safest, most expensive in context, and results always land in the conversation. **The CLI** puts Apollo *under* the agent as a composable command whose output can go straight to a file. **Lane 3** is the CLI's cheap, disk-bound access combined with the MCP's complete filter surface, and it needs nothing you do not already have after `apollo auth login`. **Lane 4** is for building software, not for running a motion.

## MCP vs CLI (the choice that actually comes up)

For an agent running Operator, the real decision is MCP or CLI, because both can be driven from inside a Claude Code session.

- **MCP** loads roughly 40 tool schemas into the model's context whether you use them or not. That is real token overhead, and it crowds the window, but in exchange you get typed, structured, in-conversation calls the model cannot fat-finger.
- **CLI** loads nothing upfront. The agent runs `apollo --help` on demand, then composes commands. It is deterministic (same command, same result), scriptable, version-controllable, and it runs with or without an AI in the loop.

The one-liner: **MCP puts Apollo *inside* the model as tools. The CLI puts Apollo *under* the agent as a composable command it can pipe, script, and re-run.**

### What the context saving actually is (measured)

Be precise about this, because the obvious guess is wrong.

| | Measured |
|---|---|
| One MCP tool schema (`apollo_mixed_people_api_search`) | ~29 KB, roughly 7,000 tokens. It is 1 of ~40. |
| CLI help, loaded on demand | `apollo --help` 1.9 KB · `people search --help` 3.0 KB · `sequences create --help` 0.9 KB |
| **Response payload, same search both lanes** | **Byte-identical.** Same totals, same ids, same obfuscated preview shape. |

So the CLI does **not** return leaner data. Its advantage is two other things:

1. **Schema overhead.** You pay for ~40 tool definitions on MCP whether or not you use them.
2. **Results can bypass context entirely.** `> leads.json` means the payload never reaches the
   model. MCP results always pass through context. In a real run, 3,238 people came down as 4 MB
   on disk with nothing in the window. That pull does not fit in a context window over MCP, so
   this is not an optimization, it is the difference between possible and impossible.

The composability is the other standout: search, filter, dedupe, and save chain in one line, which
is exactly the shape of List list work. MCP tool calls do not compose like that. Just note that
`-f csv` is broken on nested responses, so the pipeline ends in `jq`, not in `-f csv`. See
`references/cli-recipes.md`.

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
calls and no shell improvisation. The one exception is stopping a live send, which MCP cannot do at
all (see below).

**3. Is this bulk work whose output belongs on disk?** Then **lane 2 (CLI binary)** if the common
filters cover it, **lane 3** if they do not. Paginating a few thousand people, writing every stage
to a file, and re-running the job later are all things these lanes do natively and MCP cannot do
without flooding the window. See List,
`apollo-list-builder`.

**4. Otherwise, default to MCP.** For a handful of records in conversation, the typed surface is
easier to get right, and the schemas are already loaded.

**Lane 4 (API key)** does not appear in this decision at all. It is for building a service that runs
without a logged-in user. If you are running a motion, you want one of the first three.

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
| **Stop a live send** | 2, 3, or 4 — **any lane except MCP** |
| Building a separate service | 4 (API key) |

### The kill switch: everywhere except MCP

`apollo sequences abort --id <id>` deactivates a live sequence and stops sending, and
`sequences archive` retires a finished one. Both also exist on REST as
`POST /emailer_campaigns/{sequence_id}/abort` and `/archive`, so lanes 2, 3, and 4 can all stop a
send.

**The MCP is the one lane that cannot.** So if you run an entirely MCP-based motion, you can start
a campaign you have no way to stop from inside the conversation. Have the CLI installed before you
activate anything, or know the REST call. Not during an incident.

**Status:** the library is MCP-native by default and stays that way for anything guardrailed. The
CLI-backed variants of the bulk search and grade steps are live in
`apollo-list-builder` as of v1.2, with verified
commands in `references/cli-recipes.md`.

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

**Not yet verified:** that running a dynamic prompt-execution field decrements `ai_credit` specifically. The pool structure and the 800,000 limit make it the obvious candidate, but the enrichment tool that runs those fields is not exposed on any headless lane, so it could not be tested from a script. Treat as strongly indicated, not measured.

## Preflight (zero credit)

Before real work on any lane, confirm access. The CLI and API expose `GET /v1/auth/health`, which returns `{ "healthy": true, "is_logged_in": true }` and spends **no credits**. It is the cleanest way to verify a session or key works before doing anything that costs money, the same role `apollo_users_api_profile` plays on the MCP preflight in `operator-context`.

## Lane 1 can disappear mid-task. The others do not.

**Observed 2026-08-31.** MCP writes succeeded at 10:38. Minutes later the same connection returned `This connector requires authentication.` In the same minute, `apollo auth whoami` was fine and a lane 3 REST call returned 200. The MCP session had expired; the CLI's stored OAuth token had not.

This is an **availability** difference, not a capability one, and it is the part of the four-lane model that is easiest to miss. Lane 1 is a connector session that can lapse without warning in the middle of a job. Lanes 2, 3, and 4 share a credential on disk with a much longer life.

Two practical consequences:

- **For anything long-running or unattended, prefer lanes 2 to 4.** A batch that takes 20 minutes should not depend on a session that can expire at minute 12.
- **A sudden authentication error on MCP does not mean the account is broken.** Check `apollo auth whoami` before you start debugging credentials. If the CLI is fine, it is the connector that needs reconnecting, and only the user can do that.

## Hand back a URL whenever a human has to look at something

Headless work still ends in a person's browser more often than the lanes suggest, and some things **can only** be checked there: sequence bodies come back empty from every read path, and dynamic AI field prompts are not readable at all. Whenever you create or change something in one of those, print the link rather than the id.

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
- **Assuming the CLI needs an API key.** It is OAuth (`apollo auth login`). The API key is the third lane, for building your own client.
- **Assuming the CLI can run any search the MCP can.** It cannot. NAICS, SIC, tenure, headcount growth, founded year, and market segments are MCP-only. Check the filter surface before picking the lane.
- **Trusting `-f csv`.** It silently produces a one-cell dump on nested responses. Use `-f json | jq … | @csv`.
- **Reading only `.accounts` from `companies search`.** Results are split across `.accounts` and `.organizations`, and the split shifts by page. Reading one array quietly loses most of the result set.
- **Expecting `industry` from a search.** It comes back null on both lanes. Only `naics_codes` and `sic_codes` are populated pre-enrichment, and they are enough to grade composition for free.
