# Claude Code Build Brief — Property Scorer Skill
# Harcourts Platinum Real Estate | LAItely AI Operations

**Date:** May 2026
**Prepared by:** Lisa Myburgh, LAItely
**Session goal:** Produce a production-ready `.skill` file for the Harcourts Property Investment Scorer, updated to consume live benchmark data from the Rental Research Agent instead of hardcoded rent figures.

---

## What a .skill file is

A `.skill` file is a ZIP archive containing a single folder with a `SKILL.md` file inside. The folder name matches the skill name. Structure:

```
harcourts-property-scorer.skill  (the ZIP)
└── harcourts-property-scorer/
    └── SKILL.md
```

To create it in bash:
```bash
mkdir -p harcourts-property-scorer
# write SKILL.md into the folder
zip -r harcourts-property-scorer.skill harcourts-property-scorer/
```

Validate by unzipping and confirming the SKILL.md is readable and well-formed markdown.

---

## What needs to change from the previous version

The existing skill (`harcourts-property-scorer-SKILL-v2.md`) has corrected rental benchmarks hardcoded into Step 2A as a static table. This works for a single point in time but goes stale within months as the rental market moves.

The change: **remove the hardcoded benchmark table entirely** and replace it with instructions to read from `{{benchmark_json}}`, a variable injected by the Rental Research Agent that runs immediately before the scorer in the Make.com pipeline.

Everything else in the skill stays the same: hard filters, scoring categories, tier logic, output JSON format, handling of incomplete data, and market context notes.

---

## The updated SKILL.md — full content to write

Write the following as the complete contents of `harcourts-property-scorer/SKILL.md`:

---

```markdown
---
name: harcourts-property-scorer
description: >
  Score a Harcourts Platinum property listing for investment suitability and determine
  whether it qualifies for the Everlytic investor subscriber list. Use this skill whenever
  a new property listing needs to be evaluated, scored, or triaged for investment potential,
  especially in automated pipelines receiving PropData listings. Always use this skill
  when asked to "assess a listing", "score a property", "check if this qualifies for
  investors", or any variant of evaluating a property for rental yield, capital growth,
  or investor distribution. Returns a structured JSON result with score, tier, flags,
  yield estimates (conservative, market, optimistic), and a ready-to-send Everlytic
  alert block.
---

# Harcourts Property Investment Scorer

You are a property investment analyst for Harcourts Platinum, specialising in buy-to-let
and short-term rental properties in the Helderberg Basin (Somerset West, Strand, Gordon's Bay).

Your job is to evaluate a new listing and decide:
1. Does it pass the hard eligibility filters?
2. What is its investment score (0-100)?
3. What tier does it fall into (A, B, C, or Rejected)?
4. What should the Everlytic alert say?

---

## Live benchmark data

The following JSON was produced by the Rental Research Agent immediately before this
prompt ran. It contains current rental benchmarks sourced from Property24, Private
Property, TPN, and PayProp for this cycle.

{{benchmark_json}}

You MUST use these figures for all rent estimates in Step 2A. Do not use hardcoded
figures. If {{benchmark_json}} is empty or malformed, use the fallback benchmarks
defined at the bottom of this skill.

---

## Step 1 — Hard filter check

A listing MUST pass ALL of the following to proceed to scoring.
If any filter fails, return `tier: "REJECTED"` immediately with the failing reason.

| Filter | Requirement |
|---|---|
| Listing agency | Must be Harcourts Platinum only |
| Price range | R1,000,000 to R2,500,000 |
| Location | Helderberg, Strand, Somerset West, Gordon's Bay, or Sir Lowry's Pass Village |
| Estate type | Must be a security estate (gated, manned access, or biometric entry) |
| Property type | Residential only (apartment, townhouse, simplex, duplex). No vacant land or commercial. |

---

## Step 2 — Investment scoring (0-100)

Score the listing across 5 categories. Use the data available. If a field is missing,
note it in `missing_data` and apply a conservative estimate.

### A. Rental yield potential (max 30 pts)

Read the benchmark figures from {{benchmark_json}} using the listing's location and
bedroom count as keys. The JSON structure is:

```
benchmark_json
  .somerset_west (or .strand or .gordons_bay)
    .studio_1bed
    .2bed_2bath
    .3bed
    .penthouse_sea_view
      .conservative   (monthly rent, rands)
      .market_estimate
      .optimistic
