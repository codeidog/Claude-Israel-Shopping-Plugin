---
name: supplement-deal-finder
description: Find the best price-per-100g deal on supplements (protein powder, creatine, gainer, pre-workout, BCAA, etc.) across multiple Israeli online retailers — not just one shop. Fans out across the canonical IL supplement specialists (Teva Bari, iHerb-IL) plus a Zap aggregator sweep and name-regex discovery over the merged stores list, parses pack sizes, computes a consistent ₪/100g, handles off-category bundles, and returns a unified ranked table. Use when the user asks things like "best ₪/100g protein in israel", "cheapest creatine across israeli retailers", "compare protein prices israel", "supplement deal scan".
---

# Supplement Deal Finder (cross-retailer ₪/100g, Israel)

Cross-retailer skill for finding the best supplement deals across Israeli online shops. Normalizes prices to ₪/100g and returns a unified ranking. Default category is whey protein powder, but works for any supplement sold by mass (creatine, gainer, pre-workout, BCAA, casein, vegan blends).

This skill is **self-contained** — everything needed to scrape each retailer is documented below. No dependency on any other skill.

## When to use

- User wants the best supplement deal *across* Israeli retailers.
- User explicitly invokes a phrase like "best ₪/100g protein in israel", "cheapest creatine in israel", "compare protein prices across israeli sites".

## Fan-out targets

Three lanes, run in parallel where possible:

### Lane A — Curated specialist retailers (confirmed)

| Retailer | Domain | Method | Free-shipping threshold |
|---|---|---|---|
| Teva Bari | `tevabari.co.il` | curl + parse (Joomla/VirtueMart, data-* attributes, no JS needed) — see § Teva Bari scrape recipe below | ₪350 |
| iHerb (IL channel) | `il.iherb.com` | WebFetch the IL category URL; prices usually shown in ILS when geo-detected, occasionally USD — convert and flag | varies, check cart |

Always include both unless the user excludes one.

### Lane B — Zap aggregator (catch-all)

Delegate to `/search-zap` with the supplement query in Hebrew. Zap surfaces tier-2 specialists this skill doesn't hardcode (e.g. supplement-only shops with limited SEO footprint). Capture the cheapest 3–5 vendors from the Zap model page.

### Lane C — Name-regex discovery over the merged store list

Load the merged stores list (`stores.json` overlaid with `<plugin-data-dir>/user-stores.json` — see `docs/search-strategies.md` § store-metadata merge). Until the upstream `categorisation.categories[]` field is populated, filter by **name regex**:

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

**For every product captured, the direct product URL is mandatory** (see § Link-capture rules below). If a fetch returns a price but no resolvable URL, drop the row — a row without a link is useless to the user.

- **Lane A — Teva Bari:** follow the inline scrape recipe in § Teva Bari scrape recipe below; use the URL-extraction variant of the scrape (the second `perl -0777` block) so each row carries its `data-url`. For bundle URLs, use the canonical bundle table.
- **Lane A — iHerb IL:** WebFetch the IL channel category URL (`il.iherb.com/c/whey-protein` for protein, etc.); for each top product capture name, brand, size, price in ILS, **and the product-page URL** (typically `il.iherb.com/pr/...`). If price is shown in USD, convert via `/convert-currency` at current rate and **flag the row** as USD-derived (FX adds uncertainty).
- **Lane B — Zap:** delegate to `/search-zap` with the Hebrew term. From the Zap model page, capture for each top vendor: vendor name, price, **and the click-through URL to the vendor's own product page** (the link Zap labels "לרכישה" / "לאתר המוכר"). The Zap model page itself is also useful — include it as a secondary "see-all-vendors" link in the row notes, but the ranking link must be the direct vendor URL so the user can complete the purchase in one click. **Every Zap-sourced row must then pass the validation pass in § Zap row validation below before it appears in the ranking.**
- **Lane C — Discovery:** name-regex filter the merged store list (see above), try Hebrew-term search at up to 5 hits via Tavily or homepage form. Cap 2 products per discovered store. **Every row must include the direct product URL**; skip silently if a hit doesn't yield one.

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

