---
name: sending-infrastructure
description: "Build the sending stack from zero before you send: dedicated domains separate from your primary, mailboxes, SPF/DKIM/DMARC/MX, redirect, and warmup. Includes the volume math (mailboxes and domains per daily send target). Use for first-time setup or when scaling adds mailboxes."
---

# Sending Infrastructure (Infrastructure)

Build the sending stack before you send a single email: dedicated domains, mailboxes, DNS, and warmup, all separate from your primary domain. `apollo-deliverability` (Infrastructure) keeps this asset alive. This skill creates it in the first place. If you have no dedicated domains or mailboxes yet, start here, because everything downstream assumes they exist.

## When to use

- Setting up outbound for the first time, with nothing but a primary company domain.
- Scaling: an existing campaign is winning and you need more mailboxes to send more (see the volume math below).
- A domain got burned and you are standing up a fresh sending stack to replace it.

## Principle

From `../operator-context/references/outbound-principles.md`: protect the asset. The infrastructure *is* the asset. Two rules drive every choice here:

- **Never send cold from your primary domain.** One reputation, and it is the company's real one. Cold outreach goes out on dedicated domains that can absorb the risk. If a sending domain degrades, you retire it, not the business's main email.
- **Build it three weeks before you need it.** Warmup takes 14 days minimum, 21 optimal, and it cannot be rushed. The single most common reason a launch slips is infrastructure that was ordered the week it was needed. Order early.

## The mental model: a separate sending stack

```
primary domain (brand.com)      → real email, never cold outreach
        │  (redirect)
        ▼
sending domains (getbrand.com, trybrand.com, ...) → cold outreach only
        │
        ├─ mailbox 1  ── warmup + up to 25 cold/day
        ├─ mailbox 2  ── warmup + up to 25 cold/day
        └─ mailbox 3  ── warmup + up to 25 cold/day
```

Every sending domain redirects to the main site, so a prospect who clicks lands on the real brand. The mailboxes on those domains do the cold sending, capped and warmed. The primary domain stays clean.

## Step 1: size the stack (volume math)

Work backwards from how many cold emails a day you actually need.

- **Ceiling per mailbox: 25 cold sends per day.** Never plan above it.
- **Mailboxes per domain: 2 to 3.** More than that concentrates risk on one domain.
- The math: `mailboxes = target daily sends ÷ 25`, then `domains = mailboxes ÷ 3` (round up).

Worked example: you want 300 cold sends a day. 300 ÷ 25 = 12 mailboxes. 12 ÷ 3 = 4 sending domains, 3 mailboxes each. Do not solve 300/day by pushing 4 mailboxes to 75 each. That burns them. Volume comes from more mailboxes, never from a higher per-mailbox rate.

Always build a backup pool on top of the working number (see `apollo-deliverability`): warm more mailboxes than you send from, so when one degrades you swap instead of stopping.

## Step 2: get the sending domains

Buy domains that are clearly the same brand, so the redirect and the from-name read as legitimate: `getbrand.com`, `trybrand.com`, `brandhq.com`, `brand-hq.com`. Keep them close to the real name. Avoid spammy variants and unusual TLDs, which hurt deliverability on sight.

Two acquisition paths, pick one:

- **Do it yourself at a registrar** (Cloudflare, Namecheap, Porkbun, and similar). Most control, lowest cost, more setup work. You buy the domains, set DNS by hand (Step 3), and connect mailboxes (Step 4).
- **Buy provisioned domains and mailboxes.** Apollo sells domain-and-mailbox provisioning, and **the MCP can now purchase mailboxes directly** (see Step 6). Third-party mailbox providers sell the same thing: pre-configured domains and inboxes with DNS handled for you. Zapmail and Mailpool are two common ones. Less control, less setup, a monthly cost. Good when you want to move fast.

Either way, the requirements in Steps 3 to 5 are the same. Provisioned just means someone else does Step 3 for you.

## Step 3: set DNS on every sending domain

This is the step that decides inbox versus spam. Each sending domain needs four things set at its DNS host. The exact records, what each does, and how to check them are in `references/dns-records.md`. In short:

- **MX:** routes replies to the mailbox. Without it, replies vanish.
- **SPF:** authorizes your sending service to send as the domain.
- **DKIM:** cryptographically signs mail so receivers trust it came from you.
- **DMARC:** tells receivers what to do with mail that fails the above, and it is where reputation is won or lost.
- **The redirect:** point the sending domain (a 301) at the main website, so clicks land somewhere real.

Skipping any of SPF, DKIM, or DMARC is the fastest way into the spam folder. Do not send until all three pass.

## Step 4: create the mailboxes

Two to three per domain, named like real people (`first@getbrand.com`, `first.last@getbrand.com`), not `sales@` or `info@`, which read as blast accounts. Set a real display name, a signature, and a professional photo where the provider allows it. Each mailbox is a person, so it should look like one.

## Step 5: turn on warmup (start the clock)

Warmup runs on every mailbox before it sends anything cold, and it never turns off, it runs in parallel with live campaigns forever.

- **14 days minimum, 21 optimal.** No exceptions. This is the non-negotiable that everything else waits on.
- **Warmup volume:** 15 to 20 warmup emails per day per mailbox, weekdays.
- Then ramp cold sends gradually on top: 5 to 10/day in week one, reaching the 25/day ceiling over 4 to 5 weeks.
- Keep the combined total (warmup plus cold) under 50/day/mailbox, always.

