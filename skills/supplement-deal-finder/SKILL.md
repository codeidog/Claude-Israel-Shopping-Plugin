---
name: supplement-deal-finder
description: Find the best price-per-100g deal on supplements (protein powder, creatine, gainer, pre-workout, BCAA, etc.) across multiple Israeli online retailers — not just one shop. Fans out across the canonical IL supplement specialists (Teva Bari, iHerb-IL) plus a Zap aggregator sweep and name-regex discovery over the merged stores list, parses pack sizes, computes a consistent ₪/100g, handles off-category bundles, and returns a unified ranked table. Use when the user asks things like "best ₪/100g protein in israel", "cheapest creatine across israeli retailers", "compare protein prices israel", "supplement deal scan".
---

# Supplement Deal Finder (cross-retailer ₪/100g, Israel)

Cross-retailer counterpart to `protein-powder-deal-finder` (which is single-retailer, Teva Bari only). Fans out across the IL supplement-retailer landscape, normalizes prices to ₪/100g, and returns a unified ranking. Default category is whey protein powder, but works for any supplement sold by mass (creatine, gainer, pre-workout, BCAA, casein, vegan blends).

## When to use

- User wants the best supplement deal *across* Israeli retailers, not at a specific shop.
- User explicitly invokes a phrase like "best ₪/100g protein in israel", "cheapest creatine in israel", "compare protein prices across israeli sites".

For single-retailer Teva Bari sweeps, defer to `protein-powder-deal-finder` instead — it has the canonical scrape path and bundle list.

## Fan-out targets

Three lanes, run in parallel where possible:

### Lane A — Curated specialist retailers (confirmed)

| Retailer | Domain | Notes |
|---|---|---|
| Teva Bari | `tevabari.co.il` | Joomla/VirtueMart, scrape via the `protein-powder-deal-finder` skill. Free shipping at ₪350. |
| iHerb (IL channel) | `il.iherb.com` | International stock, ships to IL. Prices shown in ILS when geo-detected. Watch the $40 USD IL import threshold — VAT bills above kick in. |

Always include both unless the user excludes one.

### Lane B — Zap aggregator (catch-all)

Delegate to `/search-zap` with the supplement query in Hebrew. Zap surfaces tier-2 specialists this skill doesn't hardcode (e.g. supplement-only shops with limited SEO footprint). Capture the cheapest 3–5 vendors from the Zap model page.

### Lane C — Name-regex discovery over the merged store list

Load the merged stores list (`stores.json` overlaid with `<plugin-data-dir>/user-stores.json` — see `docs/search-strategies.md` § store-metadata merge). Until the upstream `categorisation.categories[]` field is populated, filter by **name regex** instead:

```
sport | fitness | nutri | teva | gnc | gainer | protein | health
```

Take up to 5 hits and try the same Hebrew category query at each store's `website`. Skip stores where `is_tech` is `true`. This is best-effort discovery — many hits will be irrelevant (e.g. sports apparel chains), so cap fetches.

## Workflow

### Step 1 — Resolve category + Hebrew term

- Default: protein powder → `אבקת חלבון` (collective: `אבקות חלבון`)
- Other categories: creatine = `קריאטין`, gainer = `גיינר` / `אבקת גיינר`, pre-workout = `פרי וורקאוט` / `לפני אימון`, BCAA = `BCAA` (left as-is), casein = `קזאין`, vegan protein = `חלבון טבעוני`.
- State the resolved Hebrew term in the first user-facing line so the user can correct the disambiguation if wrong.

### Step 2 — Fire all three lanes in parallel

- **Lane A — Teva Bari:** invoke the `protein-powder-deal-finder` skill workflow (or its core scrape). Get its ranked list with ₪/100g already computed.
- **Lane A — iHerb IL:** WebFetch the IL channel category URL (`il.iherb.com/c/whey-protein` for protein, etc.); for each top product capture name, brand, size, price in ILS. If price is shown in USD, convert via `/convert-currency` at current rate and **flag the row** as USD-derived (FX adds uncertainty).
- **Lane B — Zap:** delegate to `/search-zap` with the Hebrew term. Pull the model-page price range and the cheapest 3 vendors.
- **Lane C — Discovery:** name-regex filter the merged store list (see above), try Hebrew-term search at up to 5 hits via Tavily or homepage form. Cap 2 products per discovered store. Skip silently if nothing parseable.

### Step 3 — Normalize sizes

Parse pack size from the product title or product page:

- Hebrew weight units: `גרם` = g, `ק"ג` / `ק״ג` / `ק"ג` = kg
- English variants: `g`, `gr`, `kg`, `lb` (1 lb = 453.592 g)
- Bundle markers (count the pack): `מארז זוגי` = ×2, `מארז שלישייה` / `מארז משולש` = ×3
- Liquids (`מ"ל` / ml): **do not include in a powder ₪/100g comparison** — flag and exclude.