**Always recompute yourself.** Several IL supplement sites silently shrink tub sizes (2.27 kg → 2 kg is common) without updating their displayed ₪/100g. Read the *current* title weight, not the legacy ratio. See § Stale-weight gotcha below for the recurring pattern.

For bundles, multiply: e.g. dual pack of 2 kg tubs = 4,000 g total.

### Step 5 — Bundles (off-category)

Bundles are the single biggest omission risk. They almost never appear on the master category page — they sit on standalone product URLs. For Teva Bari, see the known-bundles table in § Teva Bari scrape recipe. For other retailers, run an extra Hebrew search:

```
site:<retailer-domain> <category-hebrew> מארז זוגי
```

### Step 6 — Render the unified ranked table

Single ranked table across **all** retailers, cheapest ₪/100g first. Columns:

| Column | Required | Notes |
|---|---|---|
| # | ✓ | Rank |
| Retailer | ✓ | Teva Bari / iHerb / Zap-cheapest / … |
| Product | ✓ | Brand + product name |
| Type | ✓ | concentrate / isolate / blend / vegan / casein / creatine-mono / etc. |
| Size | ✓ | g (combined for bundles) |
| Price | ✓ | ₪ inc. VAT |
| ₪/100g | ✓ | recomputed |
| **Link** | **✓ MANDATORY** | Direct product URL where the user can complete the purchase. Render as a markdown link: `[Buy](https://…)`. **No row without a working link.** |
| Notes |   | `USD→ILS @ rate`, `parallel import`, `shrunk from Xkg`, `bundle ×2`, … |

A ranking without buyable links defeats the whole purpose of this skill — the user has to be one click away from buying the cheapest option.

### Step 7 — Separate sections for non-comparables

- **Specialty subgroup** (isolates, vegan, casein) — listed separately. Not apples-to-apples with whey concentrate on ₪/100g.
- **Excluded — liquids / RTD shakes** — list briefly but do not rank with powders.
- **Out-of-stock** — surface for completeness, mark `OOS`, don't rank.

### Step 8 — Bottom-line + honest coverage note

End with:

1. Single sentence: cheapest powder overall + cheapest bundle.
2. Free-shipping thresholds where known (Teva Bari ₪350; others vary — say "unknown" rather than invent).
3. **Coverage disclosure**, verbatim style:
   > Lanes scanned: Teva Bari (N products), iHerb IL (N), Zap (N candidates → V validated, D dropped), discovery (M stores, K parseable). Zap validation: ✓ V ranked, ⚠ P price-drifted (live used), ✗ D dropped (OOS=A, 404=B, blocked=C, timeout=E). Bundles checked: <list>. Stores skipped / errored: <list>. For 100% confidence on a single retailer, browse its category page directly.

## Teva Bari scrape recipe (inline)

`tevabari.co.il` is a Joomla + VirtueMart site. Product data is **embedded in HTML as `data-*` attributes** on `<div class="product">` elements — no JS rendering needed, no Playwright required. Each product div has: `data-id`, `data-name`, `data-price`, `data-brand`, `data-available`, `data-category`. The master category page shows **only in-stock products** (out-of-stock items are filtered out entirely; `data-available="1"` is reliable for in-stock state).

### Category URLs

Default — protein powder (URL-encoded Hebrew, decoded path `כושר-ופיתוח-גוף/אבקות-חלבון/כל-אבקות-החלבון`):

```
https://www.tevabari.co.il/%D7%9B%D7%95%D7%A9%D7%A8-%D7%95%D7%A4%D7%99%D7%AA%D7%95%D7%97-%D7%92%D7%95%D7%A3/%D7%90%D7%91%D7%A7%D7%95%D7%AA-%D7%97%D7%9C%D7%91%D7%95%D7%9F/%D7%9B%D7%9C-%D7%90%D7%91%D7%A7%D7%95%D7%AA-%D7%94%D7%97%D7%9C%D7%91%D7%95%D7%9F
```

For other categories (creatine, gainer, pre-workout), discover the master "all-X" URL with:

```
WebSearch site:tevabari.co.il <category> כל ה<category>
```

