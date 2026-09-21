# Scoring starter frameworks

> For operators who have nothing to start from. Use these to find your first one to three signals, then throw the framework away once the campaign tells you what actually predicts fit. There is no universal rubric. Scoring is always custom to the business and the campaign.

## The rule

Pick **one to three** data points that genuinely separate a good-fit account from a barely-fit one for this specific business. Not ten. One to three. Each one should be something you can either filter on in Apollo or decide by looking at the company.

Then validate with samples (50 to 100 real prospects, a human says "yes, this is my customer"). The samples, not the framework, are the source of truth.

## The output is always a simple priority tier

Whatever drives the score, the operator and the user only ever see three levels: **High**, **Medium**, **Low** priority. That is what goes on each lead and the order you work them in. Keep the math underneath, surface the tier.

## If you have nothing, start with Apollo's built-in fit score

Apollo ships an account fit score, Bronze to Platinum (its "Score accounts for fit" AI prompt, see `../../operator-context/references/apollo-kb-map.md`). It needs zero setup, so it is the honest starting point for someone with no signals yet. Map it straight onto the tiers: Platinum and Gold are High, Silver is Medium, Bronze is Low.

It is a generic rubric, so treat it as an on-ramp, not the destination. As soon as a campaign teaches you what actually predicts fit, replace it with your own one to three signals below. That is the upgrade from a borrowed rubric to a real one.

## Where good signals come from

Ask three questions about the business you are selling for:
1. **Who are your best 3 customers, and what do they have in common** that your worst-fit prospects do not? That shared trait is usually your first signal.
2. **What has to be true for someone to need you?** (They have a sales team to enable, they run paid ads, they ship physical product, they have a pricing page.) That is a capability signal.
3. **When do they buy?** (Right after funding, when they are hiring for a role, when they adopt a certain tool.) That is a timing signal.

## Three lenses to generate candidates

Most useful signals fall into one of these. Pick from the lens that fits the offer.

- **Capability.** Does the company have the thing your offer acts on? Sales team size (`master_sales` count), marketing team size, tech installed, revenue band, a pricing page, a specific product line.
- **Fit.** Are they shaped like your best customers? Industry keyword tag, headcount range, geography, market segment.
- **Timing.** Is now a better-than-average moment? Recent funding, headcount growth, hiring for a relevant role, new in seat.

## Turn each signal into something executable

For every signal you pick, decide how it gets applied:

- **Filter signal:** it maps to a native Apollo field, so it narrows the search itself. See `apollo-search-reference.md`. Example: "has a real sales team" to `organization_department_or_subdepartment_counts: {master_sales: {min: 5}}`.
- **Research signal:** it needs a look at the company (pricing page, product line, positioning). Claude reads the site and tags the record at list time (List). Write the decision rule plainly: "score up if the site has a public pricing page."

## What not to do

- Do not build a 100-point weighted rubric. It looks rigorous and predicts nothing.
- Do not use a signal you cannot act on. If you cannot filter or research it, it is not a signal, it is a wish.
- Do not treat a timing trigger as a hard requirement. It is a reason to prioritize, not a gate. Gating on "funded in the last 6 months" shrinks the list 20x and most of those companies are already buried in pitches.