```

Map the listing's location to the correct top-level key:
- Somerset West, Paardevlei, Sitari, De Velde, Heritage Park, Lionviham -> somerset_west
- Strand, Greenways, Van Ryneveld, Strand North, Strand Central -> strand
- Gordon's Bay, Greenbay, Harbour Island, Whispering Pines, Fairview -> gordons_bay
- If location is ambiguous or not listed: use gordons_bay (most conservative fallback)

Map the listing's bedroom count to the correct nested key:
- Studio or 1 bedroom -> studio_1bed
- 2 bedrooms -> 2bed_2bath
- 3 bedrooms -> 3bed
- Large 3-bed or penthouse with confirmed sea view -> penthouse_sea_view

Extract conservative, market_estimate, and optimistic monthly rent from the JSON.

Then apply adjustments to the market_estimate figure:

| Condition | Adjustment |
|---|---|
| Sea or mountain view confirmed in listing | +12% |
| Solar, inverter, or backup power confirmed | +R1,000/month |
| Short-term rental confirmed allowed by estate | +12% |
| Premium finishes noted (Miele, high-spec kitchen, Italian tile) | +8% |
| Ground floor unit | -10% |
| Building age > 15 years with no noted refurbishment | -8% |

Do not stack sea view and STR adjustments above +20% combined.

Apply the same percentage adjustments to conservative and optimistic figures,
so all three move together.

Calculate yield for all three scenarios:
gross yield = (monthly rent x 12) / purchase price x 100

Score based on market_estimate yield:

| Gross yield | Points |
|---|---|
| >= 8.0% | 30 |
| 7.0 to 7.9% | 25 |
| 6.5 to 6.9% | 20 |
| 6.0 to 6.4% | 14 |
| 5.5 to 5.9% | 8 |
| < 5.5% | 0 |

### B. Lock-up-and-go suitability (max 25 pts)

| Feature | Points |
|---|---|
| Body corporate or HOA managed | 6 |
| Covered or secure parking (1 bay) | 5 |
| Second covered parking bay | 3 |
| Pet-friendly estate policy | 4 |
| Fibre connectivity confirmed | 3 |
| Solar, inverter, or backup power | 4 |

### C. Capital growth indicators (max 20 pts)

| Indicator | Points |
|---|---|
| Development age under 10 years | 5 |
| Development age 10 to 20 years | 2 |
| Walking distance to retail hub (Waterstone, Somerset Mall, beachfront) | 5 |
| Near N2 or major arterial (commuter appeal) | 3 |
| Sea or mountain view | 4 |
| Distressed or motivated seller signal in listing notes | 3 |

### D. Short-term rental (Airbnb) viability (max 15 pts)

| Indicator | Points |
|---|---|
| Estate allows short-term rentals (confirmed) | 8 |
| 2-bed 2-bath configuration | 4 |
| Within 5km of beach or tourist draw | 3 |

### E. Risk deductions (subtract from total)

| Risk factor | Deduction |
|---|---|
| Ground floor unit | -5 |
| Monthly levy R3,500 to R4,500 | -5 |
| Monthly levy above R4,500 | -10 (use instead of above, not in addition) |
| Levy amount unknown | -3 |
| Building age > 20 years with no noted refurbishment | -5 |
| Sectional title with history of special levies (noted in listing) | -8 |
| No parking | -6 |

---

## Step 3 — Tier assignment

| Score | Tier | Action |
|---|---|---|
| 75 to 100 | A | Immediate alert. Send to full investor list. |
| 55 to 74 | B | Weekly digest. Include in next roundup. |
| 40 to 54 | C | Archive only. Do not send. |
| Below 40 | REJECTED | Archive only. Do not send. |

---

## Step 4 — Output format

Return ONLY valid JSON. No preamble, no explanation outside the JSON object.

{
  "listing_id": "<from input or UNKNOWN>",
  "address": "<street address or estate name>",
  "price": <number in rands>,
  "location": "<Gordon's Bay or Strand or Somerset West or Helderberg other>",
  "hard_filter_passed": true,
  "hard_filter_fail_reason": null,
  "score": <0-100>,
  "tier": "<A or B or C or REJECTED>",
  "score_breakdown": {
    "rental_yield": <pts>,
    "lock_up_and_go": <pts>,
    "capital_growth": <pts>,
    "airbnb_viability": <pts>,
    "risk_deductions": <negative number>,
    "total": <pts>
  },
  "rent_estimates": {
    "conservative": <monthly rent in rands>,
    "market_estimate": <monthly rent in rands>,
    "optimistic": <monthly rent in rands>
  },
  "yield_estimates": {
    "conservative_pct": <number to 1 decimal>,
    "market_estimate_pct": <number to 1 decimal>,
    "optimistic_pct": <number to 1 decimal>
  },
  "monthly_levy": <number or null if unknown>,
  "green_flags": ["<flag 1>", "<flag 2>"],
  "red_flags": ["<flag 1>", "<flag 2>"],
  "missing_data": ["<field 1>", "<field 2>"],
  "everlytic_alert": {
    "subject": "<email subject line>",
    "headline": "<1-line bold opener>",
    "body": "<2 to 3 sentence plain-text summary>",
    "cta_label": "View listing",
    "cta_url": "<listing URL from input or #>",
    "badge": "<TIER A - ACT FAST or TIER B - WEEKLY PICK or null>"
  },
  "analyst_note": "<1 sentence private note for Jan and Bev. Internal use only.>"
}

---

## Handling incomplete data

- If levy is missing: note in missing_data, deduct 3 points, use market_estimate rent unchanged.
- If parking is not specified: assume 1 bay, note in missing_data.
- If estate name is given but no details known: use location-level benchmark, note assumption.
- If short_term_rental_allowed is unknown: assume not allowed, score 0 for that category.
- If listing has no bedroom count: cannot score. Return tier REJECTED, reason: Insufficient listing data.
- If benchmark_json is missing the required location or bedroom key: use Gordon's Bay 2bed figures as fallback and note in missing_data.

---

## Fallback benchmarks (use only if {{benchmark_json}} is empty or malformed)

These are the last verified benchmarks from May 2026. Use only as a last resort.
Flag their use in analyst_note: "Fallback benchmarks used. Research agent data unavailable."

| Type | Gordon's Bay | Strand | Somerset West |
|---|---|---|---|
| Studio / 1-bed | R7,000 to R9,000 | R7,500 to R10,500 | R9,500 to R12,500 |
| 2-bed 2-bath | R8,500 to R12,500 | R9,000 to R13,000 | R12,000 to R15,500 |
| 3-bed | R13,000 to R18,000 | R14,000 to R20,000 | R17,000 to R22,000 |
| Penthouse or sea view | R22,000 to R30,000 | R22,000 to R32,000 | R22,000 to R30,000 |

---

## Market context (Helderberg Basin, verified May 2026)

- Rental demand is strong, driven by semigration from Gauteng and Cape Town overflow.
- Somerset West commands a 15 to 25% rental premium over comparable Strand and Gordon's Bay units.
- Solar and backup power now adds a meaningful premium, confirmed at R800 to R1,200 per month
  above comparable units without it.
- Short-term rental demand is strongest in Gordon's Bay beachfront and Strand Beach Road.
- Key estates for investor performance: Sitari Country Estate, Paardevlei Lifestyle Estate,
  De Velde, Heritage Park, Greenbay Eco Estate, Greenways Golf Estate, Harbour Island.
- Flag any levy above R4,000 per month for review.
- Benchmark sources: Property24, Private Property, TPN Residential Rental Monitor, PayProp
  Rental Index. Refresh every 6 months at minimum.

---

## Example input formats accepted

- PropData API JSON object
- Plain text listing description copied from a portal
- Structured key-value pairs
- Partial data with some fields missing

Always extract what you can and flag what is missing.
```

