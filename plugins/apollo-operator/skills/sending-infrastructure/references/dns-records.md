# DNS records for a sending domain

The depth behind Step 3 of `sending-infrastructure`. Every dedicated sending domain needs these records set at its DNS host (the registrar, or wherever the domain's nameservers point). Get all of SPF, DKIM, and DMARC passing before the first cold send. This is the difference between the inbox and the spam folder, and no amount of good copy makes up for getting it wrong.

If you bought provisioned domains and mailboxes, the provider usually sets these for you: your job is then to *verify* them (see the checklist at the bottom), not to author them.

## The four records

### MX (mail exchange)
Routes inbound mail (replies, and the warmup pool's messages) to your mailbox provider. Without a correct MX record, replies to your cold emails go nowhere, and you will not even know people answered. Your mailbox provider gives you the exact MX host and priority to set. This is table stakes: no MX, no two-way email.

### SPF (Sender Policy Framework)
A TXT record listing which servers are allowed to send email as this domain. When a receiver gets your email, it checks the sending server against this list. If the server is not authorized, the mail looks forged.

- One SPF record per domain, and only one. Multiple SPF records is itself a failure.
- It must `include:` the sending service you actually use (your mailbox provider or sending platform gives you the exact include).
- End it with `~all` (softfail) or `-all` (hardfail). Do not use `+all`, which authorizes the whole internet to send as you.

### DKIM (DomainKeys Identified Mail)
A cryptographic signature. Your provider publishes a public key as a DNS record; it signs every outbound message with the matching private key. Receivers verify the signature, which proves the mail genuinely came from your domain and was not altered in transit.

- The provider gives you the exact record (a TXT or CNAME at a selector host like `selector._domainkey.getbrand.com`). Paste it exactly. A single wrong character breaks the signature.
- Verify it is live and passing after you set it, propagation can take a few hours.

### DMARC (Domain-based Message Authentication)
A TXT record at `_dmarc.getbrand.com` that tells receivers what to do with mail claiming to be from your domain that fails SPF or DKIM, and where to send reports. This is where domain reputation is actually built.

- Start with a monitoring policy: `v=DMARC1; p=none; rua=mailto:you@brand.com`. `p=none` means "do not block anything yet, just report," so you can watch for problems without hurting delivery.
- Once SPF and DKIM are clean and you have watched the reports, you can tighten to `p=quarantine` and later `p=reject`. Never start at `reject`, one misconfiguration and all your mail disappears.
- A missing DMARC record increasingly gets mail sent straight to spam by major receivers. It is no longer optional.

## The redirect (not DNS, but part of setup)

Point the sending domain itself (a 301 redirect at the web/hosting layer) to the main website, so a prospect who types or clicks `getbrand.com` lands on the real brand. A sending domain that resolves to nothing looks abandoned and hurts trust. This is a redirect rule at your host or a redirect service, separate from the DNS records above.

## Verification checklist (before any cold send)

For every sending domain, confirm:

- [ ] MX record present and pointing at the mailbox provider.
- [ ] Exactly one SPF record, includes the real sending service, ends in `~all` or `-all`.
- [ ] DKIM record live at the provider's selector, signature passing.
- [ ] DMARC record at `_dmarc.<domain>`, at least `p=none` with a reporting address.
- [ ] Domain 301-redirects to the main website.
- [ ] For Apollo-provisioned domains, `apollo_domain_purchase_index` shows SPF, DKIM, and DMARC passing in its diagnostics.

Free public tools (MXToolbox and similar) will check SPF, DKIM, DMARC, and MX for any domain from the outside. Use one to confirm each record is live and passing before you trust it. Do not rely on "I set the record" as proof, confirm it resolves.

## Common failures

- **Two SPF records.** Only one is allowed; a second one makes SPF fail entirely. Merge includes into a single record.
- **DKIM pasted with a line break or a missing character.** The signature silently fails. Copy it exactly, verify externally.
- **Jumping straight to `p=reject` on DMARC.** One SPF or DKIM gap and every message vanishes. Start at `p=none`, tighten later.
- **Forgetting MX** because outbound seems to work. Outbound sends, but every reply is lost, which quietly kills the whole point of the campaign.
- **No redirect**, so the sending domain resolves to an error page. It reads as fake and costs you trust and clicks.
