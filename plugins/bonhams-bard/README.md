# Bonhams BARD plugin

Research tools for Bonhams jewellery specialists and management, built on BARD (Bonhams Auction Research Database): lot-level results for jewellery sales at Bonhams, Christie's, Sotheby's and Phillips.

## What it does

| Skill | Use it to |
|---|---|
| `lot-estimate-guide` | Propose an auction estimate for a piece, with comparable sales in original currency and AUD |
| `market-share-brief` | Get a one-page, answer-first brief on Bonhams against the other three houses |
| `sale-recap` | Summarise one completed sale: total, sell-through, top lots, against estimate, against last year |
| `data-health-check` | Find missing sales, mis-scoped sales and field-quality problems by comparing BARD with each house's calendar |
| `bard-home` | Build and publish the interactive BARD home page (Specialist and Management views) |
| `bard-data` | Background guide Claude loads before querying: tools, the `bard_lots` view, currency conversion, hammer versus buyer's total, known data issues |

## Setup

The plugin bundles the `bonhams-bard` connector (`https://bonhams-bard-main-39baa01.d2.zuplo.dev/mcp`). The first time it is used, sign in through the browser prompt. The connector is read-only and exposes three tools: `bard_home` (aggregates), `search_lots` (free text) and `query_table` (filtered rows from the curated `bard_lots` view or the raw `Auction_results` table).

The BARD home page needs the Artifact feature in claude.ai. It uses only `bard_home` and sends no SQL. Publish it once and share the link with the team; colleagues who then type "open BARD" are taken to that shared page instead of creating their own copies, and each sees live data through their own connector.

## Examples

- "Estimate a 5.2 ct Colombian emerald and diamond ring, circa 1930, signed Cartier."
- "How are we doing against Christie's and Sotheby's this year?"
- "Recap Sotheby's Hong Kong Fine Jewelry sale on 23 September."
- "Is the BARD data up to date?"
- "Open BARD" or "bonhams home": opens the team's shared BARD page if one has been shared with you, and otherwise publishes one.

## Changes in 0.3.0

- `bard-data` and `lot-estimate-guide` now search in English, French, Italian, German and Chinese (traditional and simplified), using a term table for stones, makers, origins, treatments and weights. They also match accented and unaccented spellings (for example "Paraíba" and "Paraiba").
- `lot-estimate-guide` lists the decisive details per stone type and asks for them before searching; if they are not given it returns a tiered result at Low confidence. Comparables now record weight, maker, made-by and cased flags, with defined rules for each. A rule for wide comparable spreads, a stale-coverage line under the confidence, and a list of materially different lots were added.
- `bard-data` adds search noise rules and keep-payloads-small guidance.

## Changes in 0.2.2

- The BARD home page's currency switch now offers GBP alongside USD and AUD. GBP figures use the same latest ECB rate as the other conversions.

## Changes in 0.2.0

- Uses the bonhams-bard gateway instead of the Supabase connector; the home page no longer runs SQL.
- `lot-estimate-guide` now searches BARD first (web results pages only as a fallback), converts with the database's exchange rates, uses Bonhams' own estimate ratios, and its output table is corrected.
- New skills: `market-share-brief`, `sale-recap`, `data-health-check`, `bard-data`.
- The home page shows an "Incomplete data" banner when a house has no sale captured for over 60 days, and states the sale scope.
- Database changes made 4 October 2026: French, Italian and German-named jewellery sales (notably Christie's Joaillerie Paris) now count; a curated, read-only `bard_lots` view gives the connector scope, normalised status and USD/AUD prices so results rank correctly across currencies; the connector's timeout rose from 3 to 10 seconds; four indexes were added.

## Known data issues (checked 4 October 2026)

- **Phillips** has not been captured since 10 June 2026; its Hong Kong Jewels Auction of 29 September 2026 is missing. Its year-to-date figures are understated.
- **Christie's** has not been captured since 26 June 2026. A quiet July to September is normal; its November and December sales are the test.
- **Mixed and collection sales** (for example Sotheby's "Fine Jewelry & Watches") are excluded from market figures by design.

No skill modifies the database.
