---
name: apollo-account-audit
description: "Audit an existing Apollo account with reads only: mailboxes and daily limits, domain authentication, live sequences that are not actually sending, Bounce Guard, performance history, lists, contacts, credits, Context Center, and website tracking. Graded findings, no changes. Use before running outbound on an inherited or client account, or when a campaign is not sending."
---

# Apollo Account Audit (Foundations)

Before you run anything on an Apollo account you did not build, find out what state it is in. This skill checks the account with **reads only** and hands back graded findings. It changes nothing: every fix waits for the operator.

It exists because accounts drift quietly. A sequence can sit active for weeks sending nothing, cold email can go out from the company's primary domain, and a website tracker can be installed without ever receiving data, and none of it shows up until someone goes looking. On the first real account this was run on, it found all three in under an hour.

## When to use

- Taking over an existing Apollo account, your own or a client's.
- Onboarding a client account through a partner seat, before touching anything.
- "Why isn't this campaign sending?"
- Quarterly, as a standing health check.

## The checks

Run them in this order, since the early ones explain the later ones. Grade each area **Good**, **Watch**, or **Fix**.

| # | Area | How | What to look for |
|---|---|---|---|
| 1 | Access and team | `apollo_users_api_profile`, `apollo_users_search` | Who is on the account, and whether you are reading it as the right user |
| 2 | Mailboxes | `apollo_email_accounts_index` | Sending from the company's **primary domain** is a Fix. So is a free-mail domain (`is_free_domain`) |
| 3 | Daily limits | `apollo email-accounts list` (CLI): `email_daily_threshold`, `max_outbound_emails_per_hour`, `seconds_delay_between_emails`, warmup status | Compare against the ceiling in `apollo-deliverability`. A limit far above it is a Fix even when volume is low today. **Do not use the analytics metric `email_daily_limit`**: it reported 775 for a mailbox set to 25 |
| 4 | Domain authentication | `apollo_email_domain_diagnosis_authentication_status` | SPF, DKIM, and DMARC verdicts. Read the check's date first; see the limits in `sending-infrastructure` |
| 5 | Live sequences | Sequence search for active ones, then `apollo_emailer_campaigns_show` on each, then `apollo_emailer_messages_search` in `metadata_mode` | An active sequence with enrolled contacts and nothing scheduled. Check touch `status` first (below) |
| 6 | Bounce Guard | `auto_pause_enabled` and `auto_pause_config` on each sequence object | Guard on, and the sequence's real volume against `min_volume`. Below it, the guard never evaluates |
| 7 | Performance history | Analytics grouped by `emailer_campaign_id`, all time, then the sequence's `contact_statuses` | Any sequence with a real bounce rate above 4%, taken from contact statuses rather than analytics `num_emails_bounced` (which reported 6 bounces where the UI showed 0). Also archived sequences, which analytics lists and search does not |
| 8 | Idle and test sequences | Sequence search | Inactive tests and abandoned drafts. Clutter, not risk, but a sign nobody owns the account |
| 9 | Lists | `apollo labels list` (CLI) or `apollo_labels_index` | Count, empty lists, age, and test lists. The MCP returns them under `data`, not `labels` |
| 10 | Contacts | `apollo_contacts_search` total, then a duplicate sample | Sort by `contact_created_at`, pull a few hundred, count repeated emails. Bulk create does not dedupe, so recent uploads are where duplicates hide |
| 11 | Credits | `apollo usage credits` or `apollo_usage_stats_credit_usage_stats` | Any pool at zero, and whether lead credits cover the next planned list. Name the pool, always |
| 12 | Context Center | `apollo_context_center_show` | Set up or empty, and whether it still matches the business |
| 13 | Website tracking | `apollo_website_visitor_domain_tracker_index` | Installed but `data_received: false` means the script is not live. `is_contact_level_tracking_enabled` says whether person-level signal is possible |
| 14 | Schedules | `apollo_emailer_schedules_index` | Send window and timezone against the policy you run to |
| 15 | Compliance | Contact search response, sequence objects, a phone sample | `disable_eu_prospecting` state against the ICP's countries, unsubscribe counts and `opt_out_rate`, and whether phone numbers carry DNC screening. The unsubscribe link setting is UI-only, so list it as a manual check. See `compliance.md` |

### The finding that looks like a performance problem and is not

**A sequence can be active, with contacts enrolled, and send nothing, because its touches are not approved.** Verified 2026-09-14: an active sequence with 5 enrolled contacts had scheduled zero emails in two weeks, and all six of its touches carried `status: "to_be_reviewed"`. Apollo's schema says `approved` means ready to send; `to_be_reviewed` is for workflows that need a human to review first. It reads as a dead campaign in any report. Check touch status before diagnosing copy, lists, or deliverability.

Approving touches starts real email, so the audit reports it and stops. The operator decides.

## Grading and output

Give each area its grade and one line of evidence. Then:

1. **Fix first:** anything that is sending unsafely, or sending nothing while looking live.
2. **Watch:** things that are fine today and will not be tomorrow (a limit above the ceiling, a guard that never evaluates at this volume, credits that will not cover the next list).
3. **Tidy:** test sequences, empty lists, stale Context Center.

End with the three actions that matter most, in order, and what each needs from the operator.

**Stop at findings.** No approvals, archives, deletions, limit changes, or list edits from this skill, however obvious the fix looks.

## What the audit cannot see

Say these out loud in the report, so a clean audit is not read as a clean bill of health:

- **Mailboxes and domains in another sending platform.** The domain check only covers mailboxes connected to Apollo.
- **Blacklists and inbox placement.** Neither is exposed.
- **Reply text.** Counts and classes only.
- **Anything behind the unpublished tools** (domain diagnosis among them) without a master API key on the v1 MCP. See `apollo-operator`.
- **Team-level settings changed in the UI**, such as Bounce Guard defaults, beyond what each sequence reports.

## Privacy

An audit of someone else's account is full of their data: prospects, list names, campaign names, results. Keep the output with that account, never in anything shared or published, and describe findings by pattern when talking about them elsewhere.

## Common mistakes

- **Reading an active, silent sequence as a performance result.** Check touch status first.
- **Trusting a domain verdict without its date.** It is the last stored check, not a live one.
- **Calling low volume safe.** A mailbox limit far above the ceiling is a Fix before it is ever used.
- **Listing campaigns from search.** Archived sequences still carry history in analytics.
- **Fixing things mid-audit.** The operator decides, after seeing the whole picture.
