---
name: sale-recap
description: >
  This skill should be used when someone wants a results summary of a single completed
  jewellery sale at any of the four houses: "recap the Sotheby's Hong Kong sale", "how did
  the Geneva jewels sale go", "results for Bonhams Fine Jewellery London", "top lots from
  last week's sale", "sell-through for that auction", "what did Christie's make in
  Magnificent Jewels", or "compare this sale with last year's". Produces an answer-first
  recap from BARD.
metadata:
  version: "0.2.0"
---

# Sale recap

Summarise one completed sale from BARD. Load the `bard-data` skill first.

## 1. Identify the sale

Ask for the house and sale name or date only if they cannot be inferred. Find the sale with `query_table` on table `bard_lots`:

- filters `auction_house=eq.<house>`, `auction_name=ilike.*<words>*`, and `sale_date=gte.<start>` / `sale_date=lte.<end>` when a date is known
- `select=auction_number,auction_name,sale_date,sale_format,currency`, `limit=5`

A house often runs a live and an online sale under similar names. Pick the sale named in the request, and if two fit, recap the live one and say so. Use `auction_number` for everything that follows. If nothing is found, check the latest captured date in `bard_home` `meta`: the sale may be newer than the data, and say that instead of suggesting the sale did not happen.

## 2. Count without fetching

With `limit=1`, `totalRows` is an exact count. Run these for the sale:

- all lots (`auction_number=eq.<n>`)
- sold (`status_norm=eq.sold`)
- unsold (`status_norm=eq.unsold`)

Sell-through is sold divided by offered. If sold plus unsold falls short of offered, state that some lots have no status and report sell-through on lots with a status.

## 3. Pull the money

Fetch sold lots from `bard_lots` with `select=lot_number,lot_title,currency,est_low,est_high,hammer_price,sold_price,sold_usd,sold_aud,estimate_band,lot_url`, `is_sold=eq.true`, `order=sold_usd.desc`, `limit=200`. Page with `offset` for sales above 200 sold lots (stop at three pages; the total can then be marked as covering the top 600 lots).

Compute from the rows:

- **Total**: sum of `sold_price` in the sale currency, and of `sold_usd` and `sold_aud`.
- **Top five lots**, each with title, price and link.
- **Against estimate**, counting lots below the low, within, and above the high. At Bonhams use `estimate_band` (hammer against estimate). At the other houses only buyer's total exists, which includes premium, so compare `sold_price ÷ 1.28` and label the result approximate, or report only the top-lot and total figures. State which approach was used.
- **Notable unsold**: the highest-estimate unsold lots (`status_norm=eq.unsold`, ordered by `est_low_usd.desc`).

## 4. Compare with the previous edition

Find the same house's comparable sale a year earlier (same city and season, so a similar `auction_name` and `auction_date` about twelve months back) and repeat the counts and total. Give the change in lots offered, sell-through and total in USD. If there is no like-for-like sale, skip the comparison and say why.

## 5. Output

Lead with the verdict in one or two sentences (total, sell-through, and how it compares with last year), then:

| Metric | This sale | Same sale last year |
|---|---|---|

Then the top five lots, a line on estimate performance, a line on notable unsold lots, and data caveats (price basis, lots without status, any stale-coverage warning). Keep it to a page. Offer to compare the sale against Bonhams' equivalent.
