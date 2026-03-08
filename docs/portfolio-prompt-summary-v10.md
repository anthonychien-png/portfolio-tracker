# Portfolio App — Prompt History Summary
## Session 10: Charts, FX History & Accuracy (March 8, 2026)

This session added several chart enhancements and a significant accuracy improvement to the portfolio app — replacing the static live FX rate with a full historical daily USD/CAD rate series so that all calculations reflect the exchange rate that was actually in effect on each specific date.

---

### Phase 1 — Ticker Allocation Chart Enhancements

**Prompt 1**
> "Please include the number of shares purchased, as well as the current market value, for all the charts."

Added a dedicated `mkDonutTicker()` function for the Holdings Allocation by Stock/Fund pie charts (member and family views). Hovering a slice now shows: ticker name, total shares held, current market value, and portfolio percentage. Multiple lots of the same ticker are aggregated. Other allocation charts (institution, account type, currency, member) were left unchanged since shares are not a meaningful dimension for those groupings.

---

### Phase 2 — Annual Return by Year Bar Chart

**Prompt 2**
> "Create a bar chart for each member view and family view that shows the total return for that specific year. Please use the closing stock price on the last business day of the year."

Added an **Annual Return by Year (%)** bar chart to each member view and the family view. Calculation logic:

- For stocks **held from a prior year**: year-start value = last trading close of Dec 31 of the prior year × shares; year-end value = last trading close of Dec 31 of that year × shares.
- For stocks **purchased mid-year**: year-start value = cost basis (purchase price × shares).
- For the **current year in progress**: year-end value = today's live price.

Bar colours are green for positive years, red for negative. Tooltip shows return %, P&L in CAD, start value, and end value. Bars are sorted chronologically. Chart appears in the last row of each member view and the family view.

---

### Phase 3 — Dollar P&L Labels on Bar Charts

**Prompt 3**
> "Include the dollar amount in CAD, as well as the %, on the Return by Member bar chart."

Created a new `mkBarWithPnl()` function that renders both the return % (bold, green/red) and dollar P&L (smaller, grey) as inline labels directly on each bar, with extra top padding to prevent clipping. Tooltip also shows both lines. Applied to the **Return by Member** family chart.

**Prompt 4**
> "Include the dollar amount in CAD, as well as the %, on the Return by Holding bar chart for each member view. Include the dollar amount in CAD, as well as the %, on the Return by Account Type bar chart under the Family view."

Extended `mkBarWithPnl()` to the **Return by Holding** (member views) and **Return by Account Type** (family view) charts. P&L for account type is calculated as `totalValue − totalCost` per group.

**Prompt 5**
> "Remove the reference to dates on the X-axis on the Return by Holding bar chart under the Member View. Show only the % and dollar amount in CAD by individual stock or mutual fund."

Removed `(purchaseDate)` from x-axis labels on the Return by Holding chart. Multiple lots of the same ticker are now aggregated into a single bar showing the combined return and P&L across all purchase dates.

---

### Phase 4 — Historical FX Rate for All Calculations

**Prompt 6**
> "When calculating the portfolio value (Annual Return by Year, Return by Holding, Return by Member, Net Worth Over Time) for each member view and family view, please take into account the closing value of the CDN vs. USD exchange rate for that specific day. For example, when calculating the Annual Return by Year in % and dollar terms, please take into the closing value of the CDN vs. USD exchange rate for the last business day of that year."

This was the most significant change of the session. Previously, all historical calculations used the **current live USD/CAD rate** for every date — meaning a 2023 year-end value was being converted at today's 2026 rate, which is incorrect.

**Solution:** On app startup, the app now fetches 3 years of daily USD/CAD closing rates from the Frankfurter.app free API (no key required, CORS-safe), storing them in a `fxHistory` map keyed by `YYYY-MM-DD`. A new `fxRateForDate(dk)` function looks up the exact date, and if missing (weekend/holiday), walks backward up to 5 calendar days to find the nearest prior business day's rate. Falls back to live rate if history hasn't loaded.

A new `toStockCurAt(amount, fromCur, toCur, dk)` function replaced `toStockCur()` everywhere historical accuracy matters:

| Calculation | FX rate now used |
|---|---|
| Net Worth Over Time | Each bar uses its own calendar date's closing rate |
| Annual Return by Year | Year-end uses Dec 31 rate; year-start uses prior Dec 31 rate; mid-year purchases use their purchase-date rate |
| Return by Holding | Cost basis uses purchase-date rate; current value uses today's rate |
| Daily / Monthly / YTD | Each reference price uses its specific reference date's rate |

---

### Phase 5 — This Prompt Summary

**Prompt 7**
> "Summarize the prompt to .md and .docx file format."

Generated this document in both Markdown and Word (.docx) formats.

---

## Cumulative Feature List (All Sessions)

| Session | Key Features |
|---|---|
| V1 | Single-member portfolio tracker, Yahoo Finance prices, basic HTML |
| V2 | Multi-member family views, consolidated family tab |
| V3 | Financial institutions (RBC, CIBC, TD, WealthSimple), Canadian account types (RRSP, TFSA, Non-Reg) |
| V4 | CAD/USD dual currency, live FX rate, per-account currency tracking |
| V5 | Light mode UI, 60-second auto-refresh, Net Worth Over Time charts |
| V6 | ROI chart filters (member, institution, account type, ticker), holdings table ticker filter |
| V7 | Net Worth dollar charts replacing ROI%, CA$/US$ currency toggles on charts |
| V8 | CAGR fix for sub-year holdings, price chip ticker strip, auto-refresh fix |
| V9 | Split-adjusted price chip fix, FX fetch 4-source fallback, Cloudflare Worker CORS proxy, ticker allocation pie charts, iOS/GitHub Pages guidance |
| V10 | Shares + value on pie tooltips, Annual Return by Year bar chart, P&L labels on bar charts, ticker aggregation on Return by Holding, historical daily FX rates for all calculations |