…or browse the manufacturer/category nav.

### Scrape the canonical product list

Via Bash:

```bash
curl -sL "<MASTER_URL>" \
  -A "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36" \
  -o /tmp/teva_category.html

# Quick view — name, price, availability, brand, id
grep -oE 'div class="product"[^>]*data-name="[^"]+"[^>]*' /tmp/teva_category.html | \
  perl -ne 'if (/data-id="([^"]+)".*?data-name="([^"]+)".*?data-price="([^"]+)".*?data-brand="([^"]+)".*?data-available="([^"]+)"/) {
    my ($id,$n,$p,$b,$a)=($1,$2,$3,$4,$5);
    $n =~ s/&amp;/&/g; $n =~ s/&quot;/"/g; $n =~ s/&#039;/'\''/g; $n =~ s/&amp;ndash;/-/g;
    print "ID=$id\tPRICE=$p\tAVAIL=$a\tBRAND=$b\tNAME=$n\n";
  }'
```

To also extract product URLs:

```bash
perl -0777 -ne 'while (/<div class="product"[^>]*data-id="(\d+)"[^>]*data-name="([^"]+)"[^>]*data-price="([^"]+)"[^>]*data-available="([^"]+)"[^>]*>(.*?)<\/div>\s*<\/div>\s*<\/div>/gs) {
  my ($id,$n,$p,$a,$blk)=($1,$2,$3,$4,$5);
  my $url = ""; if ($blk =~ /href="(\/[^"#?]+)"/) { $url = $1; }
  $n =~ s/&amp;/&/g; $n =~ s/&quot;/"/g; $n =~ s/&#039;/'\''/g; $n =~ s/&amp;ndash;/-/g;
  print "PRICE=$p\tAVAIL=$a\tURL=$url\tNAME=$n\n";
}' /tmp/teva_category.html
```

### Known protein-powder bundles (off-category — must check separately)

Bundles sit on standalone product URLs and are **not** on the master category page. For the default protein-powder run, also WebFetch each of these to capture current price + weight + stock:

| Bundle | URL |
|---|---|
| Super Effect Whey Dual Pack | https://www.tevabari.co.il/%D7%A1%D7%95%D7%A4%D7%A8-%D7%90%D7%A4%D7%A7%D7%98-%D7%90%D7%91%D7%A7%D7%AA-%D7%97%D7%9C%D7%91%D7%95%D7%9F-%D7%9E%D7%90%D7%A8%D7%96-%D7%96%D7%95%D7%92%D7%99-super-effect |
| Allin Whey 2-pack | https://www.tevabari.co.il/allin-whey-protein-mix-powder-2pack |
| ALFA Whey Dual Pack | https://www.tevabari.co.il/%D7%90%D7%9C%D7%A4%D7%90-%D7%97%D7%9C%D7%91-%D7%99%D7%A9%D7%A8%D7%90%D7%9C-%D7%9E%D7%90%D7%A8%D7%96-%D7%96%D7%95%D7%92%D7%99 |
| Combat 100% Whey Dual Pack | https://www.tevabari.co.il/muscle-pharm-combat-whey-bouble |

For categories other than protein, search for bundles with:

```
WebSearch site:tevabari.co.il <category-hebrew> מארז זוגי
```

### Size handling for Teva Bari

Many product names include size in the title (e.g. `759 גרם`, `2 ק״ג`). For products without size in the name, WebFetch the individual product page to extract size + verify stock. Out-of-stock individual pages show `חסר בארץ`.

### Stale-weight gotcha

Several products on Teva Bari (Super Effect Dual Pack, Allin, GO Whey, others) had their tub size silently reduced (commonly 2.3 kg → 2 kg per tub, or 2.27 kg → 2 kg) but the page's own stated ₪/100g was NOT recalculated. Always:

1. Read the product **title** for the current weight (e.g. `מארז זוגי X ק"ג`).
2. Look for an `הודעה חשובה: ... הוקטנה ... כעת במשקל X ק"ג במקום Y ק"ג` notice on the product page.
3. Look for spec lines that show **two weights** (e.g. `4.54 ק"ג - 4 ק"ג`) — that's the old/new conflict.
4. **Recompute ₪/100g yourself using the new weight.** Do NOT trust the site's `X ₪ ל-100 גרם` text — it's often based on the old weight and can be ~14% off.

