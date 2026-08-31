# Apollo Operator

**An opinionated go-to-market operator for Apollo.io, built as open-source Claude Code skills.**

The Apollo MCP can run your entire outbound motion. It just has no idea how. Point it at your account raw and it will run any search, build any sequence, and send any email you ask for, with no reason to tell you the targeting is off, the copy reads like a robot, or the send will burn your domain. It does what you say, not what you meant.

Apollo Operator is the knowledge layer. It is a GTM system of 17 skills that turn the raw MCP into a guided outbound motion: understanding the business, who to target, how to build and grade a list, how to write sequences that get replies, how to build the sending infrastructure and take campaigns live without torching your domain, and how to tell whether any of it is actually working.

## What it runs

One router skill (`operator-context`) loads first and hands off by phase. A phase is a kind of work, not a step you finish once.

**Foundations** (always on)
- `operator-context` — the tool map, the rules that protect your sending reputation, and the routing.
- `apollo-cli` — the four ways to run Apollo headless and when to reach for each.

**Context** — `business-brief`: one readable `brief.md` any agent can read once and understand the business. Optionally projected into Apollo's own Context Center.

**Targeting** — `apollo-icp-builder`: turn an ICP into a real Apollo search, size several angles for free, and pick one on numbers instead of instinct.

**Infrastructure** — `sending-infrastructure`, `apollo-deliverability`: dedicated domains, mailboxes, DNS, warmup, the volume math, and the bounce rules. **This phase sits early on purpose.** Warmup is a 14 to 21 day wall-clock wait, so the clock has to start while you are still building the list. Teams that leave it until the copy is approved discover a finished campaign they cannot send for three weeks.

**List** — `apollo-list-builder`, `list-quality-scorecard`: search, grade composition, suppress, enrich, verify, and grade. Nothing reaches a sequence until it passes.

**Message** — `apollo-sequence-builder`, `sequence-reviewer`, and optionally `apollo-multichannel`: short sequences matched to the persona, reviewed for spam risk, plus LinkedIn, call, and one-off email steps.

**Launch** — `apollo-go-live`: enroll, a human approves, contacts pull on reply, and there is a kill switch for when something goes wrong.

**Iterate** — `positive-reply-scoring`, `experiment-design`, `weekly-rhythm`, `re-engagement`: measure intent, run clean experiments, keep a cadence, and go back to the 90% who never replied.

**Signals** (cross-cutting) — `apollo-signals`: find the reason to reach out now.

## The four lanes

Most people know Apollo has an MCP and an API. There are four routes to the same data, and picking the right one decides how much context you burn and whether a job is possible at all.

| Lane | Auth | Results go | Best for |
|---|---|---|---|
| **MCP** | OAuth | Through context | Guardrailed, human-gated steps |
| **CLI binary** | `apollo auth login` | To disk | Bulk work with common filters |
| **REST + the CLI's own token** | Reuses the CLI's token, **no API key** | To disk | **Bulk work needing advanced filters** |
| **REST + API key** | API key | Anywhere | Building a separate service |

The third is the one most people miss. `apollo auth login` stores an OAuth token, and that token authenticates against the same REST endpoint the MCP uses. So you get the **complete MCP filter surface** (NAICS, tenure, headcount growth, department headcounts) while writing results straight to disk. Measured against MCP on an identical query: same total, 3 MB to disk, nothing in the context window.

Neither surface contains the other, which is worth knowing before you commit to one. Record collections and mailbox purchasing are MCP-only. Deals, custom-field creation, rate-limit stats, and stopping a live sequence are all REST and CLI but **not** MCP.

The rule it collapses to: **search wide off-context, act narrow on MCP.**

## Built by operators, tested live

We build this on our own Apollo account, and running it is what finds the problems. Nothing here comes from reading the docs: every item below cost us something to learn. Some of what testing caught, now encoded:

- A title filter for `founder` also returns **Founding Engineer, Founding Designer, and Founding GTM** — 9 to 14% of a raw list, and none of them buyers.
- Apollo's pagination **overlaps**, so 3,238 rows contained 2,933 unique people. Per-company capping does not remove them, and you pay per duplicate at enrichment.
- A data provider's **catch-all flag is not a verification**. On one list it reported clean on all 846 addresses while an independent verifier found 39% catch-all.
- **MCP cannot stop a live sequence.** Every other lane can. Set that up before you launch, not during an incident.
- Running several angles against one ICP **overlaps silently**: 85 people in a second list had already been enriched for the first.
- A title filter quietly inflated a search from 679 real matches to over 91,000.
- **A provider's own `verified` status is not deliverability.** 997 addresses graded `verified` came back from an independent verifier as 564 safe to send, 413 catch-all, and 3 that would have bounced. Verification is a separate control, and it belongs before you write copy, not after.
- **Bulk contact creation does not deduplicate**, despite documenting that it does. The same five contacts submitted twice produced two records each. Never retry on a timeout.
- **Enrollment can fail silently.** Passing several contact ids to the CLI joins them into one string, so Apollo finds nothing and returns exit 0 with an empty campaign. Assert on the response body, never the exit code.
- **You cannot read a sequence back.** Four separate read paths return empty bodies, so the last check before activation is a human opening the Apollo UI. Every skill that creates one hands you the link.
- **You pay per record attempted at enrichment, not per email found**, and re-enriching a record you already paid for charges again.
- **Credits are eleven separate pools**, not one balance, and the AI pool is 200 times the size of the contact-data pool. Data is the scarce resource, not generation.

## It explains itself

Outbound has a lot of jargon and a lot of ways to quietly waste money, so the operator is built to work with someone who has never done this before. It names what an artifact is missing and why, glosses jargon on first use, gives every number a baseline, and states what a step costs **before** running it.

If a step would spend more than 85% of your remaining credits, it stops and offers you the choice rather than just quoting the number.

## Requirements

- Claude Code
- The Apollo MCP connected
- The Apollo CLI (`brew install apolloio/apollo-io-cli/apollo-io-cli`) for bulk work and the kill switch
- An email verification service. Pick your own; this library does not choose one for you.

## Use

Open this folder in Claude Code. The skills live in `.claude/skills/` and load automatically. Describe what you want ("help me define my ICP and build a list on Apollo") and `operator-context` routes you.

## Philosophy

Read `.claude/skills/operator-context/references/outbound-principles.md`. The short version: outbound is a system, relevance beats volume, deliverability is sacred, humans approve the copy, and these skills surface risks but never block. You decide.

## Who it is for

Founders and go-to-market teams running outbound on Apollo who want a system, not a black box. Use it free to run your own motion. If you would rather have it built and run for you, that is what we do at Creatop.

https://creatop.net
