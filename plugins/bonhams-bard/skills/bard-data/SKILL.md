---
name: bard-data
description: >
  This skill should be used whenever Claude reads BARD (Bonhams Auction Research Database)
  through the bonhams-bard connector: calling bard_home, search_lots or query_table,
  querying "bard_lots" or "Auction_results", looking up auction lots or sale results,
  converting auction prices between currencies, comparing Bonhams with Christie's,
  Sotheby's or Phillips, or interpreting hammer price, buyer's total, sell-through or
  estimate figures. Load it before the first BARD query in a conversation, and whenever
  a BARD number looks odd.
metadata:
  version: "0.2.0"
---

# BARD data guide

BARD holds lot-level results for jewellery sales at four houses: Bonhams, Christie's, Sotheby's and Phillips. Read it only through the `bonhams-bard` connector, which is read-only.

## The three tools

| Tool | Use it for | Notes |
|---|---|---|
| `bard_home` | Aggregates: FX rates, per-house coverage and caveats, financial-year totals, regions, year-to-date, monthly trend, concentration, estimate performance, recent top lots | No arguments. Returns a cached model refreshed every three hours; `generated_at` says when. Call it first for any market-level question. |
| `search_lots` | Free-text lookup ("Kashmir sapphire ring", "Cartier tutti frutti") | Parameters `search_term`, `max_results` (up to 200). Every word must appear in the title or description. Returns raw rows, newest first, prices in sale currency only. |
| `query_table` | Exact filtering, ordering and paging | Parameters `table`, `filter`, `select`, `order`, `limit` (max 200), `offset`. Use table **`bard_lots`** by default; `Auction_results` is the raw table behind it. |

Choose by question type: aggregates from `bard_home`, discovery from `search_lots`, precise comparable sets and whole-sale pulls from `query_table` on `bard_lots`. Never loop over many pages to compute something `bard_home` already contains.

## Columns in bard_lots (curated, preferred)

- Sale: `auction_house` (`Bonhams`, `Christies`, `Sothebys`, `Phillips`), `auction_name`, `auction_number`, `sale_date` (a date), `sale_format` (`live` or `online`), `sale_scope` (`jewellery_sale`, `mixed_sale`, `collection_sale`, `non_jewellery_sale`), `region`, `fy` (integer, e.g. 2026), `fy_label` (e.g. FY26), `sale_month`
- Lot: `lot_number`, `lot_title`, `lot_description`, `lot_url`, `image_url`, `lot_class` (jewellery, watch, bag, pen, other), `item_type` (Ring, Necklace, Earrings, Bracelet, Brooch, Pendant, Cufflinks, Loose stone, Watch, Other), `is_jewellery`
- Result: `status` (raw), `status_norm` (`sold`, `unsold`, `unknown`), `is_sold` (boolean), `estimate_band` (`Below`, `Within`, `Above`, Bonhams only)
- Money in sale currency: `currency`, `est_low`, `est_high`, `hammer_price`, `sold_price`
- Money converted: `usd_rate`, `sold_usd`, `hammer_usd`, `est_low_usd`, `est_high_usd`, `sold_aud`, `hammer_aud`

`Auction_results` has the raw columns only (`auction_date` as ISO text, no converted prices, no scope). Use it only if `bard_lots` is unavailable.

## query_table patterns

Each `filter` entry is `column=op.value`. Operators: `eq`, `neq`, `gt`, `gte`, `lt`, `lte`, `like`, `ilike` (use `*` as wildcard), `in`, and `is.null`.

```
table: bard_lots
filter: ["is_sold=eq.true", "sale_date=gte.2023-10-01", "lot_title=ilike.*cartier*", "auction_house=in.(Bonhams,Sothebys)"]
order: sold_usd.desc
select: auction_house,auction_name,sale_date,lot_title,currency,est_low,est_high,hammer_price,sold_price,sold_usd,sold_aud,lot_url
```

- **Count without fetching:** set `limit=1`. `totalRows` is an exact count for the filters.
- **Keep rows small:** always pass `select`; `lot_description` is long.
- **Page:** `offset` in steps of 200; stop after about five pages and narrow the filters instead.
- **Cold starts:** the database sleeps when idle. The connector allows 10 seconds per query; if a first query still fails with a timeout (HTTP 502), retry once before reporting an error.

## Rules that stop wrong answers

1. **Rank and sum in USD or AUD, never in sale currency.** Order by `sold_usd` (or `sold_aud`), not `sold_price`, whenever more than one currency can appear. `sold_price` ordering is only valid inside one sale or one currency.
2. **Conversion basis.** Converted columns use the latest European Central Bank rate for every period, so year-on-year changes exclude currency movement. `bard_home` `fx` rows (`[currency, usd_rate, rate_date]`) give the rate date; state it once when showing converted figures. Search results from `search_lots` are unconverted: convert with `fx` (USD = amount × usd_rate; AUD = USD ÷ AUD usd_rate).
3. **Hammer versus buyer's total.** Estimates are set against the hammer. Only Bonhams has `hammer_price`; the other houses publish `sold_price` (buyer's total) only. At Bonhams, buyer's total runs at almost exactly 1.28 times hammer across currencies and price bands (about 1.27 above one million). To set a competitor result against Bonhams estimates, divide by 1.28 as a rough hammer-equivalent and label it approximate, because other houses' premium schedules differ. Prefer buyer's total versus buyer's total when both sides have it.
4. **Status.** Use `status_norm` and `is_sold`. Exclude `unknown` from sell-through, ignore prices on lots that are not sold, and drop sold lots with a null or zero price from value statistics.
5. **Duplicates.** A few Bonhams lots repeat on (`auction_house`, `auction_number`, `lot_number`). Dedupe on that key before counting.
6. **Live and online.** BARD captures Bonhams' full online calendar but mostly only competitors' flagship live sales, so compare houses on `sale_format=eq.live`.
7. **Scope.** Market figures use `sale_scope=eq.jewellery_sale`: sales named for jewellery in English, French, Italian or German ("jewel", "joaillerie", "bijoux", "gioielli", "schmuck"), excluding names that also mention watches, handbags, leather goods, fashion, pens or wine. Mixed jewels-and-watches sales and single-owner collection sales are left out, even when mostly jewellery; disclose that when comparing houses.
8. **Coverage and staleness.** Before quoting any market figure, read `meta` from `bard_home` (`[house, latest_sale_date, sell_through_reliable, caveat, dedicated_live_sales]`). If a house's latest sale is more than 60 days old, say its figures are incomplete. A missing sale is not evidence that no sale happened.
9. **History.** Bonhams, Christie's and Sotheby's run from January 2021; Phillips reaches back to 2013.
10. **Regions** come from the city in the sale name, or from the sale currency when no city is named.
11. **Financial year** runs July to June, labelled by the June year (FY26 = July 2025 to June 2026).
12. **Item type** is matched on English keywords in the lot title, so French-titled lots (for example at Christie's Paris) fall into Other.

## Search tips

Lot descriptions carry carat weights, origin, treatment and certificates ("Kashmir", "no indications of heating", "SSEF", "GIA"). `search_lots` requires every word, so use two or three short, distinct searches rather than one long one. Sotheby's Hong Kong titles include Chinese text; search the English description terms. Dedupe on `lot_url`.

## More detail

`references/known-issues.md` lists known data gaps, their measured size and the date they were checked, plus the changes made to the database. Read it when a figure looks wrong or before publishing market share.