Worked example: Super Effect Dual Pack lists ₪9.90/100g, but tubs went from 2.27 kg → 2 kg. Real total is 4,000 g, not 4,540 g. Actual ₪/100g = 449.90 ÷ 4,000 × 100 = **₪11.25**, not ₪9.90.

## Israeli supplement-market gotchas

- **Shrinkflation without ₪/100g update**: documented above for Teva Bari; similar pattern appears at other supplement retailers — recompute, don't trust the site's number.
- **Parallel import (יבוא מקביל)**: common for US brands (Optimum Nutrition, Dymatize, MusclePharm). Often the cheapest row. Warranty / quality-assurance is shorter — note it in the row.
- **iHerb VAT threshold**: IL import VAT kicks in at $75 USD landed (post-2024 rule, currently 18%). iHerb shows a VAT-inclusive total only at checkout. For comparisons, add 18% to the iHerb sticker if the cart will exceed $75 USD — flag this in the row notes.
- **Whey concentrate vs. isolate vs. blend**: concentrate is ~70–80% protein, isolate ~90%, blends vary. A ₪/100g winner that's actually a blend (with maltodextrin / creamer) is a worse ₪/100g-protein deal. Always show `Type` in the table.
- **Pre-workout serving size**: ₪/100g is the wrong unit — switch to ₪/serving when the category is pre-workout (typical serving 8–15 g). Detect category and adjust the unit; mention the switch in the first user-facing line.
- **Don't trust a single search.** First-pass coverage is typically ~35%. The master category scrape (Teva Bari) plus Zap aggregator (cross-retailer) is the authoritative pair.

## Link-capture rules

Links are the most important field in the output. The user reads this skill's result to decide what to buy, then clicks through to buy it. A row without a working link is dead weight.

- **Every ranked row MUST include a direct product URL.** No exceptions. If you have a price but no URL, drop the row from the ranking and note the skip in the coverage section.
- **URL must be the retailer's own product page**, not a search-results page, category page, or aggregator landing page. The user should land on the product they're buying, not a list to wade through.
- **Render as markdown:** `[Buy](https://exact-url)`. Not bare URLs, not "see retailer". This matters for tables.
- **Teva Bari URLs** come from the URL-extraction variant of the scrape (the `perl -0777` block in § Teva Bari scrape recipe — captures the first `href="/..."` inside each product block). Always run that variant, not the quick-view variant.
- **iHerb URLs** are stable `https://il.iherb.com/pr/...` slugs — capture from the category-page result cards.
- **Zap rows** must link to the **vendor's** product page (the "לרכישה" / "לאתר המוכר" click-through), not to the Zap model page. The Zap model page can go in the Notes column as a secondary "compare-vendors" link.
- **Discovery (Lane C) rows** must link to the discovered store's actual product page. If the only thing you can extract is a search-results URL, don't include the row.
- **Never invent or guess a URL.** If you're uncertain a URL is correct, don't include the row.
- **Verify before output (validation step):** before rendering the final table, scan the ranking and confirm every row has a non-empty link cell. If any row is missing one, either backfill it (re-fetch the product page) or drop the row.

## Zap row validation

Zap is an aggregator — it caches vendor data and the snapshot drifts. By the time the user sees a Zap row, the underlying vendor page may have a different price, be out of stock, or have moved/removed the product. Showing a stale Zap row leads the user to click through, see a different reality, and lose trust in the ranking.

**Every Zap-sourced row must be validated against the vendor's own product page before it appears in the ranking.** No validation, no row.

### Validation pass

For each candidate row produced by `/search-zap`:

