---
name: apollo-go-live
description: "The last mile: enroll a graded list into a built (inactive) sequence, get a human to approve activation, and pull contacts on complaint or bad fit. Use to take an Apollo sequence live safely."
---

# Apollo Go-Live (Launch)

Turn a built sequence and a graded list into a live campaign, safely. This is the bridge everyone skips: the List phase gives you a clean list, the Message phase builds the sequence as an inactive draft, and then nothing happens until someone connects the two and flips the switch. This skill is that step, and it is the one place where a wrong move sends real email to real people, so it is deliberate and human-gated.

## When to use

- A sequence exists in Apollo as an inactive draft (Message done).
- A graded, verified list exists as contacts in the account, ideally under one Apollo List label (List done).
- Deliverability preflight passes (Infrastructure): mailboxes warmed, schedule set, sending from a dedicated domain.

If any of those three is missing, go back and finish it. Do not improvise the last mile.

## The rule that does not move

**The machine never flips the switch.** Claude enrolls, summarizes, and recommends. A human runs the activation after confirming the sender, the schedule, and the copy. Every tool in this flow sends or commits real email, so every one of them waits for an explicit human yes.

## Process

### 1. Preflight (Infrastructure, non-negotiable)
Run `apollo-deliverability` first. Confirm warmup is complete (14 days minimum, 21 optimal), bounce risk is low (100% verified), the sending schedule is business-hours/weekdays/timezone-correct (`apollo_emailer_schedules_index`), and you are sending from a dedicated domain, not the primary. If deliverability is not clean, stop here. You cannot out-send a reputation problem. If the sending stack does not exist yet, that is a `sending-infrastructure` job, not a go-live one.

### 2. Pick the sender mailbox
Call `apollo_email_accounts_index`. Auto-select the mailbox where `default: true`, unless the operator names another. Confirm it is a dedicated sending domain that redirects to the main site, never the company's primary domain. For volume, you can pass several mailbox IDs to rotate.

### 3. Enroll the list (nothing sends yet)
Enroll into the sequence while it is still inactive, so contacts queue but no email goes out until activation. Use `apollo_emailer_campaigns_add_contact_ids`:
- `id` and `emailer_campaign_id`: both the sequence id.
- `send_email_from_email_account_id`: the mailbox from step 2 (string, or an array to rotate).
- **Enroll by `contact_ids`.** Collect the IDs of your graded list with `apollo_contacts_search` (search the list, take each contact's `id`), then pass them as `contact_ids`. The tool advertises a `label_names` shortcut to enroll a whole Apollo List by name, but in live testing it errors ("Required parameter 'contact_ids' missing"), so do not rely on it. Resolve the label to contact IDs yourself and pass `contact_ids`.
- **Leave `sequence_unverified_email: false`** (the default). This makes Apollo refuse unverified addresses, which enforces the verify-everything rule at the door. Do not flip it to true to "get more in."
- Only contacts can be enrolled. If your people are search results but not yet contacts, enrich and create them first (List), then enroll.
- **Verify enrollment.** Each enrolled contact comes back with the sequence id in its `emailer_campaign_ids`. A removal (step 6) sets it back to empty. Confirm rather than trusting the call.

### 4. Human review (surface, then wait)
Before activation, show the operator a plain summary and stop:
> Sender: `outbound@dedicated-domain.com` (dedicated, warmed 21 days). Sequence: "Q3 Trade Show Managers". Contacts enrolled: 312 (all verified). Schedule: business hours, weekdays, contact timezone. Ready to activate?

### 5. Activate (human only)
On an explicit yes, activate with `apollo_emailer_campaigns_approve` (sequence id). This flips `active: false` to `true` and starts sending on the schedule. It is irreversible for any email already dispatched. Claude does not run this on its own initiative, and never on "just do it" without the step 4 summary shown first.

### 6. After it is live: pulling contacts
Apollo already auto-handles the normal cases: every sequence built here defaults to finishing a contact on reply or interest and pausing on out-of-office. So you do not manually pull someone just because they replied. Use `apollo_emailer_campaigns_remove_or_stop_contact_ids` for the explicit cases: a complaint, a removal request, a wrong-person or do-not-contact, or a list you need to yank. Pass `contact_ids`, `emailer_campaign_ids`, and `mode: "stop"` (with a `stop_reason`) to halt them where they are, or `mode: "remove"` to take them out entirely.

### 7. The kill switch: stopping the whole send

Pulling contacts handles individuals. When the problem is the campaign itself, a bounce spike, wrong copy that got through review, or the wrong list enrolled, you need to stop everything at once.

**There is no MCP tool for this**, which is a real hole if your whole motion runs on MCP. Every other lane can do it: the Apollo UI, the CLI, and REST (`POST /emailer_campaigns/{sequence_id}/abort`).

```bash
apollo sequences abort --id <sequence_id>      # deactivate, stops sending
apollo sequences archive --id <sequence_id>    # retire a sequence you are done with
```

**Install the CLI before you activate anything**, not during an incident, because an incident is exactly when you do not want to be running a Homebrew install and an OAuth flow. Setup is in `apollo-operator`.

Abort first, diagnose second. A paused campaign costs you a day; a bounce spike left running costs you the domain. Then work the incident with `apollo-deliverability`.

## Common mistakes

- **Enrolling into an already-active sequence.** Then it sends immediately, no review gate. Enroll into the inactive draft, review, then approve.
- **Flipping `sequence_unverified_email` to true.** That defeats verification and feeds bounces straight into your domain reputation.
- **Going live before warmup finishes** to save a week. You lose the domain instead of saving the week.
- **Sending from the primary domain.** One reputation, and it is the company's real one. Always a dedicated sender.
- **Treating a reply as something you must manually remove.** Apollo finishes them on reply by default. Save the remove/stop tool for complaints and bad-fit pulls.
- **Activating without a kill switch installed.** `sequences abort` exists on every lane except MCP. Get the CLI working before go-live, not in the middle of an incident.
- **Diagnosing before aborting.** Stop the send first. The campaign is still there to investigate once it is no longer making the problem worse.
