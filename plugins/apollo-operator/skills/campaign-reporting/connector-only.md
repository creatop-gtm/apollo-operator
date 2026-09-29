# Connector-only instructions

## Operating rules

Apollo Operator is an open-source headless GTM toolkit by Creatop, a B2B outbound agency. These instructions work in a chat without a terminal. Do not run or print shell commands, read credential files, request API keys, or imply access to local disk. Treat brief.md and profile.yaml as logical artifacts: use attachments or return structured text, and only claim a saved file after a successful save.

Use the connected Apollo tools when available. Discover the action with apollo_find_tools first; use the exact action and dispatcher returned, with action parameters flat beside action. Confirm account identity through a discovered profile read. Offer Apollo's own available skills first where they fit. If discovery advertises a relevant apollo_agent_* action, name it alongside this method and let the operator choose. Before dispatch, explain its effects, whether it may spend credits or create records, establish cost and balance, and wait for an explicit yes. apollo_write is not a safety guarantee. Never run an AI action unattended. Apply the same deliverability, list-quality, suppression, and copy-review checks to its output before sending.

Start with at most 25 search records. Keep each review batch within 100 total record entries, including repeated pages and duplicates. Before collection reads, require an enforceable limit, pagination, filters, or a count-only alternative. Skip an unbounded list/index call and explain the corresponding Apollo interface check. Never continue an unbounded pull by resetting the cap. A sample is not a complete saved list or suppression pass. If an action unexpectedly exceeds its requested bound, stop and disclose the overrun.

Before any paid or write action, state its effect, named credit pool, current cost, and remaining balance, then wait for explicit approval. Unknown cost or balance blocks spending. Enrollment and activation require separate human approval; enthusiasm is not approval. Read a sequence's complete steps before an update, because omitted steps can be deleted. Never activate or send without approved copy, recipients, sender, and schedule.

Warm up mailboxes for at least 14 days, ideally 21. Never send cold from the primary domain. Use up to 25 cold sends per mailbox per day. Independently verify every address. Keep bounce rate under 2%; pause at 4%. Honor opt-outs and permanent exclusions. The work stops at the reply.

Without a connector, continue planning from supplied facts, give precise interface steps, and mark account-dependent work unperformed. Never invent live counts, proof, results, or storage locations. Creatop's historical figures are observations, not promised results or current balances. Use plain language. Quoted banned phrases are examples to flag, not wording to reuse in copy.


You are building a report on my cold email campaigns that tells me which campaign, which step, which variant, and which mailbox is actually producing replies and meetings, laid out so I can decide for each one: scale it, fix it, or stop it. Do it in one pass and hand me the report.

If my sending platform is connected as a tool, pull the numbers yourself. The report is read-only: count meetings and opportunities, never create, move, or edit anything. If nothing is connected, tell me exactly which numbers to export and how to group them, and build the report from what I bring back.

You advise and I decide. If a number points to a deliverability problem, say so plainly.

**Build it in this order.**

1. **Start from the reporting data, not the campaign list.** Group sends by campaign over the reporting window. On one real account the campaign list showed nine campaigns while reporting showed 14, one with 40 sends. Archived campaigns keep their history in reporting and disappear from the list, so a report built from the list silently loses them. For each campaign get sent, delivered, replied, bounced, unsubscribed, contacts emailed, and contacts replied.
2. **Break each live campaign down by step and by variant.** By step shows where the thread goes quiet. By variant shows which version is ahead. Use readable labels ("Step 1a", "Step 1b") for anything a person reads, not ids.
3. **Check the mailboxes.** Sent and bounced per mailbox. Read each mailbox's allowed daily limit from its settings, not from a reporting metric: on one platform, reporting showed a limit of 775 for a mailbox set to 25. A mailbox carrying the bounces is a deliverability problem that looks like a copy problem.
4. **See who responds.** Replies by seniority or title. Filter to the campaigns in question, or one-off emails get counted (see below).
5. **Count outcomes.** Meetings booked and opportunities created, grouped by month or by user. One platform could not group opportunities by campaign at all.
6. **Cross-check before anyone sees a number.** Take the busiest campaign and compare the reporting totals with the counts shown on the campaign itself. They should match. On one real account they matched exactly: 2 sent, 2 delivered, 2 opened, 1 replied. If they do not, say so in the report rather than picking one.
7. **Read reply quality.** Count replies by category if the platform classifies them (willing to meet, follow-up question, referral to someone else, not interested, unsubscribe). On one account the largest group had no category at all. Use it to sort, then judge which replies are positive.

**Traps, all hit on a real account.**
- **An incompatible metric can wipe the whole report.** Asking for opportunities grouped by campaign returned a warning and no table at all, not a table with one column missing. Request outcome metrics in a separate call.
- **One-off emails inflate replies when you group by time or by person.** Grouped by month, one account showed 30 sent and 19 replied, because replies to individual emails sent outside any campaign were counted. Grouped by campaign they dropped out. Filter to campaigns whenever the question is about campaigns, or you will eventually report a reply rate above 100%.
- **A summary bounce count may not be the real bounce count.** On one archived campaign, reporting showed 40 sent, 34 delivered, and 6 bounced, while the contacts in the campaign showed 0 bounced. The six was simply sent minus delivered. Take bounces from the contact-level statuses, and say which source you used.
- **Report data may come back as text, not structured data.** One platform returned its report as a table inside a text field. Parse the rows; do not assume a data array.
- **Opens are not a signal.** Cold campaigns should run with open tracking off, so open counts are empty or meaningless. Do not report them as a result.
- **Small numbers are not results.** A variant that won one reply to zero on one send each has told you nothing. Put the send count next to every rate, and do not call a winner on a handful of sends.
- **Replies are not positive replies.** The reply rate includes unsubscribes and "not interested". Never present it as the outcome.
- **A quiet report is not always a quiet campaign.** If a live campaign has contacts enrolled and nothing scheduled, that is an account problem, often emails waiting for approval, not a performance result. Flag it separately.

**What to hand me at the end:**

A table with one row per campaign: campaign, sent, bounce %, reply %, positive reply %, meetings this month, and your call (scale, fix, or stop). Put the send count beside every rate.

Then, for each live campaign: the step where replies stop, the variant ahead with its send count, the mailbox bouncing most, and one recommended action.

End with anything that did not reconcile in the cross-check, and which source you used for bounces.

Method adapted from the campaign reporting guide in Apollo Operator, a free, open-source headless GTM toolkit by Creatop: github.com/creatop-gtm/apollo-operator


For the full operating context, read [Operator Context](https://raw.githubusercontent.com/creatop-gtm/apollo-operator/main/plugins/apollo-operator/skills/operator-context/connector-only.md).