For products without size in the title, fetch the product page (cap 5 fetches per retailer to keep the run bounded).

### Step 4 — Compute ₪/100g — and don't trust the site

```
₪/100g = price / (size_in_grams / 100)
```

**Always recompute yourself.** Per the Teva Bari skill, several IL supplement sites silently shrink tub sizes (2.27 kg → 2 kg is common) without updating their displayed ₪/100g. Read the *current* title weight, not the legacy ratio.

For bundles, multiply: e.g. dual pack of 2 kg tubs = 4,000 g total.

### Step 5 — Bundles (off-category)

Bundles are the single biggest omission risk. They almost never appear on the master category page — they sit on standalone product URLs. For Teva Bari the `protein-powder-deal-finder` skill already enumerates known bundles. For other retailers, run an extra Hebrew search:

```
site:<retailer-domain> <category-hebrew> מארז זוגי
```

### Step 6 — Render the unified ranked table

Single ranked table across **all** retailers, cheapest ₪/100g first. Columns:

| Column | Notes |
|---|---|
| # | Rank |
| Retailer | Teva Bari / iHerb / Zap-cheapest / … |
| Product | Brand + product name |
| Type | concentrate / isolate / blend / vegan / casein / creatine-mono / etc. |
| Size | g (combined for bundles) |
| Price | ₪ inc. VAT |
| ₪/100g | recomputed |
| Link | direct product URL |
| Notes | `USD→ILS @ rate`, `parallel import`, `shrunk from Xkg`, `bundle ×2`, … |

### Step 7 — Separate sections for non-comparables

- **Specialty subgroup** (isolates, vegan, casein) — listed separately. Not apples-to-apples with whey concentrate on ₪/100g.
- **Excluded — liquids / RTD shakes** — list briefly but do not rank with powders.
- **Out-of-stock** — surface for completeness, mark `OOS`, don't rank.

### Step 8 — Bottom-line + honest coverage note

End with:

1. Single sentence: cheapest powder overall + cheapest bundle.
2. Free-shipping thresholds where known (Teva Bari ₪350; others vary — say "unknown" rather than invent).
3. **Coverage disclosure**, verbatim style:
   > Lanes scanned: Teva Bari (N products), iHerb IL (N), Zap (N vendors), discovery (M stores, K parseable). Bundles checked: <list>. Stores skipped / errored: <list>. For 100% confidence on a single retailer, run `/protein-powder-deal-finder` (Teva Bari) or browse the retailer's category page directly.

## Israeli supplement-market gotchas

- **Shrinkflation without ₪/100g update**: see Teva Bari skill — recompute, don't trust the site's number.
- **Parallel import (יבוא מקביל)**: common for US brands (Optimum Nutrition, Dymatize, MusclePharm). Often the cheapest row. Warranty / quality-assurance is shorter — note it.
- **iHerb VAT threshold**: IL import VAT kicks in at $75 USD landed (post-2024 rule). iHerb shows a VAT-inclusive total only at checkout. For comparisons, add 18% to the iHerb sticker if the cart will exceed $75 USD — flag this in the row.
- **Whey concentrate vs. isolate vs. blend**: concentrate is ~70–80% protein, isolate ~90%, blends vary. A ₪/100g winner that's actually a blend (with maltodextrin / creamer) is a worse ₪/100g-protein deal. Always show `Type` in the table.
- **Pre-workout serving size**: ₪/100g is the wrong unit — switch to ₪/serving when the category is pre-workout (typical serving 8–15 g). Detect category and adjust the unit; mention the switch in the first line.

## Rules

- Never invent a product, price, or retailer URL. If a fetch fails, omit and note in the coverage section.
- All prices in ILS (`₪`), VAT-inclusive. iHerb USD prices must be explicitly converted and flagged.
- Don't mix powders (mass) and liquids (volume) in the same ranking.
- Always show the resolved Hebrew query in the first user-facing line so the user can correct it.
- Cap per-retailer fetches to keep the run bounded (5 product-page fetches per retailer is a reasonable ceiling).
- Don't run more than 3–4 Google IL variants per session (rate limits, per `general-search` rules).

## Examples of valid invocations

- "Best ₪/100g protein across all israeli sites" → default, whey concentrate, all three lanes
- "Cheapest creatine in israel right now" → swap category to creatine, drop bundles section if no creatine bundles exist
- "Compare gainer prices israel" → gainer category, mass-based ranking
- "Best deal on vegan protein in israel" → specialty subgroup is the *main* ranking; mark whey rows as excluded
- "Best pre-workout deal in israel" → switch unit to ₪/serving; ₪/100g not meaningful

## Out of scope

- Single-retailer Teva-Bari-only sweeps → use `protein-powder-deal-finder`.
- Macro / amino-acid profile comparison — only price-per-mass.
- Ordering / cart automation.
