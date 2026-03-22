# Portfolio App — Prompt History Summary
## Session 12: Price Accuracy, After-Hours, & Unlimited Members (March 22, 2026)

This session focused on four areas: fixing return calculation accuracy (YTD, Monthly, Daily), adding after-hours/pre-market price display, resolving Cloudflare Worker and GitHub Pages hosting issues, and enabling unlimited family members with a clean setup flow.

---

### Phase 1 — Return Calculation Fixes

**Prompt 1**
> "Can you check if the calculations for YTD is correct? For example, PLTR YTD return is around negative 7%, but it should be closer to negative 12%."

Root cause: YTD baseline was using the **first trading day of the new year** (Jan 2) instead of the **last close of Dec 31**. Code walked forward through history and took the first bar on or after Jan 1. Fixed by walking backward to find the last bar strictly before Jan 1 — the industry-standard Dec 31 close baseline.

**Prompt 2**
> "Could you double check if the calculations for Monthly return, daily return, and annualized return is also correct?"

Audit results:
- **Daily** ✓ — correctly uses `meta.regularMarketPreviousClose`
- **Monthly** ✗ — was using 22nd-to-last bar (drifts 2–5 days around holidays). Fixed to find the last bar at or before exactly 30 calendar days ago
- **Annualized** ✓ — CAGR for ≥1 year, simple annualisation for <1 year, both correct
- **YTD** — already fixed in Prompt 1

---

### Phase 2 — After-Hours Price Display

**Prompt 3**
> "Is there an after hours quote for stocks available from Yahoo API you can include into this app?"

Yahoo's v8 chart API returns `postMarketPrice`, `postMarketChange`, `preMarketPrice`, `preMarketChange`, and `marketState` in the `meta` object. Added these fields to `parseQuote` and updated the price chip to show:
- **After** badge (purple) with extended-hours price and change vs today's regular close
- **Pre** badge (amber) with pre-market price and change vs yesterday's close
- Reference line showing today's regular close for context

**Prompt 4**
> "The latest after hours prices are incorrect. NVDA should be $174.90, showing $172.70. PLTR should be $151.96, showing $150.68."

Root cause: `includePrePost=false` on the 5d quote URL was stripping all extended-hours data. Changed to `includePrePost=true`. Also added a separate `yahooExtUrl` (`interval=1m&range=1d&includePrePost=true`) fetched after the main quote, using the last bar from the 1m array as the actual latest extended-hours price rather than the often-stale `meta.postMarketPrice`.

**Prompt 5**
> "The $ and % price change for NVDA and PLTR from regular trading hours is still incorrect. NVDA should be -$5.86, -3.28%. PLTR should be -$5.00, -3.21%."

Root cause: Setting `includePrePost=true` on the 5d chart corrupted `meta.regularMarketChange` because Yahoo recalculates it using `chartPreviousClose` (a stale/split-adjusted value) instead of the actual prior session close. Fixed by reverting the 5d quote URL to `includePrePost=false` and keeping the 1m extended-hours fetch separate.

**Prompt 6**
> "The change in price and % change is still incorrect."

Root cause: `meta.postMarketPrice` in Yahoo's response is stale from a prior session (e.g. NVDA returning 180.25 instead of 174.90). Fixed `parseExtQuote` to read the **last bar** from the 1m array and validate its timestamp is within 24 hours, completely ignoring `meta.postMarketPrice`.

**Prompt 7**
> "Still showing incorrect daily change. Is there another free API available?"

Root cause identified via arithmetic: `172.70 − 180.25 = −7.55` matching the wrong number, confirming `chartPreviousClose = 180.25` (wrong) vs actual `178.56`. Fixed `parseQuote` to derive both `cur` and `prev` from the **5d bars array** (two most recent distinct-date closes), bypassing all meta fields.

**Prompt 8**
> "Now showing US$0.00 (+0.00%). Tool should use latest business day's closing price on weekends/holidays."

Root cause: On weekends, `meta.regularMarketPreviousClose` equals `meta.regularMarketPrice` — Yahoo sets both to the same last close. Fixed by grouping all 5d bars by calendar date using UTC timestamps, building an ordered array of distinct trading dates, and taking `dateOrder[-1]` (last business day) and `dateOrder[-2]` (business day before that).

