# Known BARD data issues

Last reviewed **4 October 2026** against the live database. Re-check anything dated before quoting it as current; the `data-health-check` skill does this.

## 1. Competitor capture gaps (stale scrapers)

| House | Latest sale captured | What is known |
|---|---|---|
| Phillips | 10 Jun 2026 | **Confirmed gap.** Phillips held The Hong Kong Jewels Auction on 29 Sep 2026 (online sale 22–30 Sep), per its own press release, and BARD has neither. Its Geneva sale is scheduled for 9 Nov 2026. |
| Christie's | 26 Jun 2026 (Joaillerie Paris) | **Not yet proven.** Christie's captured jewellery calendar is thin from July to September in earlier years (flagship sales run October to December), so a quiet autumn start is normal. The test is whether its November and December Geneva, London and Paris sales appear. One soft signal: online jewels sales were captured in September 2023 and 2024 but not 2025. |
| Bonhams | 25 Sep 2026 | Current. |
| Sotheby's | 23 Sep 2026 | Current. |

Effect: year-to-date comparisons that include Phillips (and later Christie's) understate those houses. Their current-year share is not meaningful until capture resumes.

## 2. Scope rule (fixed 4 October 2026)

Until 4 October 2026 a sale counted as a dedicated jewellery sale only if its name contained "jewel", which dropped Christie's "Joaillerie Paris" (two sales a year, US$20–28M a year). The rule in `bard.lots_v` now also accepts "joaillerie", "bijoux", "gioielli" and "schmuck", and treats names that also mention leather goods or watches in French, Italian or German ("maroquinerie", "montres", "horlogerie", "orologi", "uhren") as mixed sales. The home cache was refreshed the same day.

Effect on Bonhams' live share (four-house total):

| FY | Before | After |
|---|---|---|
| FY24 | 8.45% | 8.23% |
| FY25 | 8.71% | 8.38% |
| FY26 | 7.86% | 7.65% |

Christie's FY26 live value rose from US$375.7M to US$397.5M. Bonhams' "Joaillerie et Maroquinerie" and "Bijoux, Montres & Maroquinerie" sales correctly remain mixed. One small 2021 Sotheby's single-owner sale ("Claude Lalanne : Bijoux et accessoires", about US$1M) is now counted; it sits outside the page's four-year window.

**Still excluded by design**, even where mostly jewellery: mixed jewels-and-watches sales such as Sotheby's "Fine Jewelry & Watches" (October; about 207 of 259 lots are jewellery) and "Precision & Brilliance: Prestigious Jewels & Watches" (December 2025, about US$25M), Christie's "Modern Icons: Jewels & Watches" (June 2026), and single-owner collections such as Christie's "A Treasured History: The Stream Family Collection" (June 2026, about US$13M). Disclose this when comparing houses.

## 3. Field-level quality

| Issue | Size | Handling |
|---|---|---|
| Sotheby's lots with no status | 23 | `status_norm` = unknown; exclude from sell-through. An earlier large unknown-status caveat has cleared. |
| Sotheby's sold lots with no price | 133 | Exclude from value statistics. |
| Sotheby's unsold lots carrying a price | 20 | Ignore the price. |
| Phillips lots with a zero price | 65 | Drop from price statistics. |
| Bonhams duplicate lot keys | 10 | Dedupe on house, auction number, lot number. |
| Christie's lots with no currency or low estimate | 4 | Converted prices are null; skip. |
| Hammer price | Bonhams only | See the hammer rule in SKILL.md. |
| Item type on French titles | Christie's Paris | Falls into Other. |
| `withdrawn`, `published` fields | unreliable | Do not use. |

## 4. Connector changes (4 October 2026)

- **Curated view for the connector.** `public.bard_lots` exposes the curated `bard.lots_v` (scope, live/online, region, FY, item type, normalised status) with prices converted to USD and AUD, so queries can rank across currencies. It is read-only and runs with the caller's rights.
- **Timeouts.** The connector runs as the database role `anon`, whose statement timeout was 3 seconds; cold queries with exact counts failed. It is now 10 seconds.
- **Indexes** added on `Auction_results` for house plus auction number, house plus status, house plus date, and trigram search on `lot_title`.

## 5. Reference ratios (Bonhams live jewellery sales, July 2024 to October 2026)

Use these when setting or sanity-checking estimates. Refresh them if more than a year old.

| Low estimate band | Median high ÷ low | Median hammer ÷ low | Hammered below low | Hammered above high |
|---|---|---|---|---|
| under 1k | 1.50 | 1.33 | 24% | 42% |
| 1k to 5k | 1.50 | 1.33 | 14% | 37% |
| 5k to 20k | 1.40 | 1.20 | 20% | 29% |
| 20k to 100k | 1.50 | 1.12 | 20% | 20% |
| 100k and over | 1.57 | 1.10 | 18% | 21% |

Bands are in sale currency (USD, GBP, HKD, EUR pooled). Buyer's total at Bonhams runs at about 1.28 times hammer.
