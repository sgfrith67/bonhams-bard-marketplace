---
name: lot-estimate-guide
description: >
  This skill should be used when a jewellery specialist wants an auction estimate or
  comparable sales for a piece or gemstone: "estimate this piece", "what should I estimate
  this at", "comparable sales for", "lot estimate for", "what's this worth at auction",
  "find comps", or when they describe a piece (carat weight, brand, period, condition) or
  paste a condition report or lot description and want pricing context. Trigger on a
  detailed piece description even if the word "estimate" is not used.
metadata:
  version: "0.2.0"
---

# Lot estimate guide

Assist a Bonhams jewellery specialist in researching comparable sales and proposing a reasoned auction estimate. The output is an **auction estimate** (the expected hammer range at a competitive international sale), never an insurance or replacement valuation. Say so if asked.

Load the `bard-data` skill first for tool usage, currency conversion and the hammer versus buyer's total rule.

## 1. Understand the piece

Gather what the specialist gives: stone type, carat weight, cut, colour, clarity, origin and treatment; metal; maker or signature; period and style; condition; item type; certificates (GIA, Gübelin, SSEF) present or absent; provenance, box and papers. If a decisive detail is missing (stone weight, or signature on a high-value piece), ask one focused question before searching. Otherwise proceed and state assumptions.

The weight may come as a range (for example "3–4 ct" or "up to 2.5 ct") from the BARD home page. Treat a range as a filter: keep only comparables whose main-stone weight, read from the title or `lot_description`, falls inside it, show each comparable's weight, and widen the range only if fewer than three comparables remain, saying so.

## 2. Check the data first

Call `bard_home` once. Read `fx` (for conversion) and `meta` (latest sale per house). If a house's latest sale is more than 60 days old, note that its recent results are missing and keep that in mind when judging market conditions.

## 3. Find comparables in BARD

Run two or three differently-worded searches and merge them, deduping on `object_id` or `lot_url`:

- `search_lots` with brand plus type, then stone plus weight plus origin or treatment, then period or style (`max_results` about 25).
- `query_table` on `bard_lots` for precision: `is_sold=eq.true`, `sale_date=gte.<three years back>`, `lot_title=ilike.*<brand or stone>*` (or `lot_description=ilike.*<origin or weight>*`), optionally `auction_house=in.(...)` or `item_type=eq.<type>`, ordered by `sold_usd.desc`, with `select` limited to house, sale name, date, title, currency, estimates, `hammer_price`, `sold_price`, `sold_usd`, `sold_aud`, `hammer_aud` and `lot_url`. Fetch `lot_description` only for the shortlisted lots, to confirm weight, origin, treatment and certificates.

Prefer the last three years. Aim for 4 to 8 strong comparables, weighted to the closest match on stone weight and quality, signature, period and condition. Include unsold lots only as context (they show where demand stopped), never as price evidence.

Where BARD returns fewer than three usable comparables, or a house relevant to the piece has stale coverage, consult that house's own results page (and only these four: Bonhams, Phillips, Christie's, Sotheby's). Do not use retail sites, news, trade publications or any other auction house, because they mix in retail or insurance values. Results pages:

| House | URL |
|---|---|
| Bonhams | https://www.bonhams.com/auctions/results/?categories=Handbags%2C+Jewels+%26+Watches |
| Phillips | https://www.phillips.com/calendar/results?Departments=Jewels%2CWatches |
| Christie's | https://www.christies.com/en/results |
| Sotheby's | https://www.sothebys.com/en/results |

BARD covers jewellery sales only. For watches, use the house pages above and apply the watch factors in `references/pricing-factors.md`.

## 4. Put prices on a like-for-like basis

- Show each result in its **original currency and in AUD** (default; use another currency if the specialist names one), using `sold_aud` and `hammer_aud` from `bard_lots` (or the `bard_home` rates for `search_lots` results). State the rate date once.
- Bonhams comparables: use the **hammer** for estimate-setting and show the buyer's total too.
- Christie's, Sotheby's and Phillips comparables: only the buyer's total exists. For estimate-setting, divide by 1.28 as a rough hammer-equivalent and label that column approximate. Rank comparables on buyer's total versus buyer's total when comparing among competitors.

## 5. Build the estimate

1. Judge the **expected hammer** from the closest comparables, adjusting for the factors in `references/pricing-factors.md` and for how recent each result is.
2. Set the range using Bonhams' own habits (reference ratios in `bard-data/references/known-issues.md`, section 5): the high estimate is typically about 1.5 times the low, and median hammer lands at about 1.33 times the low estimate below 5k, 1.2 times at 5 to 20k, 1.12 times at 20 to 100k and 1.1 times above 100k. So the low estimate usually sits below the expected hammer by that factor.
3. Cross-check with Bonhams' recent record for the item type from `bard_home` (`est`: rows of `[fy, type, lots, below, within, above, median hammer ÷ low]`). If the type has been hammering below the low estimate more than a third of the time, say so and lean the range lower. Loose stones are the usual example.
4. Cross-check the market: compare the most recent comparables with older ones for the same kind of piece. Report demand as stronger, softer or mixed on that evidence only.

## 6. Output

Lead with the answer. Use this order:

**Suggested estimate:** low to high in AUD and in the sale currency, with a one-sentence rationale. **Confidence:** High, Medium or Low.

- High: 5 or more strong comparables with similar stone quality and signature.
- Medium: 3 to 4 comparables, or some differences in weight or condition.
- Low: fewer than 3, or comparables differ materially.

Then the support:

**Comparable sales**

| # | House | Sale date | Description | Result (original) | Basis | Result (AUD) | Pre-sale estimate | Link |
|---|---|---|---|---|---|---|---|---|

Basis is "Hammer" or "Total" (or "Total ÷ 1.28" where a hammer-equivalent is shown). Add one line per comparable on why it is relevant and how it differs from the subject piece.

**Market conditions:** one or two sentences from the comparables and the Bonhams estimate record.

**Caveats:** missing certificate, untested stones, condition issues, currency movement on cross-region comparables, stale competitor coverage, and the approximate hammer-equivalent for competitor results.

## Do not

- Quote a figure that is not traceable to a comparable or a stated ratio.
- Mix currencies in a ranking.
- Present a competitor's buyer's total as a hammer.