The clock starts the day warmup starts. This is why Step 1's "build it three weeks early" matters: the infrastructure has to sit and warm while you build the list and the sequence.

## Step 6: provision and verify with the MCP

**This changed.** The MCP used to be read-only on infrastructure. It can now **purchase mailboxes** on a domain your team already owns, which means an agent can stand up part of the sending stack rather than only inspect it.

### Purchasing mailboxes (`apollo_email_account_purchase_create`)

Real money, real provisioning, and irreversible from the tool. Treat it with the same ceremony as sending email.

1. Call `apollo_domain_purchase_index` first to get a `domain_purchase_id`. **A mailbox can only be provisioned on a domain the team already owns**, and the mailbox address must match that domain. Never guess the id.
2. Pick a type. Cost per mailbox, in unified credits: **shared 300 · google 800 · outlook 1500.** All mailboxes in one request must share a type, so the total is always count × per-type cost. Legacy export-credit teams pay roughly a fifth of that.
3. **Say the confirmation exactly as the tool requires**, with real numbers: "Purchasing [N] [type] mailbox(es) will consume [N × cost] credits. Do you want to proceed?" Do not paraphrase it, and do not proceed without an explicit yes.
4. Provisioning is asynchronous. New mailboxes start `pending_setup` and become `active`. Poll `apollo_email_account_purchase_index`.

**Setting DNS is still manual**, so Steps 3 to 5 do not go away. **Buying domains is changing:** Apollo's MCP router documents `apollo_domain_purchase_create`, at **1,500 unified credits per domain**, with a full WHOIS registrant required and no undo. It is not in the published tool list and has not been tested here, so treat it as coming rather than available, and compare the credit cost against a registrar before choosing it.

**Do the arithmetic before you suggest this.** Three google mailboxes is 2,400 credits, which on a 4,000-credit plan is most of a month's enrichment budget. Buying mailboxes and enriching a list compete for the same pool, and an operator who does not realise that will run out mid-campaign. Apply the 85% rule from `apollo-list-builder`.

### Verifying what exists

The read paths are unchanged, and they are the preflight `apollo-go-live` depends on:

- `apollo_domain_purchase_index`: for domains provisioned through Apollo, returns each domain's SPF, DKIM, and DMARC diagnostics plus the mailboxes on it. Use it to confirm DNS is actually passing before you send, not to guess.
- `apollo_email_account_purchase_index`: lists Apollo-provisioned mailboxes and their status (`pending_setup`, `active`, `inactive`). A mailbox in `pending_setup` is not ready. Do not enroll it.
- `apollo_email_accounts_index`: lists mailboxes connected by OAuth (the ones you set up yourself and linked), with the `default` sender flag. This is the list `apollo-go-live` draws the sender from.

### Checking authentication on any connected domain

`apollo_email_domain_diagnosis_authentication_status` reports SPF, DKIM, and DMARC for the domains behind **every mailbox connected to Apollo**, not only the ones Apollo provisioned. For each domain it returns the raw DNS values (`spf_values`, `dmarc_values`, `dkim_host`) and an `analysis` object with a verdict per check, `good`, `warning`, `error`, or `pending`, plus `domain_created_on` for domain age and `healthy_since` for how long it has passed.

Four limits decide how far to trust it, all verified 2026-09-14:

- **It only sees domains with a mailbox connected to Apollo.** On an account whose cold mailboxes live in another sending platform, it reported one domain, the team's primary, and none of the sending domains. It is not a general DNS checker.
- **It reports the last stored check, not a fresh one.** The one domain returned had last been checked almost eight weeks earlier. Read `authentication_status_updated_at` before trusting a verdict, and re-check DNS directly after any change.
- **At most 10 domains per call**, issue-first. Never conclude that every domain is healthy from one response; pass `domain` to check a specific one.
- **It is not in the published tool list.** It surfaced through the MCP v2 router and runs only when called directly on the v1 MCP with a master API key (see `apollo-operator`).

It covers authentication only: no blacklists and no inbox placement. If your mailboxes live outside Apollo, the checks live at your DNS host and in the checklist in the references file.

## Output and handoff

A sending stack that is: separate from the primary domain, correct on SPF, DKIM, DMARC, and MX, redirecting to the main site, and warmed for at least 14 days. Once it passes, `apollo-deliverability` (Infrastructure) maintains it, and `apollo-go-live` uses it to send. Until it passes, nothing goes out.

## Common mistakes

- **Sending cold from the primary domain.** The one mistake you cannot walk back. Always a dedicated stack.
- **Ordering infrastructure the week you want to launch.** Warmup needs 14 to 21 days. Order three weeks early or the launch slips.
- **Solving for volume with a higher per-mailbox rate.** 25/day is the ceiling. More volume means more mailboxes, not hotter ones.
- **Skipping DMARC** (or SPF, or DKIM) to "set it up later." Later is after you have already trained receivers to send you to spam.
- **Generic mailbox names** (`sales@`, `info@`, `team@`). They look like blast accounts and deliver like them. Name mailboxes like people.
- **Enrolling a `pending_setup` mailbox.** Check status first with `apollo_email_account_purchase_index`. Pending is not ready.
