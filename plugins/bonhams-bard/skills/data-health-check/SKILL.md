---
name: data-health-check
description: >
  This skill should be used when someone asks whether BARD data is complete, current or
  trustworthy: "check the BARD data", "is the data up to date", "data health check",
  "are the scrapers working", "why is Phillips missing", "why are Christie's numbers
  low", "is anything missing from BARD", or before sharing a market-share figure or
  publishing the BARD home page. Compares what BARD holds with each house's own sale
  calendar and reports gaps for the data owner.
metadata:
  version: "0.2.0"
---

# Data health check

Find out whether BARD is missing sales, mis-scoped sales or bad fields, and report it so it can be fixed. Load the `bard-data` skill first, and read `bard-data/references/known-issues.md` for the previous findings to re-test rather than assume.

## 1. Coverage by house

Call `bard_home` and read `meta`: `[house, latest_sale_date, sell_through_reliable, caveat, dedicated_live_sales]`. For each house compute days between `latest_sale_date` and today. Classify: current (under 30 days), watch (30 to 60), stale (over 60). Seasonality matters: a house may legitimately have no jewellery sales for weeks, so staleness is a prompt to check, not a finding.

## 2. Compare with each house's own calendar

For every house that is stale or on watch, and for Bonhams as a baseline, look up the house's published jewellery sale calendar and press releases on its own site (bonhams.com, christies.com, sothebys.com, phillips.com), covering the period after BARD's latest captured date up to today. List each completed jewellery sale with its date and name, including online editions.

Then test each one in BARD with `query_table` on the raw table `Auction_results` (so the test does not depend on scope rules): `auction_house=eq.<house>`, `auction_date=eq.<date>`, `limit=1`, `select=auction_number,auction_name`. `totalRows` of 0 means the sale is **missing from BARD**. Also test for the same date within a few days, since dates can differ by a day across time zones.

Report only sales that the house's own site shows as completed. Treat a future-dated sale as scheduled, not missing.

## 3. Scope check

The BARD aggregates count only `sale_scope = jewellery_sale`. Find jewellery that falls outside it: query `bard_lots` with `sale_scope=in.(mixed_sale,collection_sale)`, `is_jewellery=eq.true`, `is_sold=eq.true`, `sale_date=gte.<12 months ago>`, `order=sold_usd.desc`, `select=auction_house,auction_name,sale_date,sold_usd`, `limit=200`, and group the rows by sale. Report any sale that looks like a dedicated jewellery sale but was classed otherwise (often a new naming pattern or language), with its value. Compare with `known-issues.md`.

## 4. Field quality spot-checks

Use `totalRows` with `limit=1`:

- lots with unknown status: `status_norm=eq.unknown`, per house (table `bard_lots`)
- sold lots with no price: `is_sold=eq.true`, `sold_price=is.null`, per house
- lots with no currency: `currency=is.null`, per house
- lots where `est_low` exceeds `est_high`: not filterable directly, so sample the most recent sale instead

Compare against the previous counts in `known-issues.md` and report only changes and anything now above 2 percent of a house's lots.

## 5. Report

Lead with a one-line verdict ("BARD is current for Bonhams and Sotheby's; Phillips has not been captured since 10 June"), then a table:

| House | Latest captured | Days behind | Missing completed sales | Scope notes |
|---|---|---|---|---|

Then, for the data owner: the exact missing sales (house, name, date, link to the house page), any scope issue with its measured effect, and any field-quality changes. Say plainly what is confirmed (the sale exists on the house's site and not in BARD) and what is only suspected (a quiet calendar that may be seasonal).

State the effect on the numbers: which year-to-date figures and which share figures are understated, and by roughly how much where it can be measured. Do not apply fixes to the database; recommend them. If this check changes the picture in `known-issues.md`, say which lines are out of date.
