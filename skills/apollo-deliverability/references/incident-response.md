# Deliverability incident response

> When something breaks, triage in this order. The goal is always the same: stop the bleeding first, diagnose second, resume only when it is safe.

## Bounce rate spiking

1. **At 4% or above, pause the campaign immediately.** Do not wait to understand why. Stop the sending, then investigate.
2. **Check the list.** A bounce spike is almost always a verification failure: unverified or stale emails got in. Re-verify, drop the bad ones.
3. **Check the mailboxes** with `apollo_email_accounts_index`. If specific mailboxes are bouncing, isolate and rest them.
4. Resume only after the list is clean and bounces are back under 2%.

## Replies fell off a cliff (but bounces are fine)

1. This is usually inbox placement, not copy. Your emails may be landing in spam.
2. Check spam placement (Apollo's inbox-placement tooling, UI, see the KB map). 
3. Look at recent changes: a new copy variant with spam words, a volume jump, a new unwarmed mailbox.
4. Reduce volume, swap in backup mailboxes, fix the trigger, ramp back up slowly.

## A mailbox or domain got blacklisted

1. Pull that mailbox out of rotation now. Swap in a backup (this is why you keep them).
2. Stop cold sending from the affected domain. Let it rest and keep only warmup running.
3. Check the blacklist reason. Usually volume too fast, or a bad list. Fix the cause before reusing it.

## Warmup looks stuck or blocked

1. Confirm warmup is actually enabled on the mailbox.
2. Do not launch cold from it until warmup has run the minimum. A stuck warmup is a reason to delay, not to push.
3. If a mailbox will not warm, retire it and promote a backup.

## The principle under all of it

Stop first, diagnose second. The cost of pausing a campaign for a day is nothing. The cost of burning a domain is weeks and money. When in doubt, pause.
