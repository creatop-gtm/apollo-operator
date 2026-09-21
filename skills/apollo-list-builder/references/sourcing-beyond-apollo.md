# Sourcing beyond Apollo

> This library automates sourcing in Apollo, because that is what the MCP can do. But Apollo is not the only place good lists come from. These sources are manual and out of scope for the skills, listed here so you know when the manual effort is worth it.

| Source | Best for | Why go here instead of Apollo |
|---|---|---|
| Sales Navigator | People-first search, fresh titles, boolean depth | Sometimes fresher on job changes and title nuance. Export, then enrich. |
| Google Maps | Local SMBs (restaurants, clinics, contractors, trades) | Apollo is thin on small local businesses. Maps has them with address and phone. |
| Trade-show / conference exhibitor lists | Vertical events where your buyer literally shows up | The strongest "signal" list there is: they paid to be in the room. Ideal when you sell to that event's audience. |
| Industry and association directories | Narrow verticals Apollo tags poorly | A niche directory can beat a broad database on fit. |
| Government / public databases | Licensed professionals, registered entities, permits | Authoritative, and often the only complete source for a regulated segment. |

## How to think about it

- **Apollo first.** It is the automated path and covers most B2B software and services targets well.
- **Go manual when Apollo is thin or wrong** for the segment: very small local businesses, a specific event's attendees, a licensed profession, a niche vertical.
- **Whatever the source, the rest of the pipeline is the same:** dedupe, enrich, verify, apply your scoring signals, and grade with `list-quality-scorecard` before anything gets sent. A list from a directory still has to pass the same bar as a list from Apollo.

A trade-show exhibitor list is worth calling out: attendance is a genuine buying signal when you sell to that audience, which is exactly the "a signal is only as good as its offer fit" idea from `outbound-principles.md`.