---

### Phase 3 — Proxy Worker & Hosting Debugging

**Prompt 9**
> "The price chip is still showing US$0.00 (+0.00%) for both NVDA and PLTR."

Added visible debug info to each price chip (`bars:N last:XXX prev:YYY`) and bumped the version string to force browser cache invalidation. Confirmed the browser was serving cached JS from GitHub Pages.

**Prompt 10**
> "Are the graphs and charts using the latest closing price from regular trading hours or after hours?"

Confirmed: all charts and portfolio calculations use `pd.current` and `pd.history` — regular session closes only. After-hours price (`pd.postMarketPrice`) is only used in the price chip strip, never in portfolio values, returns, or chart calculations.

---

### Phase 4 — Unlimited Members

**Prompt 11**
> "Provide the ability to add unlimited members. Only show members that have been added, instead of all four members on the initial form."

Major restructuring:
- `state.members` now starts as an empty array `[]` instead of 4 hardcoded members
- `state.holdings` starts as `{}` keyed by dynamic IDs
- `MEMBER_HEX` extended to 16 colours with a `memberColor(i)` helper that cycles for any number of members
- `buildSetupPanel` rewritten: shows only existing members + a dashed **"+ Add Member"** card
- `addMember()` — generates unique ID via `Date.now()`, focuses new name input
- `removeMember(mid)` — confirms, removes member and holdings, cleans up panel/tab
- `clearAll` resets to 0 members
- `loadState` guards missing holdings keys
- Startup shows setup panel when no members exist
- Fixed JS syntax errors in quote escaping (`''` → `\\'`, `==='Enter'` → `=="Enter"`)

**Prompt 12**
> "Encountered SyntaxError: Unexpected string"

Fixed two quote-escaping bugs introduced in the new `buildSetupPanel`:
1. `removeMember(''+m.id+'')` → correct escaping `removeMember(\\'' + m.id + '\\')`
2. `event.key==='Enter'` inside HTML attribute → `event.key==&quot;Enter&quot;`

**Prompt 13**
> "Please remove Tony, Joanne, Magnus, and Prisca as members. The form should start with no members."

Explanation: members are stored in browser localStorage, not in the code. The app code already starts with no members for new users. User chose "Please do nothing" — no code changes made.

**Prompt 14**
> "Only show team member(s) specified on all forms. Hide Member 2, 3, 4 if only one is set up."

Added migration logic to `loadState` and `saveMembers` that automatically strips members whose name still matches the default `Member N` pattern AND who have no holdings. On next load, old saves with unused `m1/m2/m3/m4` placeholder members are silently pruned, keeping only real named members or default-named members who have holdings.

---

### Phase 5 — This Session Summary

**Prompt 15**
> "Summarize all prompts"

Generated this document in both Markdown and Word (.docx) formats.

---

## Cumulative Feature List — All Sessions

| Session | Key Features |
|---|---|
| V1 | Single-member portfolio tracker, Yahoo Finance prices, basic HTML |
| V2 | Multi-member family views, consolidated family tab |
| V3 | Financial institutions (RBC, CIBC, TD, WealthSimple), RRSP/TFSA/Non-Reg account types |
| V4 | CAD/USD dual currency, live FX rate, per-account currency tracking |
| V5 | Light mode UI, 60-second auto-refresh, Net Worth Over Time charts |
| V6 | ROI chart filters (member, institution, account type, ticker) |
| V7 | Net Worth dollar charts, CA$/US$ currency toggles on all charts |
| V8 | CAGR fix for sub-year holdings, price chip ticker strip, auto-refresh fix |
| V9 | Split-adjusted price chip fix, FX 4-source fallback, Cloudflare Worker proxy, ticker pie charts |
| V10 | Shares + value on pie tooltips, Annual Return by Year chart, P&L labels on bar charts, historical FX rates |
| V11 | RESP/FHSA, Buy/Sell transactions, sell-aware calculations, Canadian mutual fund manual NAV, Worker hardening, file:// detection |
| V12 | YTD/Monthly/Daily return accuracy fixes, after-hours/pre-market price chips, business-day-aware daily change, unlimited members with dynamic add/remove |
