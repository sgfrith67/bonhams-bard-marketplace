---
name: market-share-brief
description: >
  This skill should be used when a Bonhams manager, director or specialist asks how
  Bonhams is performing against Christie's, Sotheby's and Phillips in jewellery auctions:
  "market share", "how are we doing", "Bonhams versus competitors", "jewellery market
  update", "management brief", "year to date", "regional performance", "are we losing
  share", or "prepare a briefing on the jewellery market". Produces a short, answer-first
  brief from BARD.
metadata:
  version: "0.2.0"
---

# Market share brief

Produce a one-page, answer-first brief on Bonhams' jewellery auction performance against the other three houses. Load the `bard-data` skill first for conversion rules and caveats.

## 1. Gather

Call `bard_home` once. It contains everything needed: `fx`, `meta`, `fy` (per house, year and live or online: sales, lots offered, lots sold, unknown-status lots, value), `reg`, `ytd`, `trend`, `conc`, `est`, `recent`. Use `query_table` only to answer a specific follow-up. Note `generated_at` and `cur_fy`/`fy_day` (days into the current financial year).

## 2. Check what can be trusted before writing

- For each house, compute days from `meta` latest sale to `generated_at`. Over 60 days means its current-year figures are incomplete: exclude that house from year-to-date comparisons and say so up front.
- Read each `meta` caveat. Sotheby's sell-through is only valid where `sell_through_reliable` is true. Competitors report buyer's total only.
- Market share is calculated on **live** auctions only, for **completed** financial years. The current year is a year-to-date comparison against the same day last year.
- The aggregates count dedicated jewellery sales only (English, French, Italian and German names, including Christie's "Joaillerie Paris" since 4 October 2026). Mixed jewels-and-watches sales and single-owner collection sales are excluded. Mention this in one line whenever a share figure is quoted (see `bard-data/references/known-issues.md`).

## 3. Write the brief

Use this structure, answer first. Quote values in USD with an AUD equivalent (state the rate date once), and name the financial year as FY26 = July 2025 to June 2026.

**Headline (one or two sentences).** The single most important message, with its number. Example shape: share held or lost, and the main reason.

**Three supporting points**, each one sentence of claim followed by the evidence:

1. **Share and rank.** Bonhams' live value in the last completed year, change on the prior year, share of the four houses' live value, and rank.
2. **Momentum.** Year to date against the same day last year, live and online, using only houses with current coverage. Add Bonhams' sell-through (sold divided by offered) for the last two years.
3. **Where it comes from.** The regions where Bonhams gained or lost share, and reliance on top lots (share of value from the top 10 percent of lots and from the single top lot). Add the item types where estimates are landing furthest from results, from `est`, when it adds a reason.

**Data caveats.** Stale houses with dates, mixed and collection sales excluded, hammer versus total, and the currency basis (one latest rate for every period, so year-on-year excludes currency).

**Suggested next steps.** Two or three concrete actions, for example a competitor capture gap to chase or an item type whose estimates need review.

Keep it to one page. Do not add a methodology section; the caveats cover it. Do not describe a competitor's missing data as a market fall.

## 4. Delivery

Give the brief in the reply. If the person wants to keep or share it, make a document, and offer to build the BARD home page (the `bard-home` skill) for the live version.
