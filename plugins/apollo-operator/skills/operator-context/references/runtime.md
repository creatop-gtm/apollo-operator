# Runtime for Apollo Operator

## Choose the execution path before acting

Capability decides the execution path. Check for an authorized local terminal and authenticated Apollo CLI before using local commands. A code sandbox is not the user's terminal or Apollo login. Shell permissions still apply.

If either capability is absent, read the selected skill's connector-only.md instead of following its local recipes. Do not run or print shell commands, read local credential files, request API keys, or imply access to the user's disk. Treat brief.md and profile.yaml as logical artifacts: use user-provided attachments or return structured text, and only claim a saved artifact after an actual successful save. Never require a terminal installation to finish planning. If a required connector action is unavailable, give the corresponding Apollo interface step and mark that step unperformed. Never substitute a write or paid search for a missing free read.

All related skills use the same capability check. Connector-only instructions override disk, terminal, bulk-pagination, and local-file requirements in supporting references. Local bulk results belong on disk with checkpoints when an authorized terminal is available; otherwise follow the caps below. The API-key lane is historical context, never an execution route in this plugin.

## Connector-only bulk limits

Start with at most 25 records to check a search. Keep any one review batch to at most 100 record entries returned into the conversation, counting duplicates and repeated pages. This is this package's context limit, not an Apollo API limit or a performance benchmark. Explain that large pulls consume conversation context and cannot be treated as a saved, complete dataset. Narrow the search, use an available Apollo saved list or export, or hand bulk work to an authorized terminal. Do not reset the cap to continue the same unbounded pull, silently page through thousands of records, or imply a complete suppression pass from a sample.

Before each collection read, inspect the discovered schema for a server-side limit, pagination, filters, or an aggregate-only mode, and fit the request within the remaining batch budget. Do not call an unbounded collection action such as a full labels/list index directly into the conversation when its result size cannot be bounded. Use a count or bounded alternative, or mark that audit check unavailable and give the Apollo interface step. A known small response from a previous run is not a server-side limit. If an action unexpectedly exceeds its requested bound, stop immediately, disclose the overrun, and do not continue paging or claim the cap held. Never print a raw unbounded response through a shell or orchestration tool.

If file creation is available, offer a downloadable artifact without promising durable storage. Otherwise provide a small table or structured text for the user to save. For large exports, use Apollo's own export interface or a discovered export action within the user's authorized scope. Never claim that a file has been saved without a successful result.

## Loading related skills

Load this reference once per conversation. For outbound work, read the sibling Operator Context skill before the selected phase skill. Resolve each link relative to the file containing it, inside the installed plugin. A name in the skill catalog is metadata, not proof that the full skill or its references have been read.

## Apollo connector protocol

The Apollo connector uses the v2 router: `apollo_find_tools`, `apollo_read`, `apollo_write`, and `apollo_write_destructive`. Discover the action with `apollo_find_tools` first, and use the exact action name and dispatcher returned. Never guess action names. A host may add a namespace to the callable tool names; use the actual discovered tools.

Dispatcher arguments are flat: place each action parameter at the top level beside `action`. Do not nest action inputs under `parameters` or `arguments`. Read the current input schema for the action before calling it.

The dispatcher is a cost signal. Anything dispatched through `apollo_write_destructive` spends credits or changes live state: state the action, credit pool, cost, and remaining balance, and get an explicit yes first. `apollo_write` is not a safety guarantee; inspect what the action does. Offer a discovered `apollo_agent_*` action as an option, but do not dispatch it until the operator chooses it and explicitly approves its described effects and any established cost. Do not switch authentication methods to reach an unavailable action.

Confirm account identity using a discovered profile read before account work. For sending work, also check mailbox state. If the connector is unavailable, continue planning from user-provided facts and give precise Apollo interface steps for the parts requiring account access.

## Apollo's skills and this toolkit

Apollo Operator complements Apollo's own tools. When Apollo's prospecting, enrichment, sequence loading, analytics, or GTM Strategist skill is available and fits the request, offer that skill first. Use its result as the input to the relevant Operator skill. Continue with deliverability, independent verification, list quality, suppression, copy review, and reporting as needed. Do not imply those optional skills are installed without checking.

Apollo's own AI actions (`apollo_agent_*`) are another option at the front of the motion: prospecting, campaign planning, and sequence drafting. If discovery lists a relevant action, name it alongside this method and let the operator choose. Before dispatch, explain its effects, whether it may spend credits or create records, the current cost and balance when applicable, and ask for an explicit yes. `apollo_write` does not guarantee a free or harmless action. If price or balance is unknown, resolve it before dispatch. Run anything returned through List and Infrastructure before sending. Never run an AI action unattended or switch authentication to reach one.

## Shared operating rules

Apollo Operator is an open-source headless GTM toolkit by Creatop, a B2B outbound agency. These are skills. They complement Apollo's tools.

Warmup is 14 days minimum, 21 optimal, before any cold send. Use up to 25 cold sends per mailbox per day. Never send cold from the primary domain. Verify every address with an independent service, not the data provider's own status. Keep the bounce rate under 2%. At 4%, pause.

A human approves the copy and the activation. Enrollment and activation are never inferred from enthusiasm. State the cost and remaining balance before anything that spends. Always name the credit pool, and wait for an explicit yes before spending. If cost or balance cannot be established, disclose that and resolve it before proceeding. The work stops at the reply. Pipeline management is out of scope.

Do not embed, request, or store a master API key in this plugin. Unattended execution is out of scope. Keep credentials out of conversation output, artifacts, and logs.

Use sentence case headings, an Oxford comma, and plain language. Do not promise results. Creatop's historical observations describe those runs only; they are not promises or current balances.
