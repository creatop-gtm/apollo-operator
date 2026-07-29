# Angles: running more than one against a single ICP

An **ICP** is who to reach. An **angle** is *why they are relevant right now*. One ICP usually supports several angles, and picking between them is one of the highest-leverage decisions in the whole motion. It is also nearly free, because people search costs nothing.

---

## The three kinds, and why the difference matters

The kind of angle determines how long a list stays good, which determines how you should spend on it.

### Timing signal

Something just happened. They are hiring an SDR, they raised a round, someone changed jobs, they adopted a tool.

- **Sharpest relevance available.** The reason to write is obvious and specific.
- **Decays fast.** Once they hire, the moment is gone. A hiring-signal list is stale in weeks.
- **So: rebuild it, do not stockpile it.** Enrich close to send, and re-run the search rather than reusing a saved list.

### Technographic fit

They already use a tool your offer wraps around, replaces, or complements.

- **Self-qualifying.** Owning the product proves budget, intent, and category awareness at once.
- **Decays slowly.** Companies do not swap core tooling often.
- **Usually small.** A technographic universe is a fraction of the firmographic one. Treat it as a precision list, not a volume list.
- **Strongest when you have a credible claim on that tool** (a partnership, a certification, deep expertise). Then it is a moat, not just a filter.

### Structural fact

Something permanent about how the company is built. 11 to 200 employees with 0 to 2 people in sales. No marketing team. One location. Founder-led.

- **Does not decay.** True until the company changes shape, which is slow.
- **Usually the largest universe of the three**, so it is the volume workhorse.
- **Weakest single reason to write**, because nothing just happened. It needs the copy to do more work, and it pairs well with a vertical to stay specific.

| | Relevance | Decay | Typical size | Use as |
|---|---|---|---|---|
| Timing signal | Highest | Fast | Small to medium | Sharp, rebuilt often |
| Technographic | High | Slow | Small | Precision play |
| Structural | Moderate | None | Large | Volume workhorse |

---

## Size several before committing to one

Because people search is free, size every candidate before spending anything. Use `per_page: 1` and read `total_entries`.

**Always compute the unfiltered baseline too.** It shows what choosing an angle actually buys you. In a real session, one ICP produced:

| Angle | Universe |
|---|---|
| No angle (baseline) | 203,908 |
| Structural: 0-2 salespeople | 127,751 |
| Structural + vertical: B2B SaaS, 0-2 salespeople | 2,837 |
| Timing: hiring an SDR or BDR | 3,243 |
| Technographic: already uses the tool | 1,044 |

Seven angles sized in a few minutes, zero credits, and the decision made on numbers instead of instinct.

**Expect roughly 85% attrition** from raw universe to sendable, after drift cuts, deduping, verified-only filtering, and catch-all removal. Two real runs landed at 15% and 13%. So a 1,000-person angle is a 150-person campaign, and a 3,000-person angle is a first month of sending. Say that out loud before anyone gets attached to the big number.

---

## One angle per campaign

Two angles in one campaign means one message trying to be relevant for two different reasons, and a result you cannot attribute to either. Separate campaigns cost nothing extra and tell you which angle actually works.

Copy has to change with the angle, not just the list:

- **Timing:** lead with the event. "Saw you're hiring an SDR."
- **Technographic:** lead with the tool. "You're already running X."
- **Structural:** lead with the consequence. "Most 40-person companies without a sales team end up with the founder doing outreach at 9pm."

Same offer, three genuinely different first lines. If the same email would work for all three angles, the angles were not different enough to be worth separating.

---

## Suppress between angles, every time

Angles against one ICP **overlap**, and the overlap is invisible unless you check. In a real three-angle run: **85 people in the second list had already been enriched for the first**, and **19 more appeared in both new lists.** Every one would have been paid for twice and emailed twice.

- Keep one suppression file per business of every person id already enriched or enrolled.
- Subtract it from each new list **before** enriching, since that is where the money is.
- When two angles both want the same person, assign them to the **higher-intent** angle and remove them from the other, deliberately.

```bash
comm -12 new_ids.txt already_enriched.txt | wc -l    # what the overlap would have cost
```

Full mechanics in `apollo-list-builder`, step 3.

---

## Sequencing several angles

Do not launch four campaigns in one week. With one live campaign a result is attributable; with four it is noise, and a deliverability problem in one is hard to trace.

1. **Start with the sharpest**, usually a timing signal or a technographic fit. It is the fastest read on whether the offer lands at all.
2. **Add the volume angle** once the first has produced replies worth learning from.
3. **Keep the rest staged.** A free-stage list costs nothing to hold and everything to enrich early.

If the sharpest angle fails, that is information about the **offer**, not the angle. Widening the list will not fix it, and a bigger list of the wrong people is just a more expensive version of the same result.