1. **Fetch the vendor's product page** at the click-through URL captured from Zap (WebFetch or curl, depending on the vendor — Playwright fallback only if both fail).
2. **Extract the live price** from the vendor's own page (their price element / structured-data block / og-meta — not Zap's).
3. **Extract live stock state** — out-of-stock markers vary by retailer (Hebrew `אזל מהמלאי` / `חסר במלאי` / `חסר בארץ`; English `out of stock`, `sold out`; product page that returns to a category page is also a signal).
4. **Compare against Zap's snapshot:**

   | Outcome | Action |
   |---|---|
   | Page loads, price within **±3%** of Zap's, in stock | ✅ Include row. Use the **vendor's live price** in the table (not Zap's). |
   | Page loads, price differs > 3% | ⚠️ Include row with the **vendor's live price**, and add a `Zap snapshot: ₪X → live ₪Y` note in the Notes column. Re-rank if the new price changes the position. |
   | Page loads but OOS | ❌ **Drop the row** from the main ranking. Optionally surface in the "Out of stock" section. |
   | Page 404 / redirects to category / product moved | ❌ **Drop the row**. Note the broken Zap link in the coverage section. |
   | Page hidden behind login / captcha / blocked | ❌ **Drop the row**. Note in coverage. Don't loop on captchas. |
   | Fetch times out or errors | ❌ **Drop the row**. Note in coverage. |

5. **Never display a Zap row without going through this pass.** If validation can't be performed for any reason, the row doesn't make it into the output.

### Why this matters

- Zap's price is what Zap had last time it crawled the vendor — could be hours, could be days old.
- Out-of-stock items linger on Zap until the next crawl — the ranking would steer the user to a dead product.
- Vendor URLs go stale when retailers rename slugs or restructure their catalog.
- The whole point of this skill is "one click to buy" — a broken or stale Zap link breaks that promise.

### What to write in the coverage section

After the ranking, in the coverage disclosure, report Zap validation outcomes explicitly:

```
Zap candidates: N
  ✓ Validated and ranked: X
  ⚠ Price-drifted (live price used): Y — <list vendor names>
  ✗ Dropped: Z — <reasons: OOS=A, 404=B, blocked=C, timeout=D>
```

This gives the user transparency about why some Zap-listed vendors didn't make the table.

## Rules

- Never invent a product, price, or retailer URL. If a fetch fails, omit and note in the coverage section.
- All prices in ILS (`₪`), VAT-inclusive. iHerb USD prices must be explicitly converted and flagged.
- Don't mix powders (mass) and liquids (volume) in the same ranking.
- Always show the resolved Hebrew query in the first user-facing line so the user can correct it.
- Cap per-retailer fetches to keep the run bounded (5 product-page fetches per retailer is a reasonable ceiling).
- Don't run more than 3–4 Google IL variants per session (rate limits, per `general-search` rules).

## Validation checklist (before declaring success)

1. Every row in the main ranked table has a working markdown link in the Link column. No bare text, no "see retailer", no empty cells.
2. **Every Zap-sourced row went through the § Zap row validation pass — vendor page fetched, price/stock verified, live price used. No unvalidated Zap rows in the output.**
3. Resolved Hebrew query was stated in the first user-facing line.
4. All prices are in ILS, VAT-inclusive (iHerb USD rows explicitly flagged).
5. ₪/100g was recomputed from current size, not copied from the retailer's displayed value.
6. Bundles section (for protein) checks each known Teva Bari bundle URL.
7. Specialty subgroup (isolate / vegan / casein) is listed separately from concentrate-on-mass ranking.
8. Coverage disclosure at the end lists lanes scanned, product counts, any skipped stores, **and Zap validation outcomes (validated / price-drifted / dropped with reasons)**.

## Examples of valid invocations

- "Best ₪/100g protein across all israeli sites" → default, whey concentrate, all three lanes
- "Cheapest creatine in israel right now" → swap category to creatine, drop bundles section if no creatine bundles exist
- "Compare gainer prices israel" → gainer category, mass-based ranking
- "Best deal on vegan protein in israel" → specialty subgroup is the *main* ranking; mark whey rows as excluded
- "Best pre-workout deal in israel" → switch unit to ₪/serving; ₪/100g not meaningful

## Out of scope

- Macro / amino-acid profile comparison — only price-per-mass.
- Ordering / cart automation.
