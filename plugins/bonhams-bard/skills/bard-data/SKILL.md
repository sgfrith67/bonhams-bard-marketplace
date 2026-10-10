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
  version: "0.3.0"
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
12. **Item type** is matched on English keywords in the lot title, so French-titled lots (for example at Christie's Paris) fall into Other. For the same reason, search in French, Italian, German and Chinese as well as English (see Search tips), and filter on the title text rather than `item_type` when the lot may not be in English.

## Search tips

Lot descriptions carry carat weights, origin, treatment and certificates ("Kashmir", "no indications of heating", "SSEF", "GIA"). `search_lots` requires every word, so use two or three short, distinct searches rather than one long one. Dedupe on `lot_url`.

**Search in every language the data holds, not only English.** The data mixes English, French (Christie's Paris, Geneva), Italian and German (Geneva, Milan, Zurich) and Chinese (Sotheby's Hong Kong titles, Bonhams Hong Kong titles and descriptions, Christie's and Phillips Hong Kong). English-only searches silently miss these lots, and `item_type` is wrong for French-titled lots (rule 12). For every concept in a query, run the English term and its equivalents below, then merge and dedupe on `lot_url`. For `query_table`, combine them in one `or=(...)` filter, for example `or=(lot_title.ilike.*tourmaline*,lot_title.ilike.*碧璽*,lot_title.ilike.*碧玺*,lot_title.ilike.*tormalina*,lot_title.ilike.*turmalin*)`.

| Concept | French | Italian | German | Chinese (traditional / simplified) |
|---|---|---|---|---|
| Tourmaline | tourmaline | tormalina | Turmalin | 碧璽 / 碧玺 |
| Paraíba | paraïba, paraiba | paraiba | Paraiba | 帕拉伊巴 |
| Rubellite | rubellite | rubellite | Rubellit | 紅色碧璽 / 红色碧玺 |
| Ring | bague | anello | Ring | 戒指 |
| Diamond | diamant | diamante | Diamant | 鑽石 / 钻石 |
| Sapphire | saphir | zaffiro | Saphir | 藍寶石 / 蓝宝石 |
| Ruby | rubis | rubino | Rubin | 紅寶石 / 红宝石 |
| Emerald | émeraude | smeraldo | Smaragd | 祖母綠 / 祖母绿 |
| Pearl | perle | perla | Perle | 珍珠 |
| Carat (weight) | carats | carati | Karat | 克拉 |
| Unheated | non chauffé, sans traitement thermique | non riscaldato | unbehandelt, nicht erhitzt | 未經加熱 / 未经加热 |
| Signed | signé | firmato | signiert | 簽名 |
| Case / box | écrin, boîte | astuccio, scatola | Etui, Schachtel | 盒 |

- This table is a starting point. Spot-check a new term with `limit=1` (the `totalRows` count) before relying on it, and add terms you find in real titles (other stones, makers' Chinese names, origins such as 緬甸 Burma, 克什米爾 or 喀什米爾 Kashmir, 哥倫比亞 Colombia, 莫桑比克 Mozambique, 巴西 Brazil).
- Maker names are often transliterated in Chinese titles (卡地亞 Cartier, 蒂芙尼 Tiffany, 寶格麗 Bulgari, 梵克雅寶 Van Cleef & Arpels, 海瑞溫斯頓 Harry Winston). Search the Latin name and the Chinese name.
- Sotheby's Hong Kong titles usually give the weight in the title ("4.14克拉"); read it there before fetching the description.
- **Accents.** `ilike` is accent-sensitive. "Paraíba" and "Paraiba" are different strings, and a search for one can return nothing for the other. Search the stem ("*para*ba*") or both spellings, and search `lot_title` as well as `lot_description`.
- **Noise.** Match the stone in `lot_title` where you can, and exclude multi-piece lots ("group of rings", "demi-parure", "two rings"), watch-and-ring pairs, and lots where the stone is only a side stone mentioned in the description. Do not rely on `item_type=Ring` alone.
- **Made-by and cased flags.** Treat a lot as made by a maker only when the lot is signed or named in the title or description (for example "signed Tiffany & Co."); "with maker's mark" without a name is an unnamed maker, not a made-by. Treat a lot as cased only when the description says the maker's case, box or pouch accompanies it ("signed box", "accompanied by a signed case"). A bare keyword match on "case" is not enough (it also matches "encased"); read the description before flagging, and say "not checked" for lots you did not read.
- **Keep payloads small.** Use `limit` of about 40, always pass `select`, and fetch `lot_description` only for the shortlist. `bard_home` returns a large payload; when you need only `fx` and `meta`, read those and ignore the rest.

## More detail

`references/known-issues.md` lists known data gaps, their measured size and the date they were checked, plus the changes made to the database. Read it when a figure looks wrong or before publishing market share.
