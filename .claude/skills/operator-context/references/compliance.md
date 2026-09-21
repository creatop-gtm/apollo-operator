# Compliance: operating rules for outbound on Apollo

> **Not legal advice.** Outreach law differs by country, changes, and depends on your business. These are the operating habits this library runs on so that opt-outs, privacy settings, and call registries are handled by default rather than remembered. Agree the legal basis for your own motion with whoever owns legal for the business.

Where this reference cites Apollo's own behaviour, the source is Apollo's help center (article names given) or a live check on a real account, dated.

---

## 1. Every cold email carries an opt-out, and every opt-out is honoured

- **Sequence emails:** Apollo attaches an unsubscribe link to emails sent from sequences, and a recipient who unsubscribes is removed from future steps in every sequence they are in (Apollo help center, "Configure Your Email Unsubscribe Link"). **Check the setting is on in the workspace before the first send.** No API field exposing it was found on 2026-09-14, so this is a UI check.
- **One-off emails** (sent outside a sequence): do not assume the link is attached. Write an opt-out line into the email yourself.
- **A real physical address** belongs in every cold email, usually in the signature.
- **A plain-language opt-out line helps, and does not replace the link.** "Reply 'no thanks' and I'll close the loop" is good copy; it only works if someone actually reads and actions the reply.

## 2. Opt-outs outlive the tool

Apollo's own unsubscribe handling protects Apollo sequences. It does not follow a person into a rebuilt list, a re-enrichment, or another sending platform. So:

- **Add every unsubscribe, "not interested", and complaint to the suppression file** the moment it happens, and subtract that file from every new list before enriching (see `apollo-list-builder`, step 3).
- **Never re-engage them**, however much time passes (see `re-engagement`).

Where to read them, all verified 2026-09-14:

| Source | What it gives |
|---|---|
| `apollo_analytics_sync_report`, metric `num_emails_unsubscribed` | Unsubscribe counts by sequence, step, or period |
| `apollo_emailer_messages_search` in `metadata_mode` | The Unsubscribed count for one sequence |
| `apollo emails search --stats replied` (CLI) | Replies Apollo classed as `unsubscribe` |
| `opt_out_rate` on each sequence object | The sequence's opt-out rate |

## 3. EU contacts: decide before you prospect, not after

- **Apollo has a workspace-wide GDPR setting** that removes EU-located people from prospecting, emailing, and email tracking for every user (Apollo help center, "Configure GDPR Settings on Apollo", under Settings, Rules of engagement, Prospecting config).
- **Read its state before building a list.** Contact search responses carry `disable_eu_prospecting` (`false` on the account checked 2026-09-14). If it is off and your ICP includes EU countries, that is a decision someone should make on purpose.
- **If you do prospect EU people:** have a documented lawful basis agreed with whoever owns legal, keep the message relevant to the person's role, include the opt-out, and honour removal requests promptly. Apollo keeps a Removal Requests list for requests it receives.
- **Person-level website visitor identification is US only** in Apollo, so first-party intent angles rarely reach EU people anyway (see `apollo-signals`).

## 4. Calls: screen before anyone dials

- **Apollo's DNC screening** checks phone numbers against do-not-call registries in the US, Germany, France, Canada, Australia, and the UK (TPS and Corporate TPS). Flagged numbers show "Do Not Call" and are blocked from Apollo's dialer, including numbers imported from a CRM; a user can override per contact where they have consent (Apollo help center, "Enable Do Not Call (DNC) Phone Screening"). It has to be enabled.
- **Reading the status:** phone numbers returned by a reveal carry `dnc_status_cd`. Only `found` means the number is on a registry. `pending` means screening is still running, **not** that the number is clear. Saved contacts' phone numbers carry `dnc_country_supported`, which says whether the number's country can be screened at all.
- **Numbers dialed outside Apollo are not blocked.** If a call task is worked from a personal phone or another dialer, check the status first (see `apollo-multichannel`).
- Since 2026 Apollo also exposes DNC status in CRM field mapping and CSV exports, and a call restrictions filter in search, per its release notes.

## 5. Recorded calls need consent where the law requires it

Some jurisdictions require every party's consent to record a call. Apollo added recording-rule guardrails for two-party consent jurisdictions in 2026, per its release notes. If calls are recorded, follow those rules, and treat the recordings and transcripts as confidential source material (see `business-brief`).

## 6. Business contacts only

Do not email consumers. Cold outreach in this library targets people in their professional role at their business address. Contacts carry a `free_domain` flag; a free-mail address on a B2B list is a list problem to remove, not a person to email.

## 7. Prospect data stays private

Lists, enriched records, replies, and call recordings belong to the business running the motion. Never paste them into anything published or shared with a third party, and describe patterns rather than people when discussing results.

---

## What Apollo does not do for you

- Carry opt-outs into other tools or future lists. The suppression file does.
- Decide whether you have a lawful basis to contact someone.
- Block numbers dialed outside its own dialer.
- Show, through the API, whether the unsubscribe link setting is on.
