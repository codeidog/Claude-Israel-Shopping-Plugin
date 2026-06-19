![banner](banner.png)

# Israel-Shopping

Israeli shopping workflows for Claude Code — tech retailers, Zap price comparison, Hebrew term resolution, ILS conversion, and RRP/market checks.

> Renamed from `Israel-Skills` on 2026-04-20. The scope was always shopping; the new name reflects it. Non-shopping Israel utilities will live in a separate plugin.

## Commands (slash)

- `/tech-product-search` — search tier-1 IL tech retailers (Ivory, KSP, Bug, TMS) with Hebrew-term resolution and a Tavily→WebFetch→Playwright fallback chain.
- `/general-search` — broader Israeli retail search (Ace, Home Center, Office Depot, Audioline, Google IL, Zap).
- `/search-zap` — query zap.co.il, the canonical Israeli price-comparison aggregator.
- `/rrp-check` — compare an Israeli price against international RRP / market pricing.
- `/convert-currency` — ILS ↔ USD/EUR/GBP conversion.
- `/pn-check` — given an IL product listing, find the canonical manufacturer part-number on the international market.
- `/store-zap-lookup` — given a retailer name, find its Zap profile URL, rating, and review count.
- `/whats-this-brand` — identify an obscure Israeli-market brand (white-label, importer-specific, OEM equivalent).
- `/freier-check` — playful "are you a freier" verdict given an IL price and international RRP.
- `/add-store` — append a vendor to the user's persistent store overlay (survives plugin updates).

Strategy and know-how (Hebrew terms, URL patterns, Playwright fallback, sourcing waterfall, store-merge convention) live in [`docs/search-strategies.md`](docs/search-strategies.md). The commands reference it rather than restating.

## Skills (auto-invoked)

Skills under [`skills/`](skills/) trigger from natural-language phrases without an explicit slash command:

- `search-aliexpress`, `free-shipping-only`, `exclude-combo-deals`, `il-reviews-show`, `fetch-listing`, `fetch-listing-api`, `compare-to-local` — AliExpress IL-context tooling (search with IL filters, scrape listing with landed cost + 18% VAT, filter reviews to Israeli buyers, compare to local).
- `supplement-deal-finder` — cross-retailer ₪/100g ranker for supplements (whey, creatine, gainer, pre-workout, etc.) across Teva Bari, iHerb-IL, Zap, and discovered specialists. Handles bundles, shrinkflation, parallel-import notes, and switches to ₪/serving for pre-workout.

## Data

Store metadata is read live from [`danielrosehill/Israel-Online-Stores`](https://github.com/danielrosehill/Israel-Online-Stores) (`stores.json`) — 800+ Israeli retailers with tier, delivery, Eilat-door, and Zap-profile metadata. User-added stores overlay this list from `<plugin-data-dir>/user-stores.json` (`$CLAUDE_USER_DATA/israel-shopping/user-stores.json`, or the XDG/`~/.local/share/claude-plugins/israel-shopping/` fallback — see the `meta-tools:plugin-data-storage` canonical skill) and survive plugin updates.

# Sources

None are bundled yet — see [`INTEGRATIONS.md`](INTEGRATIONS.md) for status. These tables credit the upstream projects considered as roadmap dependencies.

## Consumer rights / legal

| Project | Description |
| --- | --- |
| [`Ansvar-Systems/israel-law-mcp`](https://github.com/Ansvar-Systems/israel-law-mcp) | Hosted MCP exposing Israeli statutes; check Consumer Protection Law coverage. |

## Currency / FX

| Project | Description |
| --- | --- |
| **TODO** | No candidate yet. Needs a mid-market-rate FX MCP (exchangerate.host, Frankfurter, open.er-api.com) for reproducible cross-border comparisons. |

## Domestic retailers & price comparison

| Project | Description |
| --- | --- |
| [`danielrosehill/Israeli-Tech-Shopping-MCP`](https://github.com/danielrosehill/Israeli-Tech-Shopping-MCP) | Own project: browser-automation comparison across IL tech retailers. |
| [`guymon92/ksp-mcp`](https://github.com/guymon92/ksp-mcp) | KSP.co.il MCP. |
| [`Simtob-Eran/mcp-israeli-price-comparison`](https://github.com/Simtob-Eran/mcp-israeli-price-comparison) | Generic IL price-comparison MCP. |
| [`TalKleinBgu/Zap`](https://github.com/TalKleinBgu/Zap) | Zap.co.il product dedup / normalization pipeline. |

## Grocery

| Project | Description |
| --- | --- |
| [`aloncarmel/supermeskill`](https://github.com/aloncarmel/supermeskill) | SuperMe skill. |
| [`asafr93-rosa/shopping-list-il`](https://github.com/asafr93-rosa/shopping-list-il) | Shopping-list tool. |
| [`danielJL-altius/israelgrocerymcp`](https://github.com/danielJL-altius/israelgrocerymcp) | Alt grocery MCP. |
| [`matipojo/shufersal-mcp`](https://github.com/matipojo/shufersal-mcp) | Shufersal MCP. |
| [`OpenIsraeliSupermarkets/israeli-supermarket-scarpers`](https://github.com/OpenIsraeliSupermarkets/israeli-supermarket-scarpers) | Python scraper library (Shufersal, Rami Levy, Victory, etc.). |

## International / cross-border

| Project | Description |
| --- | --- |
| Amazon | **TODO** — no candidate yet that handles ships-to-IL eligibility. |
| [`justinritchie/aliexpress-mcp-server`](https://github.com/justinritchie/aliexpress-mcp-server) | AliExpress MCP for cross-border lookups. |
| Newegg | **TODO** — no candidate yet. |
| [`rootsbymenda/iherbchecker`](https://github.com/rootsbymenda/iherbchecker) | iHerb availability/price check. |

## Pharmacy

| Project | Description |
| --- | --- |
| [`alexpolonsky/agent-skill-maccabi-pharm-search`](https://github.com/alexpolonsky/agent-skill-maccabi-pharm-search) | Maccabi pharmacy search. |
| [`skills-il/health-services`](https://github.com/skills-il/health-services) | Broader health services (lift pharmacy sub-skills). |
| [`tomron/agent-skill-clalit-pharm-search`](https://github.com/tomron/agent-skill-clalit-pharm-search) | Clalit pharmacy search. |

## Installation

```bash
claude plugins marketplace add danielrosehill/Claude-Code-Plugins
claude plugins install israel-shopping@danielrosehill
```

## License

MIT
