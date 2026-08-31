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

## Read the schedule before you activate, because it decides when anything happens

`apollo sequences schedules` returns more than the id you need for `--schedule-id`. The useful part is `schedule_hash`, and it is worth reading before every launch:

```json
{"id":"…","name":"Normal Business Hours","time_zone":"America/Los_Angeles",
 "use_contacts_time_zone":true,"skip_holidays":true,
 "schedule_hash":{"monday":`8,17`,"tuesday":`8,17`,"wednesday":`8,17`,
                  "thursday":`8,17`,"friday":`8,17`}}
```

Three fields change what activation actually does:

- **`schedule_hash`** gives the real send window per weekday as `[start_hour, end_hour]`. Check it against the sending rules you are working to. Apollo's stock "Normal Business Hours" is **8 to 17**, which is not the same as a 9 to 18 policy, and nobody notices the hour of drift until they look.
- **`use_contacts_time_zone: true`** means the window is evaluated in **each recipient's** timezone, not the schedule's. This is what you want for a US list, and it means activating at one moment produces sends spread across hours. A US-wide list activated at 09:00 Eastern sends to the East Coast immediately and holds the West Coast for another two hours.
- **`skip_holidays`** silently defers sends on holidays, which will look like a broken campaign if you are watching the queue that day.

**So an empty message queue right after activation is usually correct, not broken.** Before diagnosing anything, check the local time for the contacts in the list against the window. Only treat it as a fault once a contact is inside their own window and still has nothing queued.

## Enrollment fails silently. Check the response body, never the exit code.

**Verified 2026-08-31, and it cost a false "done" in a real session.**

`apollo sequences add-contacts --contact-id <ids...>` is documented as variadic, but the CLI **joins the ids into a single string**. Apollo then looks up one contact whose id is `"id1 id2 id3 id4 id5"`, finds nothing, and returns **HTTP 200 with exit code 0**:

```json
{"contacts": [],
 "skipped_contact_ids": {"6a95… 6a95… 6a95… 6a95… 6a95…": "contact_not_found"}}
```

Nobody is enrolled, nothing warns you, and the shell says success. The failure only shows up when a human opens the sequence and sees it empty.

**Two rules, both mandatory:**

- **Assert on the response body after every enrollment.** `contacts` must have the length you expect and `skipped_contact_ids` must be empty. Treat a non-empty `skipped_contact_ids` as a failure even on a 200. The skip reasons are informative (`contact_not_found`, and others for suppression and ownership rules), so print them.
- **Enroll more than one contact over REST, not the CLI.** `POST /emailer_campaigns/{id}/add_contact_ids` with a proper `contact_ids` array works correctly. Single-contact CLI enrollment is fine because there is nothing to join.

**Enrolling does not send while the sequence is inactive.** Enroll first, verify the count, then activate as a separate deliberate step. That ordering gives you a real checkpoint between "the right people are in" and "email is going out."

## Common mistakes

- **Enrolling into an already-active sequence.** Then it sends immediately, no review gate. Enroll into the inactive draft, review, then approve.
- **Flipping `sequence_unverified_email` to true.** That defeats verification and feeds bounces straight into your domain reputation.
- **Going live before warmup finishes** to save a week. You lose the domain instead of saving the week.
- **Sending from the primary domain.** One reputation, and it is the company's real one. Always a dedicated sender.
- **Treating a reply as something you must manually remove.** Apollo finishes them on reply by default. Save the remove/stop tool for complaints and bad-fit pulls.
- **Activating without a kill switch installed.** `sequences abort` exists on every lane except MCP. Get the CLI working before go-live, not in the middle of an incident.
- **Diagnosing before aborting.** Stop the send first. The campaign is still there to investigate once it is no longer making the problem worse.