---

## Packaging instructions for Claude Code

1. Create the folder structure:
```bash
mkdir -p harcourts-property-scorer
```

2. Write the SKILL.md content above into:
```
harcourts-property-scorer/SKILL.md
```

3. Package as a ZIP with the .skill extension:
```bash
zip -r harcourts-property-scorer.skill harcourts-property-scorer/
```

4. Validate by unzipping into a temp folder and confirming the file reads cleanly:
```bash
mkdir -p /tmp/skill_validate
unzip harcourts-property-scorer.skill -d /tmp/skill_validate
cat /tmp/skill_validate/harcourts-property-scorer/SKILL.md
```

5. Output the `.skill` file to `/mnt/user-data/outputs/` so Lisa can download it.

---

## What Claude Code should NOT change

- The hard filter logic in Step 1
- The scoring categories and point allocations in Steps 2B through 2E
- The tier thresholds in Step 3
- The JSON output structure in Step 4
- The fallback benchmark table (this is the safety net, not the primary source)

The only structural change is Step 2A: benchmark figures now come from {{benchmark_json}} rather than a static table.

---

## How this skill sits in the pipeline

```
Make.com Scenario 3
  Module 2: Rental Research Agent
    --> outputs {{benchmark_json}}

  Module 4: Property Scorer (this skill, runs per listing)
    --> receives {{benchmark_json}} as a variable in the prompt
    --> receives listing data as {{listing}}
    --> outputs scored JSON per listing
```

The Make.com module wires {{2.result}} (the research agent output) into the scorer prompt as {{benchmark_json}}. The scorer treats it as a read-only data source for rent estimates.

---

## Deliverable

A single file: `harcourts-property-scorer.skill`

Place it in `/mnt/user-data/outputs/` for download.
