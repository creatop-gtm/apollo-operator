---
name: campaign-reporting
description: "Report a campaign's real results from Apollo's own analytics: per sequence, step, variant, and mailbox, plus meetings booked and opportunities, read-only. Use weekly, at the end of an experiment, or before deciding to scale, change, or stop a campaign."
---

# Campaign Reporting (Iterate)

Which sequence, which step, which variant, and which mailbox is actually producing replies and meetings. This skill reads it straight from the operator's own Apollo account and lays it out so a decision can be made: scale it, fix it, or stop it.

It is **read-only**. It counts meetings booked and opportunities created, and it never creates, moves, or edits anything. Managing the pipeline stays out of scope, per `operator-context`.

## When to use

- Weekly, as the Friday retrospective in `weekly-rhythm`.
- At the end of an experiment, to read the result `experiment-design` set up.
- Before deciding to scale, change, or stop a campaign.
- Monthly, to see the whole account at once.

## Where the numbers come from

All verified live on 2026-09-14.

| Source | What it gives | Use it for |
|---|---|---|
| `apollo_analytics_sync_report` | 70 metrics across 55 dimensions, filtered by sequence, user, or team, over a preset or custom date range | The report itself |
| `apollo_emailer_messages_search` with `emailer_campaign_id` and `display_mode: "metadata_mode"` | The status counts the Apollo UI shows for one sequence | Cross-checking the analytics totals |
| `apollo emails search --stats replied` (CLI) | Every replied message, each with Apollo's `reply_class` | Reply quality, before `positive-reply-scoring` |
| `apollo_emailer_campaigns_activity_feed` | One contact's timeline in a sequence | Explaining a single odd result |

The analytics call takes `metrics`, `group_by`, optional `pivot_group_by`, `filters` (`emailer_campaign_ids`, `user_ids`, `team_ids`), `date_range` (a preset such as `last_30_days`, or `range_start` and `range_end`), and `sort`.

## Build the report

### 1. Start from analytics, not from the sequence list

Group by `emailer_campaign_id` over the reporting window. **Do not build the list of campaigns from sequence search.** On a real account, search (MCP and CLI alike) returned 9 sequences while analytics reported 14, including one with 40 sends. **Archived sequences keep their history in analytics and vanish from search** (confirmed in the Apollo UI 2026-09-14), so a report that starts from search silently loses every archived campaign.

Useful metrics: `num_emails_sent`, `num_emails_delivered`, `num_emails_replied`, `num_emails_bounced`, `num_emails_unsubscribed`, `num_contacts_emailed`, `num_contacts_replied`.

### 2. Break each live campaign down by step and variant

Filter to one sequence with `emailer_campaign_ids`, then:

- **By step:** `group_by: ["emailer_step_id"]`. Shows where the thread goes quiet.
- **By variant:** `group_by: ["emailer_template_id"]`. Returns readable labels such as "Step 1a" and "Step 1b". Grouping by `emailer_touch_id` gives the same split but only as ids, so use templates for anything a person reads.

### 3. Check the mailboxes

`group_by: ["email_account_id"]` with `num_emails_sent` and `num_emails_bounced`. Read each mailbox's allowed limit from `apollo email-accounts list` (`email_daily_threshold`), not from the analytics metric `email_daily_limit`, which reported 775 for a mailbox set to 25. A mailbox carrying the bounces is a deliverability problem wearing a copy problem's clothes; hand it to `apollo-deliverability`.

### 4. See who responds

`person_seniority` or `person_title_unanalyzed` show which roles reply. **Always filter these to the campaign's sequence ids** (see the traps below), or one-off emails get counted.

### 5. Count outcomes, read-only

`num_all_meetings_scheduled_via_email` and `num_opportunities`, grouped by **month or user**, not by sequence. Opportunities cannot be grouped by sequence at all: the request returns only a warning and no data.

### 6. Cross-check before anyone sees a number

Pick the busiest sequence and compare its analytics totals with `apollo_emailer_messages_search` in `metadata_mode`. They should match. On a real account they did exactly (2 sent, 2 delivered, 2 opened, 1 replied), and the Apollo UI agreed. If they do not, say so in the report rather than picking one.

### 7. Read reply quality

Pull `apollo emails search --stats replied` and count `reply_class`. Classes seen on a real account: `willing_to_meet`, `follow_up_question`, `person_referral`, `not_interested`, `unsubscribe`, and none at all (the largest group). Use it to sort, then run `positive-reply-scoring` for the metric that matters.

## The report

Keep it to what a decision needs:

| Campaign | Sent | Bounce % | Reply % | Positive reply % | Meetings (month) | Call |
|---|---|---|---|---|---|---|

Then, per live campaign: the step where replies stop, the variant ahead (with its send count next to it), the mailbox bouncing most, and one recommended action. End with anything that did not reconcile in step 6.

## Traps, all verified 2026-09-14

- **The analytics result is a markdown table, not structured data.** It comes back in a `summary` string. Parse the table rows; do not look for a JSON array.
- **An incompatible metric wipes the report.** Asking for `num_opportunities` grouped by sequence returned a warning and no table at all, not a table with one column missing. Request outcome metrics in a separate call.
- **One-off emails inflate replies in time and people groupings.** Grouped by month, one account showed 30 sent and 19 replied, because replies to individual emails sent outside any sequence are counted. Grouped by sequence, they drop out. Filter by sequence ids whenever the question is about campaigns.
- **Analytics "bounced" is not the UI's bounced count.** On one archived sequence analytics reported 40 sent, 34 delivered, and 6 bounced, while the Apollo UI showed 34 delivered and **0 bounced** contacts. The 6 is exactly sent minus delivered. Report delivered and sent from analytics, and take bounces from the sequence's contact statuses (`contact_statuses.bounced`, `hard_bounced`) or the UI, and say which you used.
- **Opens are not a signal here.** Cold campaigns in this library run with tracking pixels off, so `num_emails_opened` is either empty or meaningless. Do not report it as a result.
- **Small numbers are not results.** A variant that "won" one reply to zero on one send each has told you nothing. Put the send count next to every rate, and use the sizing rule in `experiment-design` before calling a winner.
- **Replies are not positive replies.** The reply rate includes unsubscribes and "not interested". Never present it as the outcome.

## Common mistakes

- Listing campaigns from sequence search and missing the archived ones.
- Quoting analytics `num_emails_bounced` as a bounce rate without checking it against contact statuses.
- Reporting opens.
- Declaring a variant winner on a handful of sends.
- Grouping replies by month without filtering to sequences, and reporting a reply rate above 100%.
- Treating a quiet report as a quiet campaign. If a live sequence shows nothing scheduled, that is an `apollo-account-audit` finding, not a performance result.
