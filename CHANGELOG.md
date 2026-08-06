# Changelog

All notable changes to Theta Vault are documented here.  
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/), versioning follows [Semantic Versioning](https://semver.org/).

---

## [3.22.0] — 2026-08-06

### Added
- **The hold-to-expiry counterfactual (#298).** Every closed trade now carries what it *would* have
  made had it never been touched, and the portfolio carries the management edge that falls out of
  it. Audit §8.3 item 14 — "the highest learning value in the backlog." Two columns:
  `underlying_close_at_expiry` and `held_to_expiry_pnl`.
- **It is the fifth quantity off the one vertex scan.** Max loss reads `max(V)`, max profit
  `min(V)`, spread width `max(V)` per unit, POP the region where `netPremium − V(S) > 0`. The
  counterfactual reads `netPremium − V(S)` at a single S: the price the underlying actually closed
  at on expiration day. Nothing is modelled — no vol, no drift, no terminal distribution. A
  counterfactual built on a simulated price would be testing the simulation, not the decision.
- `heldToExpiryPnl` in `payoff.ts` and `held_to_expiry_pnl` in `import.py`, asserted against the
  same 17 numbers in both suites.
- `management_edge`, `management_edge_n` and `management_edge_win_rate` on `PerformanceStats`,
  averaged over the rows that HAVE a counterfactual.
- `TradeDetail` renders "Held to exp / Mgmt edge / (settled X)" on closed trades that carry one.

### Validation
- **The engine was tested against reality before it wrote anything.** 45 trades in this book were
  genuinely held to expiry. The vertex payoff reproduces their realized P/L on **24 of 24
  non-index rows** and on **0 of 17 SPX rows**. Of the rows ultimately written, **11 of 11**
  expired rows reproduce realized P/L to **$0.00**.
- **That agreement is weaker than it reads, and #303 records why.** Trades held to expiry are
  self-selected — you let them run when they are working — so **10 of those 11 landed outside all
  strikes**, where the expiry payoff is a flat constant and the fetched price could be wrong by a
  wide margin without failing the test. Only trade 1235 landed inside the strike range. The
  sloped region of the payoff is covered by 36 hand-derived fixture assertions across both
  languages, but those are synthetic; the real-book test mostly validates the credit arithmetic
  and the NULL gates. Surfaced by an advisor review at the commitment boundary, not by the suite.
- Second backfill pass wrote **0** rows. ATTACH diff against the pre-run snapshot: only the two
  new columns differ, no shared column moved, 459 = 459.

### Data
- 262 of 449 closed trades carry a counterfactual. 187 are gated, by five gates each derived from
  a probe rather than a spec: index 92 (#299), multi-expiry 34, not-yet-expired 28, anchor 12
  (#300), equity-leg 10, leg/header credit 8 (#302), futures 3.
- **On this book: realized −$3,331.80 against −$51,978.07 if held — a management edge of
  +$48,646.27.** Managing helped on 117 trades (+$71,678), hurt on 130 (−$23,032), and was moot on
  15. It loses more often than it wins and wins far bigger when it does. Robust to outliers:
  dropping the five largest contributors still leaves +$37,369 over 257 trades.
- The metric is descriptive, not prescriptive. Its sample is the set of trades Chris's own rules
  selected for early exit, so it measures what management DID, not what more of it would do.

### Fixed
- **`_price_cache` was keyed on ticker alone and returned on any hit**, so a caller requesting a
  window the first caller never fetched received a dict silently lacking its dates — and
  `get_price_on_date` then walked back up to five sessions and handed over an older price as if it
  were the requested one. Every other price consumer ranges over entry..exit; an expiration date
  sits *after* the exit on every trade closed early, so the expiry lookup was exactly that caller.
  The cache now stores its covered range and widens to the union on a partial hit.
- `_differs` hoisted to module scope so every backfill shares one answer about whether a
  recomputed column actually moved (#288).

### Known
- **#299 — 17 of 17 SPX rows fail the identity, so index products are gated out entirely.** The
  residual does not resolve to one convention: 1543 and 1568 reproduce `pnl` at the *previous*
  session's close, 1626 at the stored date, 1673 only against `pnl + commission`, and five match
  nothing within five sessions. Trade 1543 is the clean example — short 7090/7100 call vertical,
  SPX closed at 7137.90 on the stored expiration, and the book records the full credit kept.
- **#300 — Yahoo serves split-adjusted history.** CRWD split 4:1; 10 rows store pre-split strikes
  against a series that now reads 111–148. The counterfactual defends itself with an anchor gate
  (Yahoo's close on the entry date must reproduce the stored spot within 2%); `mfe`, `mae` and
  `underlying_price_at_close` use the same path with no anchor and would be overwritten with
  split-adjusted values by a forced re-run.
- **#301 — the open/close commission split is discarded at write time**, so the counterfactual is
  charged the full round trip. On early-closed rows that overstates its cost by $1–8 against an
  edge measured in hundreds. A known bias rather than an unknown one.
- **#302 — 8 rows store leg premiums that contradict `credit_received`** (trade 1650: legs net
  11.77, header says 0.25). All 8 are SPX rolls.
- **#303 — the real-book ground truth barely exercises the price-sensitive payoff region** (see
  Validation above). 102 closed trades reached their expiration date, not 45; the 3 `assigned`
  rows are free ground truth in the sloped region and were left on the table.

---

## [3.21.0] — 2026-08-06

### Fixed
- **Stored POP came from three clamped delta heuristics and was wrong for every non-directional structure (#294).** `pop` was written by `(1 − |Δ|) × 100` on the insert path, by `(1 − putΔ − callΔ) × 100` in a branch keyed on the broker's `spreadType` string, and by `(1 − 2|Δ|) × 100` in `backfill_greeks` — whichever writer happened to touch the row. All three clamped to `[10, 95]`, which put **156 of 459 rows on exactly 95.0 or 10.0**. Trade 1652 (PLTR long put, Δ −0.0002) stored **95.0**. Trade 1494 (SPX long butterfly) stored **94.62**, and that is the number that multiplied into the **+$9,178 expected value against $300 of real risk** in audit §C3. After the recompute, 1494 reads **11.81** and 1652 reads **0.01**; the clamp artifacts are gone.

### Changed
- **POP is now the profit region read off the payoff vertices (#294).** `P(S) = netPremium − V(S)` is piecewise linear with kinks only at strikes, so the set where it is positive is a union of intervals whose endpoints are the breakevens — each one a linear interpolation between two adjacent vertices. POP is the risk-neutral measure of that set: `Σ [N(d2(a)) − N(d2(b))]`. The upside tail uses the terminal slope `−callSlope`; `S = 0` is already a vertex, so the downside needs no special case, and `K = 0` maps to probability 1 rather than a `log(0)` blow-up. One engine, no strategy labels, mirrored in TypeScript and Python and asserted to agree to 0.01pp.
- **`spread_width`, `cached_max_profit`, `cached_max_loss`, `credit_width_ratio` and now `pop` all come off one vertex scan.** The scan gained `vertices` and `valueAt` so callers can read the payoff *between* the extrema rather than only at them.
- The three delta formulas are marked `@deprecated` in place. They still run so the column is never empty mid-import; the reconciliation pass overwrites them at the end of every run.

### Data
- Reconciliation pass rewrote `pop` on **456** rows. Second pass reported 0. ATTACH diff against the pre-run snapshot: **`pop` is the only column that changed**, 0 rows added or deleted.
- Populated **340 of 459**, down from 456 — because the previous number was populated where it had no business being. The 119 NULLs are: **53** 0-DTE trades (entered and expiring the same day, so the terminal distribution is a point mass — #296), **48** calendars and diagonals with no single-expiry payoff, **15** covered calls whose capping shares are not in `legs_json`, **2** with no stored spot, **1** with no IV.
- The distribution is now shaped like a premium seller's book — 28 rows below 20%, 37 in 20–40, 76 in 40–60, 135 in 60–80, 64 above 80 — rather than 156 rows pinned to a clamp.

- **The `pop <= 1 ? × 100` scale guess is gone from all seven stored-column readers.** The column is contractually 0–100; the guess only worked while the old clamp kept every value above 10. Four rows now store a POP at or below 1.0 (trades 1531, 1540, 1600, 1652), and trade 1652's genuine **0.01%** would have rendered as **1%** — a hundredfold overstatement on the very row that demonstrates the fix. Removed from `compliance.ts` (2), `DashboardTab.tsx` (3), `TradeDetail.tsx`, `PayoffChart.tsx`.
- **`portfolio.ts` averaged a missing POP as zero.** With 3 NULLs that was a rounding error; with 119 it would drag the portfolio average toward zero with every calendar, covered call and 0-DTE row. `avg_pop` now averages over the positions that have a POP. A missing probability is unknown, not zero.

### Validation
- The engine agrees with `estimatePop` to **0.00pp on 313 of 327** rows — every iron condor (123), every cash-secured put (41), every butterfly and broken-wing butterfly (24). Agreement on the structures where that function's premise holds is what licenses trusting it where the premise fails.
- All **14** disagreements are single-breakeven structures where `estimatePop` guesses direction from `be < spot`, and in every one the gap is exactly `100 − region`: it returns the complementary side. Six short put verticals, three short call verticals, two long calls, one long put, one long put vertical, one ZEBRA. Trade 1652 is the extreme: region **0.01**, `estimatePop` **99.99**.

### Known
- **`estimatePop` still inverts outside-profit structures on the live path (#295, open).** Given two breakevens it always assumes profit lies between them; a 95/105 long strangle at spot 100 returns **0.00** where the true answer is **30.50**. The stored column no longer depends on it, but the UI's live-Greeks path still calls it.
- **55 of 457 rows store an `underlying_price` that contradicts their own fills (#297, open).** Detection is cheap and needs no external data: on those rows a leg's *entry premium* is below its own intrinsic value at the stored spot, which no one sells. Trade 1589 (MU) stores spot 524.56 against a short 420 call filled at **17.14** when intrinsic there would be **104.56**. This corrects a claim made in this project's own commit message for 1589: its POP of 1.92 is a correct function of a **bad input**, not a correct probability. The engine is right; the spot is not. The same rows are, unsurprisingly, most of the largest POP movements — so some of the apparent improvement in this release is measuring bad spot data more faithfully, not better.
- **POP inherits every modelling bias already in the app.** Drift is `r`, not `r − q`, so dividend payers (MCD, T, IBM, JPM, GLD) have call-side probability overstated and put-side understated (§8.4 item 30, unfixed). One `implied_volatility` per position means a skewed structure — precisely the butterflies and jade lizards this work was built for — is integrated at a flat vol. And POP is a *terminal* measure: early assignment on a deep-ITM short is not modelled, which is exactly the situation in the #297 rows.
- POP buckets do not cleanly predict realized win rates in this book, and that is expected rather than a defect — trades are actively managed at profit targets and at 21 DTE, so realized outcomes are not expiry outcomes. The one signal that does survive: the old POP's bottom bucket won 50% and its top bucket 66% (no discrimination), while the new POP's bottom bucket wins 37.9% against a 66% baseline.

---

## [3.20.1] — 2026-08-06

### Fixed
- **The `width_25` and `width_50` exit rules were false for every imported trade (#290).** `evaluateExitRule` answered both with `trade.closed_at_target === true`. The importer ALTERs that column in with `DEFAULT 0` and never writes it — only the trade form ever ticks it by hand — so it is `1` on **0 of 459 rows**. Selecting either rule on a playbook scored every closed trade in it as "Exit Rule missed" no matter how the trade was actually managed. The rules now evaluate realized P/L against `pct% × spread_width × multiplier × quantity`, which became possible when `spread_width` reached 347 rows in 3.20.0. `closed_at_target` is still honoured as a manual override, but it is no longer the whole rule.
- **The Dashboard printed a width-based GTC target from a different width than the rule is graded against (#291).** `effectiveWidth` spanned the outermost long strikes, which is neither wing of a broken-wing butterfly: on trade 1765 (longs 740 / 725, shorts 735×2) that span is 15 where the risk width is 5. The GTC column pointed three times further out than the exit rule the compliance checker measures. It now reads `spread_width` — the same canonical width, `max(V)` per unit off the payoff vertices — and falls through to the profit-target branch when the width is unknown.
- **A missed width rule now reports progress in the rule's own units (#290).** The message read "missed (closed at 86% max)", which sounds like a near-miss on a trade that reached 6% of its width. It now reads "missed (closed at 6% of width)" whenever the rule being graded is width-based.

### Added
- 16 regression assertions for the width rules in `scripts/fixtures/test_payoff_regressions.ts`, anchored to real rows: 1765 (5-wide, clears both targets), 1762 (3-wide 2-lot — pins the per-unit trap at 150 rather than 75), 1702 (a $551 winner that still misses 50% of a 15-wide, proving the rule is not a win/loss proxy), 1749 (debit fly), plus null-width, zero-width, manual-override and futures-multiplier cases. TypeScript gate 75 → **91**; Python gate unchanged at 53.

### Known
- **The width-based GTC *price formula* is still questionable (#292, open).** `gtc = |credit| + width × pct` treats every width-rule structure as a debit fly bought to be sold. A credit put BWB whose tent peak carries intrinsic value does not close that way — trade 1765 collected 0.33 against a max profit of 5.33. Filed rather than redesigned; GTC order-entry pricing sits under the audit's "belongs in a trading platform" list.
- On the 30 closed butterflies and BWBs where 25% of width is even attainable, **2 reached it.** That is real feedback about management, and it was invisible while the rule was stubbed.
- **The rule can still be unreachable by construction on credit spreads (#293, open).** A 5-wide credit vertical taken for 1.50 has a max profit of $150 against a `width_50` target of $250 — permanently false. That is the same defect as #290 narrowed from the whole book to "credit structures where the credit is below `pct × width`". The playbook picker offers every rule against every strategy; it should gate on compatibility.
- **The comparison is net-of-commission on the left and gross on the right.** Realized `pnl` is net of the closing commission; the target is derived from strike geometry. A fill exactly at the printed GTC therefore scores as a miss by roughly the commission. Always conservative, a few dollars, and deliberately left alone rather than papered over with an epsilon — but it means "I closed right at the target and it says I missed" is expected on a borderline trade.
- `closed_at_target` is a generic flag, not a width-specific one — `CalendarTab` renders it as "50% profit target" and the analytics tabs read it as a target-close marker. Ticking it earns compliance against a width rule as well. Preserved as-is because changing it would re-grade trades Chris ticked by hand.

---

## [3.20.0] — 2026-08-06

### Fixed
- **`spread_width` reached only five strategy labels and missed 214 trades (#286).** `compute_spread_width` dispatched on `spreadType` and read the denormalized `shortStrike` / `longStrike` columns, which are `0` on 55 short put verticals whose `legs_json` carries the strikes perfectly well. Every butterfly, every broken-wing butterfly and 20 short call verticals got nothing at all. The TypeScript twin `computeSpreadWidth` returned the *outer* strike span, which is the wide wing on a BWB — 50 on trade 1434, where the risk wing is 25.
- **Trade 1634 carried a negative width of −10, and trade 1325 a width that hid two naked short puts (#286).** Both are now NULL. 1325's stored 10 described its call spread while two short 7-strike puts sat underneath it, so the sizing check and the credit/width test were both reading a defined-risk structure that was not one.
- **The reconciliation pass rewrote columns it had determined were unchanged (#288).** Recomputing a stored double reproduces it to within about one ULP rather than bit-for-bit, so rewriting rows for a width change re-rounded three `cached_max_loss` values by ~4e-13. Semantically zero, but it made the "no other column moved" audit unprovable — which is the check that catches a real regression. Each column now keeps its stored value wherever the tolerance says it did not move.

### Changed
- **Width is now derived from the payoff vertices, shared with the max profit / max loss engine (#286).** `width = max(V)` per unit, falling back to `−min(V)` when the structure carries no expiry liability at all, where `V(S)` is the expiry liability scanned at `{0, every strike}`. Width is `max(V)` because `max(V)` is what max loss is measured against — that makes it a property of the strike geometry with no premium in it and no dependence on the strategy label. It is deliberately not the payoff's total span: on a broken-wing butterfly the span is the wide wing and `max(V)` is the narrow risk wing, and only the narrow wing reproduces the stored BPR.
- **NULL, never 0, for anything that is not structurally a spread (#286).** Net put quantity or net call quantity nonzero (an unhedged leg — a cash-secured put's `V(0)` is its whole strike, so an ungated scan would have reported a width of 205 on trade 1221), more than one expiration, or a strategy label that says calendar/diagonal while the legs do not show two expiries.
- `computeSpreadWidth` (TypeScript) and `compute_spread_width` (Python) are marked `@deprecated` for risk math. `TradeForm` now writes width from the vertex derivation.

### Added
- **`credit_width_ratio` backfilled for every credit structure that has a width (#287).** It stood at 8 of 459 rows because only `TradeForm` ever wrote it; 325 rows had both inputs and a NULL ratio. Population 8 → 333, every value inside (0, 100]. Trade 1434 reads 14.0% against its narrow risk wing; against the wide wing it would have read 7%.
- 10 TypeScript and 11 Python regression fixtures covering the width rule on the same structures in both engines — BWB, debit fly, multi-lot normalization, the wider-side iron condor, and the four NULL cases. Suites now stand at **75** TypeScript and **53** Python.

### Data
- Backfilled `spread_width` on 112 rows: 108 NULL → value, 3 demoted to NULL, 1 corrected (trade 1719, 20 → 30). Population **242 → 347**. `credit_width_ratio` written on 325 rows.
- The rule was validated before any write by recomputing the 242 rows that already carried a width: **238 reproduce**, and all four disagreements are bad stored values rather than rule failures.
- Idempotent — a second pass reports 0 — and an ATTACH diff against the pre-write `VACUUM INTO` backup shows zero changes to `cached_max_profit`, `cached_max_loss`, `bpr`, `pnl`, `credit_received`, `legs_json`, `strategy` or `quantity`.

### Known
- **Trade 1719's stored BPR of 1345 matches neither side of the structure (#289).** It is three 10-wide short call verticals stacked with a 20-wide short put vertical; the call side's aggregate liability is 30, so max loss is 2072 — which is what `cached_max_loss` already holds. `compute_bpr`'s iron-condor branch takes `max(put_width, call_width)` from the first two legs of each type and never aggregates stacked spreads. Filed, not fixed.
- The strict both-sides-hedged gate also rejects a call ZEBRA (trade 1504), which is defined-risk with net-long tails. Three of the four rows it rejects this way are single-leg long options where NULL is unambiguously right. Left alone deliberately; widening the gate is its own change.

---

## [3.19.0] — 2026-08-05

Closes the last critical item on the audit's §8.2 list.

### Fixed

- **`cached_max_profit` and `cached_max_loss` were never written at import (#283).** They sat on 15 and 14 of 459 trades. `import.py` created both columns and never populated them; the only writer was `TradeForm`. Everything that divides by max profit was computing on 3% of the book — `% of max profit captured`, the `profit_target_N` compliance rule, the AnalyticsTab basis, the alerts profit target — and everything needing max loss fell back to `bpr`, which is collateral, not risk. `% of max profit captured` now reads on **259** closed profitable trades instead of 15.
- **Max profit and max loss were sampled from a padded chart window (#284).** `calculateMaxProfit` and `calculateMaxLoss` take the extremum of a payoff curve drawn over `[minStrike − pad, maxStrike + pad]` with `pad = max(spread × 0.45, spot × 0.04)`. A cash-secured put's worst case is at S = 0 and a long call's best case is at S → ∞; both lie outside that window. Every one of the 15 populated rows was wrong as a result — trade 1603 stored a **$51.60** max loss on a 3-lot CSP against a true **$3,840**, trade 1722 stored **$2,034.45** against **$20,559**, and trade 1597 stored a **$63.80** ceiling on a long call whose upside has none.
- **Seven calendar and diagonal trades carry `legs_json` that does not show two expiries (#285).** Two have `expirationDate` null on every leg, two show the same date on both legs, three carry a single leg. A single-expiry calculation reads these as ordinary verticals; trade 1800 produced a $3,317 max profit that way. Both columns are now written as NULL whenever the strategy label is calendar-family but the legs disagree. The underlying leg data still needs repairing at import.

### Changed

- **Both quantities now come from the payoff vertices, in one scan.** The expiry payoff is piecewise linear in S with kinks only at strikes, so *both* extrema of the liability `V(S) = Σ (short ? +1 : −1) × intrinsic × qty` are attained at a vertex. Profit at expiry is `P(S) = netPremium − V(S)`, so max loss reads off `max(V)` and max profit off `min(V)` — probing `{0, every strike}` gives both exactly, with no window to get wrong and no dependence on strategy label, leg order or wing symmetry. `boundedMaxLoss` (shipped in 3.18.0) and the new `boundedMaxProfit` share the scan in both languages.
- **NULL where a bound does not exist, never 0.** A naked short call has no max loss and a naked long call no max profit; a calendar has no single-expiry payoff at all, since its value at front expiry depends on the back month's implied vol — a model output, not a bound. A covered call stores only its short call leg, so the shares that cap the downside are not in the record and its max loss is NULL rather than invented. Every consumer tests `!= null`, so a 0 would read as a real zero.
- **`TradeForm` writes through the same functions**, so hand-entered and imported trades land on one basis — and it now passes the contract multiplier, which the old call path did not.
- **The importer reconciles both columns as a whole-table pass** at the end of each run rather than inline in `insert_trades`, because trades reach the table through four writers (insert, roll update, intraday IC merge, expiry auto-close). Idempotent: a second run writes 0 rows.

### Added

- 20 TypeScript fixtures and 17 Python fixtures for the two quantities (65 and 42 total, from 45 and 25). Eight expected numbers appear in both suites, so cross-language drift fails a test. Fixture leg data is copied verbatim from `trades.db`; the `/NG` iron condor case pins the futures multiplier and independently reproduces that trade's stored BPR.

### Data

- Backfilled 458 of 459 rows. `cached_max_profit` populated on 405, `cached_max_loss` on 436, with every NULL accounted for: 47 multi-expiry or unbounded-upside for profit, 15 covered calls plus one credit diagonal for loss. An ATTACH diff against the pre-repair snapshot confirms `pnl`, `bpr`, `legs_json`, `status`, `credit_received`, `pop` and `spread_width` are untouched. The two inverted GLD condors (1531, 1540) correctly return NULL max profit — they were entered below their floor liability.

---

## [3.18.0] — 2026-08-05

### Fixed

- **The importer's butterfly BPR regressed to the full wing span (#280).** Between 2026-05-06 and 2026-05-12 `compute_bpr` began reading a broken-wing butterfly's risk as `K_hi − K_lo` instead of the difference between the wings. Trade 1756 stored **$3,900** of buying power reduction against a true max loss of **$900**. Five trades were affected, carrying $7,400 of phantom BPR into ROC, AROC, heat, BPR% and every sizing check that touched them. The regression window was identified by the corrected rule reproducing 16/16 stored values written before it and disagreeing with exactly the 5 written after.
- **`calculateBpr` disagreed with the importer on every defined-risk structure (#281).** Hand-entered trades sat on a different BPR basis than imported ones: the credit was never subtracted (trade 1225 read 1000 where the broker took 922), and a butterfly body ratio was read as lot size (trade 1434 read 10,000 against a true 2,150).
- **`long_call_butterfly` was missing from `StrategyType` (#282).** Trade 1673 fell back to `custom`, which classified a $235-max-loss debit fly as undefined-risk, gave it no guide, rendered its max profit as a negative number, and left its BPR NULL. Added alongside `long_put_butterfly`.

### Changed

- **Max loss is now read off the payoff at its vertices instead of asserted per shape.** `boundedMaxLoss` (TypeScript) and `bounded_max_loss` (Python) evaluate `V(S) = Σ (short ? +1 : −1) × intrinsic × qty` at `{0, every strike}` and return `max(V) − net premium`. The expiry payoff is piecewise linear with kinks only at strikes, so the maximum is attained at a vertex and the probe is exact — no sweep resolution, no dependence on strategy label, leg order, wing symmetry or body ratio. Verticals, condors, butterflies, ratio flies and diagonals now fall out of one function instead of five hand-derived shapes.
- A tails-only derivation was implemented first and rejected on review: correct for the 1-2-1 butterfly it was fit to, wrong for reverse flies (the payoff maximum sits at the body, not a tail), for 2-3-1 wing imbalance (the tail is leg-weighted), and for 1-3-1 bodies (which carry naked shorts and are not defined-risk at all). All three are now fixtures.
- `boundedMaxLoss` returns null rather than 0 when risk is unbounded above the top strike or when the derived max loss is non-positive. A non-positive max loss means a guaranteed profit at entry, which in this book has always signalled a data defect, and a zero would be indistinguishable from "unknown" at every call site that tests `!bpr`.

### Added

- 12 BPR fixtures in `test_payoff_regressions.ts` (45 assertions total) and 12 in `test_pnl_regressions.py` (25 total). Every value is either the number the broker actually took, read from `trades.db`, or hand-derived from the expiry payoff where the stored value is known wrong.
- `scripts/fix_bwb_bpr.ts` — the one-shot repair. It imports `boundedMaxLoss` rather than restating the formula, since a second copy of the rule is what caused the bug. Corrected 5 rows and backfilled 9 that were NULL; all 14 are closed trades, so open-position sizing was unaffected ($90,829 before and after).

---

## [3.17.0] — 2026-08-04

### Changed

- **`calculateP50` rewritten as a moving-boundary lattice (#279).** The old implementation returned `1 − P(underlying ever touches strike ∓ ½credit)` — a "probability of never being tested" metric containing no time-decay term at all. Since decay is the entire mechanism by which a premium seller reaches 50%, it read roughly 8x low: an ATM 45-DTE cash-secured put at 25% IV returned **9.31%** where tastytrade shows 70–85%. It now returns **77.7%**.

  Reaching 50% is a first-passage problem against a boundary that *moves*. The target is hit when `P&L(S, τ) = netPremium − V(S, τ) ≥ target`, and because the structure value `V` depends on remaining time as well as spot, the boundary starts unreachable at entry and falls toward its intrinsic level at expiry — that same ATM put decays to half its credit with the underlying *unchanged* at roughly 3.6 DTE. Freezing the boundary at any single level reproduces the original error: a frozen barrier that has fallen below spot reports as already touched while the position still holds the full credit.

  The engine now propagates probability mass over a 64-step recombining CRR lattice and absorbs it the first time a node's P&L reaches the target. Closed-form Black-Scholes at every node, deterministic, and structure-agnostic — **one engine replaced nine hand-derived per-strategy formulas**, each of which had been an independent chance to be wrong. The iron-condor branch in particular used to multiply two first-passage probabilities as though the touch events were independent, when they are one path.

- **The 50% target now depends on the sign of the opening cashflow.** Net-credit structures target half the *credit*; net-debit structures target half the *max profit* read off the expiry payoff. For naked shorts, strangles and credit verticals max profit **is** the credit, so the two coincide and the ambiguity never surfaces. Credit butterflies, ratio spreads and lizards are where they diverge violently: trade 1434 collects **3.50** against a **28.50** theoretical max, so half-of-max would target 14.25 — a pin on the body — and print single digits for a trade that gets closed at half credit routinely. Put-BWB cohort average moves **5.9 → 82.0** across 18 trades; debit butterflies are unchanged at 15.1.

- **Coverage went from 12 to 455 of 459 trades.** The old switch fell through to `return null` for every strategy it did not name — long verticals, ZEBRAs, long options, `custom`, and `long_call_butterfly` (which is missing from `StrategyType`, still open). Reading the structure off the legs instead of the strategy label removes that whole class of gap. Cohort averages shift as expected: cash-secured put 46.5 → 85.0, short put vertical 46.4 → 79.0, covered call 49.8 → 88.0, iron condor 19.1 → 53.9.

- **Two trades now return `null` where they previously returned the 5.0 clamp floor.** Trades 1531 and 1540 (GLD) are *inverted* iron condors — the short call sits below the short put — collected for 4.86 and 4.54 against guaranteed minimum liabilities of 5.00 and 7.00. Max profit is negative, so there is no 50% of it to reach. Worth a look: those are locked-in losses at entry.

- **Multi-expiration structures return `null` explicitly.** Calendars, diagonals and PMCCs are out of model — this engine prices every leg to one expiry, so a back-month leg would be valued at intrinsic on the front month's expiration and the structure would collapse to a fake guaranteed loss. They returned `null` under the old implementation too, so nothing regresses; the difference is that the refusal is now deliberate and documented rather than incidental.

  Known biases, all in the conservative direction and left uncorrected rather than papered over with a fudge factor: discrete monitoring at 64 steps (Broadie-Glasserman-Kou boundary shift of `0.5826·σ·√dt`, ~1–4pp low); risk-neutral drift, which earns none of the real-world drift or vol risk premium a premium seller actually collects; and a single flat IV with no skew or term structure.

- Removed the now-unused local `normalPdf` and `bsD1` duplicates from `payoff.ts`. `greeks.ts` remains the single Black-Scholes implementation — a second copy is how the breakeven defect in #273 survived.

### Added

- 16 new `calculateP50` assertions in `scripts/fixtures/test_payoff_regressions.ts` (suite now 33). Credits in the synthetic cases are priced with `bsOptionPrice` at the same IV the assertion uses, so each position is internally consistent. Covers the tastytrade band, moneyness/duration/volatility ordering, the condor-vs-its-own-put-side relation, the credit-vs-debit target rule pinned against both real DB butterflies, all five refusal paths, and a 459-call performance budget (currently 208ms).

---

## [3.16.0] — 2026-08-04

### Fixed

**Six calculation- and display-layer defects found by a full-application audit (#273–#278).** The Black-Scholes core in `src/calculations/greeks.ts` was verified correct — every defect below lived in how those primitives were called, what fell back when data was missing, and how results were rendered. The TypeScript calculation layer previously had **zero** test coverage; `scripts/fixtures/test_payoff_regressions.ts` now guards it with 17 assertions anchored to real `trades.db` rows.

- **#273 — analytical breakevens dropped leg quantity.** `calculateBreakevensForJournal` summed leg premiums with no quantity factor, defended by a stale "matches rust — NO qty factor" comment. But `leg.quantity` is a per-unit **ratio** for butterfly bodies and ZEBRA long pairs (a 1-lot fly stores `[1,2,1]`) while it is an **absolute** contract count for verticals, condors and CSPs (a 6-lot stores `[6]`). Verified against the book: put_butterfly 1494 read a 14.65 debit instead of 3.00 (breakevens 6114.65/6285.35 instead of 6103/6297), and put_broken_wing_butterfly 1434 read a −91.86 credit instead of +3.50, placing its breakeven at 6141.86 instead of 6046.50 — 95 points off and on the *wrong side of the short strike*. Fixed by normalizing every leg quantity against the structure's minimum, which resolves both storage conventions to the same per-unit ratio. `czbr`/`pzbr` `numLong` and `ratio_spread` `nShort` are normalized identically so multi-lot structures stay lot-size invariant.

- **#274 — Expected Value fabricated upside when max loss was unknown.** `TradeDetail` fell back to `maxLoss = Math.abs(t.cached_max_loss ?? t.bpr ?? 0)`. `cached_max_loss` is populated on 14 of 459 trades and `bpr` is NULL on every symmetric butterfly, so `maxLoss` silently became `0` and EV degenerated to `POP × maxProfit` — trade 1494 displayed **EV +$9,178** on a butterfly whose real risk is $300. BPR is collateral, not max loss; for a cash-secured put the two differ by exactly the credit. `maxProfit`/`maxLoss` are now `number | null`, derived from the spread width for defined-risk structures and the payoff sweep for butterflies, and `null` otherwise so the UI renders "—" (or "undefined" for undefined-risk max loss). `calculateExpectedValue` now rejects `maxProfit <= 0` and `maxLoss <= 0`. The butterfly fallbacks also no longer multiply by trade quantity a second time (the payoff sweep already scales by `leg.quantity`), which was reading 9× on a 3-lot fly, and they now use the contract multiplier instead of a hardcoded 100.

- **#275 — max P/L and numerical breakevens ignored the calendar back-month adjustment.** `calculateMaxProfit`, `calculateMaxLoss` and `calculateBreakevens` hardcoded `strategy`, `frontMonthDte` and `iv` to `undefined` when calling `calculatePayoffCurve`, whose back-month Black-Scholes block is gated on `strategy != null`. The entire adjustment was therefore dead code for max P/L: every calendar, diagonal, PMCC and double-diagonal — 48 trades in the book — had its maximum computed as if the long back-month leg expired worthless alongside the front month. The three arguments are now optional parameters threaded through to the curve. `payoff.worker.ts` received them and dropped them; it now forwards them, along with the contract multiplier it had no field for at all (every futures payoff chart was off by the multiplier ratio).

- **#276 — Avg ROC rendered 100× too small on the Dashboard.** `PortfolioStats.avg_roc` is a ratio. `OverviewTab`, `AnalyticsTab` and `PerformanceCharts` all multiply by 100; `DashboardTab:1424` (Risk Distribution) and `:1552` (strategy breakdown) did not. A 12% average ROC rendered as "0.1%" on one tab and "12.0%" on the next.

- **#277 — net theta could not report its own sign.** `Math.abs()` plus a hardcoded "+" and hardcoded green at five sites (`DashboardTab:189, :721, :1573`; `OverviewTab:484, :500`) meant a net-long-premium book — negative theta, bleeding daily — displayed as positive green income. Also fixed `OverviewTab:504`, where the Avg POP colour threshold compared a 0–1 fraction against 70/60 and so was permanently red even at 95% POP, two lines from a display expression that scaled correctly.

- **#278 — per-leg Greeks double-applied the short sign and mixed units.** The `TradeDetail` per-leg block applied `const sign = isShort ? -1 : 1` to values `estimateLegGreeks` had **already** signed, inverting delta on every short leg and contradicting the correct aggregate two blocks above. The same row rendered three different scales (delta per-share, theta in contract dollars, vega per-share) with `Math.abs()` hiding the sign on theta and vega. Now displayed verbatim from `estimateGreeks` — signed, in position dollars, with unit suffixes — so the per-leg rows sum exactly to the aggregate GREEKS block.

### Added

- `scripts/fixtures/test_payoff_regressions.ts` — first regression suite for the TypeScript calculation layer. 17 assertions covering butterfly and BWB breakevens against real trades 1494 and 1434, lot-size invariance for ratio structures, the absolute-quantity vertical convention, ZEBRA ratio division, and the EV null guards. Run with `bun scripts/fixtures/test_payoff_regressions.ts`.

Verified: `bun scripts/fixtures/test_payoff_regressions.ts` 17/17 pass · `python3 scripts/fixtures/test_pnl_regressions.py` 13/13 pass · `bunx tsc --noEmit` clean · `bun run build` succeeds.

---

## [3.15.42] — 2026-08-01

### Fixed

**Three special-path P&L mis-bookings in the importer (#269, #270, #271)** — each previously required a hand repair after daily reconciliation; all now book to the broker's realized figure and are guarded by regression fixtures in `scripts/fixtures/test_pnl_regressions.py`.

- **#271 — roll-close used stale `credit_received`.** A rolled single short put mislabeled as a `put_calendar_spread` carries a net roll value in `credit_received`, not the leg's own open premium, so `(credit_received − debit_paid)×100` detonated the P&L (RKLB 1780 booked −2924 vs true −1569). `compute_pnl_and_debit` now derives the open credit from the leg premiums and uses it whenever it disagrees with the stored `credit_received` by more than a cent; correctly-structured trades (where they agree) are unchanged.

- **#270 — expired trades kept their open-time round-trip commission.** Expiry incurs no closing commission, but the stored `commission` is a round-trip estimate, over-subtracting ~$1.32/vertical (AAPL 1787 booked 169.35 vs broker 170.67). `auto_close_expired_legs` now strips the closing half when the stored commission is round-trip-shaped (per-contract > $1.00); an already-open-only commission is left untouched. (~1¢ rounding residual vs broker.)

- **#269 — cash-settled index ITM expiry booked as worthless.** SPX/XSP/NDX/RUT settle ITM via blank-amount `EXP … exercise` rows, so the worthless default (closePremium=0) could be wildly wrong (SPX 1756 booked +94.84 vs true −333.53). The importer now never books a cash-settled index expiry worthless: it reconciles the realized P&L from the statement's authoritative P/L-YTD delta (`pl_ytd − pl_open − DB_realized(excl. this trade)`) when it is cleanly attributable — a P/L row exists, exactly one index trade for the ticker is in the sweep, and the result sits inside the leg-based risk envelope — and otherwise WARNs and leaves the trade OPEN for manual settlement. It never silently books a number it cannot stand behind.

Verified: `scripts/fixtures/test_pnl_regressions.py` — 13 assertions, all pass (RKLB −1569.00, correctly-structured IC unchanged, AAPL open-only commission, SPX −333.53, and all three #269 leave-OPEN guards). `py_compile` clean.

---

## [3.15.41] — 2026-08-01

### Fixed

**Import History no longer mis-attributes expiry/ITM closes (#272)** — the Import History panel's "amount" is computed as `SUM(trades.pnl) WHERE import_session_id = session.id`. Inserted and updated trades (including standard orphan-close and roll/assignment closes) flow through `insert_trades` and are stamped with the current session via `id_map`. But `auto_close_expired_legs` books expiry and index-ITM closes with a direct `UPDATE` that never enters `id_map`, so those closes kept the trade's *opening* `import_session_id` — crediting realized P&L to the wrong (earlier) import. Symptom: the settling day showed $0.00 while an old session's amount was inflated.

Fix: `auto_close_expired_legs` now returns the list of trade ids it closed; the session-linking step unions those ids with `id_map` before stamping `import_session_id` and computing the session's `total_pnl`. Additionally, the stored `import_sessions.total_pnl` is now recomputed for **all** sessions at the end of every import to match the live figure the UI shows, self-healing any staleness left by a post-import manual pnl repair.

Verified: `auto_close_expired_legs` return type is a list; the linking block unions `expired_ids`; a synthetic expiry-close import stamps the current session id and the panel total equals the live `SUM(pnl)` recompute. Data cleanup for the 7 historically mis-attributed trades was applied 2026-07-31 (commit `8b13c14`); this release stops the recurrence at the source.

---

## [3.15.40] — 2026-07-18

### Fixed

**Startup no longer flashes on first launch (macOS TCC / Documents permission)** — the app's database lived at `~/Documents/Projects/theta-vault/trades.db`. macOS gates `~/Documents` behind a TCC consent prompt, so the first cold launch of any given build showed "Theta Vault would like to access files in your Documents folder"; until it was granted the SQL plugin could not open the DB, and the splash/main windows flashed repeatedly during the blocked state. Granting the prompt persists, which is why quitting and relaunching loaded cleanly — and why freshly-built versions (which re-trigger the prompt) flashed "most of the time".

Fix: the project now lives at `~/Projects/theta-vault`, outside every TCC-protected folder, so the prompt never fires. `resolve_db_url` is unchanged in logic (still the single `trades.db` next to `src-tauri/`) — it simply resolves to the new, unprotected location; a comment now warns against moving the project back under `~/Documents`. The Python importer shares the same file at the same relative path.

Verified: cold launch of the release build from the new location (WebKit caches cleared to force a true first-run) shows no permission prompt, no flashing, and the Dashboard loads directly.

---

## [3.15.39] — 2026-07-17

### Fixed

**Rolling both sides of a condor in one statement no longer half-closes the trade (#259)** — each VERT/CONDOR ROLL order built its close-update from the stale DB legs snapshot; with two rolls on the same trade in one batch, the apply loop (which has no same-id coalescing) let the second UPDATE clobber the first, leaving the parent half-closed with phantom legs. This was the oldest open importer bug.

New `_merge_or_build_close_update()` routes every close-update through one door: if the batch already holds an update for that trade, the builder is re-run seeded with the *pending* update's legs (all key-map / all-closed / exec-time logic reused, none duplicated) and the result merges in place — commissions and fees sum across orders, the original fingerprint is kept, and nothing is appended twice. Wired into the VERT ROLL, CONDOR ROLL, and orphan-close sites; the assignment path (3.15.37/38) already guarded.

Verified reproduce-first on a synthetic double-rolled SPY condor: pre-patch shows the documented clobber (call legs null, 2 phantoms); post-patch the parent closes fully at the hand-computed −$150.00 with exit at the second roll's time, both new verticals open at absolute credits, 4/4 reconciled, re-run idempotent. Regressions (IBM assignment e2e, 7/16 replay, live-copy idempotency) byte-identical to 3.15.38.

---

## [3.15.38] — 2026-07-17

### Fixed

**Hardening of the #261 assignment-close path (Forge audit findings)** — (F1) the fresh path could emit a second update dict for a trade that already had one pending (e.g. an expiry close for another leg of the same trade); the apply loop has no same-id coalescing, so the second UPDATE would clobber the first. The merge scan now also covers updates produced by the assignment pass itself, and the fresh path refuses to append when ANY update for that trade id is pending — printing a loud review-manually warning instead (covers the case where expiry processing already closed the assigned leg at 0.0). (F2) exp-close updates now carry `_close_exec_time`, and the merge falls back to the update's `exitTime`, so a morning assignment can no longer win `max()` against a later same-day settlement by default. (F3) documented that the strict leg match is the guard against a fee-contaminated strike derivation (fails safe with a warning).

Verified: full re-run of the 3.15.37 test matrix (IBM e2e byte-identical −507.81, 7/16 regression unchanged, idempotency 0/0) plus new unit probes for the F1 guard (pending non-open leg → 0 emissions + WARN) and the F2 fallback (16:00 settlement beats 10:00 assignment).

---

## [3.15.37] — 2026-07-17

### Fixed

**Share-assignment EXP rows now close the assigned leg on the parent trade (#261)** — assignment rows (`BOT/SLD N.0 TICKER UPON …`) carry a share quantity and no strike/CALL/PUT text, so the option-expiry parser skipped them and the assigned short leg stayed phantom-open. Required manual repair on consecutive imports (GLD 1747 on 7/16, IBM 1794 on 7/17).

New `_build_tos_assignment_closes()` runs after ATH and expiry processing: recovers the strike as |Amount| / shares (assignment settles at strike; BOT shares = short put assigned, SLD = short call), finds the open leg, and prices it by anchor priority encoding the manual-repair convention: (a) a same-type same-expiry leg closed within 5 days → paired price ± strike width (deep-ITM legs carry no extrinsic, so the spread books at its true width); (b) a same-statement share-disposal row → strike vs disposal price; (c) no anchor → **loud warning, leg left open** — never a silently wrong price. Assignment metadata (`was_assigned`, `assigned_shares`, `cost_basis` = strike) is persisted via the update path; `exit_reason` = `assigned`; exit time is the later of the paired close and the assignment event.

Verified: replaying the 7/17 statement against the pre-import DB closes IBM 1794 end-to-end byte-identical to the manual repair (pnl −$507.81, assigned leg 84.55 via paired-leg anchor, was_assigned/100/290); the 7/16 replay is behavior-identical to 3.15.36 with no false trigger from the GLD share-sale row; unit probes cover the disposal anchor (exact) and the no-anchor warn path; live-copy re-run idempotent (0/0, 23/23). Known noise: re-importing a statement whose assignment was already applied prints two benign WARNs (trade already closed).

Not covered (filed separately): assignments that occur in a statement gap with no EXP row in any imported file (the GLD 1747 shape) — needs position-delta inference.

---

## [3.15.36] — 2026-07-16

### Fixed

**A multi-leg close order spanning split trades now closes ALL of them (#266)** — when an earlier roll split a position into two DB trades (iron condor → call vertical + put vertical), a later single closing order covering all legs matched only the first strike-intersecting trade; `_build_tos_close_update` silently dropped the sibling's fills, leaving it phantom-open. Hit twice in production: JPM 1777/1778 (7/2) and AMZN 1789/1790 (7/16), both requiring manual DB repair.

The orphan-close block now matches iteratively: a new `_split_close_fills_for_trade()` gives each matched trade only the close fills corresponding to its own legs (keys mirror the builder's `(strike, opt_type, expiry-month)` map with `(strike, opt_type)` fallback, so single-trade closes behave exactly as before); the remainder tries the remaining open trades. Full order commission goes to the first matched trade, matching the house-style manual repairs. A partial-unmatched remainder now prints a WARN instead of vanishing.

Verified: e2e replay of the pre-import DB (631904b) against the 7/16 statement closes BOTH AMZN trades at exactly the manually-repaired values (1789 +$106.36, 1790 −$49.00, legs 2.79/1.62); GLD single-trade close unchanged; idempotent re-run on the repaired DB: 0 added / 0 updated / 25 of 25 legs matched / 0 phantoms. Cross-audit (Forge): PASS — split set proven a strict superset of the builder's close set, loop bounded three ways, fingerprints per-trade distinct, and the fallback restructure recovers an intraday-match path old code denied.

Watch item (not fixed here): the VERT-ROLL path shares `_match_tos_close_to_db` and has the same latent shape if a roll's close side ever spans split trades — tracked on #266.

---

## [3.15.35] — 2026-07-02

### Fixed

**Roll-opened trades no longer store the roll's net credit as the spread's credit (#265)** — when a trade was opened via `VERT ROLL` (or same-expiry CONDOR roll), the importer handed the Cash Balance amount — the NET credit of the entire roll (close + open combined, e.g. `@.07`) — to `_build_tos_new_trade` as the new trade's `credit_received`. Legs store absolute premiums, so when the rolled trade later closed at an absolute debit, `pnl = (roll_net − absolute_debit) × 100` produced large phantom losses. Three trades detonated on the 7/2 import: JPM short_call_vertical −$377.66 (true +$36.00), PLTR −$840.00 (true +$220.00), CRCL −$606.00 (true −$80.00).

Fix is at the ingestion point: the roll-open call site now passes `None` for `cb_amount`, forcing the builder to derive the absolute credit from the open legs' premiums (short − long). The plain-open path is untouched — Cash Balance remains authoritative there (verified: MCD covered call still parses cr=+4.30).

Also fixed in the same fallback branch: the leg-premium sum is now normalized by trade quantity, so a future multi-lot roll stores the per-unit credit (2-lot AAPL vertical → 1.72, not 3.44) — consistent with the per-unit convention `compute_pnl_and_debit` expects since 3.15.34.

Data repair applied alongside: the three detonated trades corrected, JPM put-vertical phantom closed (+$17.00), and the six latent roll-opened trades' credits rebased to absolute (AAPL 1787→1.72 / 1788→0.98, AMZN 1789→1.29 / 1790→0.68, CRCL 1793→6.25, IBM 1794→4.93). Verified: scratch-DB re-import parses CRCL cr=+6.25 / IBM cr=+4.93; live re-run idempotent (0 added / 0 updated); 29/29 open legs reconcile; JPM and CRCL statement-YTD gaps collapsed to fees-level.

---

## [3.15.34] — 2026-06-30

### Fixed

**P&L no longer double-counts quantity on multi-contract trades (#260)** — `compute_pnl_and_debit` accumulated the close credit by multiplying each leg's close premium by the leg's absolute quantity, and then the final P&L line multiplied the whole sum by the trade quantity again — squaring the contract count (qty²) for any strategy that stores leg quantity as an absolute count (covered_call, cash_secured_put, multi-contract verticals/condors). A 6-lot covered call rolled for a small per-share gain was reported as −$2,889.97 instead of +$440.03.

Root cause: `leg.quantity` is stored as the ABSOLUTE contract count for most strategies (a 6-lot covered call stores leg `[6]`), but as a per-trade-unit RATIO for butterfly center legs (a 1-lot butterfly stores `[1,2,1]`). The close-credit loop treated the absolute count as a ratio. The fix normalizes each leg to its per-trade-unit ratio (`leg_qty / trade_qty`) before accumulating, so the final `× trade_qty` produces the correct absolute contract count for every strategy: a covered call's 6/6 = 1, a butterfly center's 2/1 = 2.

Verified against four cases: T covered-call 6-lot (+$440.03), NVDA put-vertical 2-lot (+$51.40), SPY iron condor 1-lot (+$69.35, unchanged), and a 1-lot butterfly with a 2× center leg (ratio weighting preserved).

Note: this patch fixes P&L computed on future imports. Historical closed trades that already stored a double-counted P&L require a one-time recompute sweep (tracked separately).

---

## [3.15.33] — 2026-06-26

### Fixed

**CONDOR ROLL wing-merge is now idempotent** — when re-running an account-statement import that contains a CONDOR ROLL (one side of an iron condor rolled to new strikes), the importer no longer overwrites the parent IC's closed-wing legs with the new strikes. Previously, re-running destroyed the partial-close state by dropping the closed call (or put) legs from the parent IC and replacing them with the new strikes, while ALSO keeping the separately-created new-side trade — producing duplicate call legs across two trades and breaking reconciliation with QTY MISMATCH errors.

The fix adds a re-entrancy guard to `_find_partial_ic_for_wing_merge`: before returning a partial IC as a wing-merge target, the function now checks whether ANY other open trade in `db_open_trades` already holds OPEN legs at the same strikes / expiration / option type as the new fills. If so, the wing merge has already been processed on a prior run, and the function returns None to prevent destructive overwrite.

Concrete example: 6/26 statement had AMZN 280/285 → 260/265 and AAPL 310/315 → 300/305 call rolls. On the first import these correctly partial-closed 1758/1772 and created 1781/1782 as new short call verticals. Re-running used to destroy 1758/1772 by dropping the closed 280/285 and 310/315 legs and reusing the new strikes; now the guard sees 1781/1782 already hold those strikes and skips the merge.

---

## [3.15.32] — 2026-06-22

### Fixed

**Import early-return no longer skips post-processing** — when an account statement contains zero genuinely new trades (e.g. only re-reported prior closes), the importer previously printed "No new trades to import" and exited before running `auto_close_expired_legs`, `reconcile_open_positions`, the Account Summary print, or the P&L YTD cross-check. The result: 6/18-expired trades stayed marked `open` in the DB even though the 6/22 statement no longer listed their legs in the open Options table.

The fix replaces the early-return with a `skip_trade_import` flag. The new-trade ingestion section (absorb intraday IC call rolls → Greeks enrichment → playbook assignment → `insert_trades` → roll-chain linking) only runs when there ARE new trades. Auto-expiry, reconciliation, account summary, and P&L cross-check ALWAYS run for ThinkOrSwim statements — they're independent of new-trade arrival and the user wants the statement reconciled regardless.

**DB cleanup (2026-06-18 expirations):** Seven trades auto-closed by the post-fix re-import:

- IWM 1767 (long_diagonal_spread) — worthless, P&L −$826.67
- QQQ 1576 (iron_condor) — **manually adjusted to assignment**: short 675C assigned + long 685C exercised → net −$1,000 settlement + $465 credit = **−$535**, exit_reason='assigned' (auto-close had computed +$465 worthless)
- SPX 1598 (put_broken_wing_butterfly) — worthless, P&L +$154.84
- SPX 1700 (short_put_vertical) — worthless, P&L +$267.42
- SPX 1714 (short_put_vertical) — worthless, P&L +$39.84
- SPX 1715 (short_put_vertical) — worthless, P&L +$60.12
- WMT 1768 (long_diagonal_spread) — worthless, P&L −$817.67

After cleanup: 60/60 open option legs reconcile exactly with the 6/22 statement Options snapshot.

---

## [3.15.31] — 2026-06-18

### Fixed

**Import auto-expiry no longer fires on 0DTE legs still listed open on EOD statement** — when a same-day expiration trade is opened intraday (e.g. SPX 0DTE put vertical opened ~2pm ET, expires at the close), the broker EOD statement for that day still carries the legs in the open Options table because settlement processes overnight. The previous `auto_close_expired_legs` rule fired purely on `today > expirationDate` and zeroed both legs at $0, marking the trade closed. The next day's statement then showed the legs as MISSING FROM DB because the DB had auto-closed too early.

The hardened rule now takes the statement's open positions snapshot as the authoritative settlement clock. `auto_close_expired_legs(conn, stmt_open_keys)` accepts a set of `(ticker, strike, expiry_iso, OPT_TYPE)` tuples drawn from the statement's Options + Futures Options tables. If ANY of a candidate trade's null-premium legs is still in that set, the trade is skipped — the broker hasn't settled it yet. Only when the NEXT statement omits the leg does auto-close trigger, which is the correct behavior.

The DB-state fix: trade ID 1776 (SPX 7470/7465 PUT vertical, 2026-06-17) was reopened — `closePremium` reset to NULL on both legs, `status='open'`, `pnl/exit_date/exit_reason` cleared. Next 2026-06-18 statement import will see the legs absent from the Options snapshot and auto-close it cleanly with `exit_reason='expired'` and the proper credit-only P&L.

---

## [3.15.30] — 2026-06-04

### Added

**Portfolio Backtest sub-tab (#254)** — new Performance → Backtest view for combined weighted cumulative P&L across selected strategies.

- 4 KPI cards (TOTAL P&L, WIN RATE, TOTAL TRADES, RETURN ON CAPITAL) computed from the weighted subset; win rate and trade count stay unweighted since weight is a sizing knob, not duplication.
- Contract Size Weighting panel with one row per selected strategy: colored dot (from shared `STRATEGY_COLORS` palette), strategy label, defined-risk badge (DR/UR — tastytrade convention), and −/value/+ stepper clamped to [0, 5] in 0.5 increments.
- Combined Cumulative P&L Recharts AreaChart; curve coloring switches between `--green` and `--red` based on final P&L sign.
- Default top-6 selection: pinned set (`put_broken_wing_butterfly`, `iron_condor`, `short_put_vertical`, `cash_secured_put`, `long_diagonal_spread`, `short_diagonal_spread`) for any strategy with ≥1 closed trade, slack filled by highest-volume non-pinned strategies.
- Live recalculation: chart and KPIs update on every weight change; Recalculate button briefly flashes green as visual feedback. Reset to 1× restores baseline.
- Data aggregation in pure `backtestMath.ts` (group closed trades by `(strategy, exit_date day)`, per-strategy running cumsum, union-axis forward-fill combine).

**Strategy color extraction** — `STRATEGY_COLORS` map moved from inline `ChartsTab.tsx` into shared `src/lib/strategyColors.ts` with `stratColor()` helper. Added entries for `put_broken_wing_butterfly`, `call_broken_wing_butterfly`, `short_diagonal_spread`, `call_butterfly`, `long_call_butterfly`, `long_put_vertical`, `long_call_vertical`, `czbr`, `calendar_spread`, `custom`.

### Fixed

**Sign-inverted P&L on multi-contract orphan-close imports (#252)** — corrected ID 1603 SOFI cash_secured_put: `debit_paid` 2.34 → 0.78 (was double-counted as `closePremium × quantity` instead of per-share), `pnl` −343.99 → +124.01. Scope check across all 409 trades confirmed ID 1603 was the only affected row. Importer fix itself tracked in #252 — this changelog entry covers the data correction.

**Misclassified diagonal-roll ticket (#253)** — corrected ID 1757 RKLB: `strategy` `short_diagonal_spread` → `cash_secured_put`, `rolled_from_id` set to 1744, `roll_count` = 1. The original DIAGONAL ticket was structurally a put-roll (TO CLOSE 17 JUL 105 + TO OPEN 21 AUG 95), so the resulting open position is a single short put, not a diagonal spread. Importer fix tracked in #253.

### Imported

- `scripts/2026-06-04-AccountStatement.csv` — 7 trades (SOFI long_call close, SOFI CSP close, AAPL SPV close, MSFT SPV close, RKLB CSP close, CRM SPV close, RKLB diagonal-roll open). All open-position legs reconciled 56/56 against statement.

---

## [3.15.28] — 2026-05-27

### Fixed

**Futures contract multiplier — BPR, MaxP, MaxL, ROC now correct for /CL, /NG, /ZC, /ES, etc.**

Pre-3.15.28, all dollar-producing calc paths (`legPayoffAtPrice`, `calculateBpr`, `calculateRoc`, `calculatePnlFromLegs`, `calculateAnnualizedYield`, `calculateExtrinsicRemaining`, `estimateLegGreeks`) hardcoded the $100 equity multiplier. Futures options use different multipliers: /CL = $1000/pt, /NG = $10,000/pt, /ZC = $50/pt, /ES = $50/pt, etc. So /CL trades displayed BPR/MaxP/MaxL/ROC understated by 10×; /ZC was overstated 2×; /NG understated 100×. P&L itself was correct because the import script already applied the right multiplier when writing pnl.

- **New helper**: `src/lib/contractMultiplier.ts` — `getContractMultiplier(ticker)` returns 100 for equities/ETFs/index options and the CME-spec multiplier for futures roots (24 supported, from `/ES` to `/6A`). `futuresRoot()` strips contract-month codes (`/CLN26` → `/CL`).
- **Threaded through**: every dollar-producing function now takes an optional `multiplier` parameter (defaults to 100 for backward compatibility). Callers — TradeDetail, JournalTab, DashboardTab, PlaybookTab, OverviewTab, portfolio.ts aggregation — pass `getContractMultiplier(t.ticker)`.
- **Data backfill**: `scripts/backfill-futures-bpr.py` rescales stored `bpr`, `cached_max_profit`, `cached_max_loss` for all 3 existing futures trades. Backup: `trades.db.bak_20260527_futures_multiplier`. Verified on /CL #1710: BPR $71 → $710, MaxP $29 → $290, ROC 63.3% → 6.33% (matches Schwab broker statement).

### Known limitation

Stored per-share Greeks (`trade.delta`, `trade.theta`, `trade.vega`, `trade.gamma`) for multi-contract trades imported under the post-2026-05-27 leg.quantity convention (where leg.quantity = trade.quantity) have trade.quantity baked into the stored value, causing downstream `× trade.quantity` consumers (DashboardTab BWD, portfolio.ts net Greeks) to double-count. Affects only multi-contract symmetric trades imported via the new convention; equity 1-contract trades are unaffected. Filed as follow-up — to be fixed by normalizing import script to always store per-1-contract Greeks regardless of leg.quantity.

---

## [3.15.27] — 2026-05-27

### Fixed

**Leg quantity convention — silent undercount bug class**

`leg.quantity` is now the absolute per-leg total contract count, set on every leg of every trade. Previously, multi-contract trades imported pre-3.15.27 stored `leg.quantity` as `null`, causing the legs-path (BPR, payoff curve, BS Greeks from legs) to default to 1 and silently understate position size by `trade.quantity ×`. 146 legs across 32 closed multi-contract trades (and 1 open: T #1232 covered_call qty=6) were affected.

- **Data migration**: `scripts/migrate-leg-quantity.py` walks `trades.db legs_json` and sets `leg.quantity = trade.quantity` wherever it was missing. Butterfly/BWB ratio legs (27 legs where `leg.quantity != trade.quantity`) are preserved as-is. Idempotent — re-runs are no-ops. Backup written to `trades.db.bak_20260527_leg_qty_migration` before migration.
- **Write guard**: New `normalizeLegs(legs, tradeQty)` helper in `src/db/queries.ts` is applied at every leg write path — `insertTrade`, `updateTrade` (when `legs` provided), `closeTrade` — so new rows can never re-introduce the bug.
- **Read guard**: `hydrateTrade()` also backfills `leg.quantity` from `row.quantity` if any pre-migration row sneaks through.

Verification: open-position reconciliation against the broker statement still passes 77/77 leg match post-migration.

---

## [3.15.1] — 2026-05-20

### Added

**Journal — Annualized ROC column (AROC%)**

New toggleable column in the Journal table: **AROC%** (Annualized Return on Capital). Formula: `ROC% × 365 ÷ entry DTE`. Shows what your actual ROC would be if annualized — the correct way to compare a 14-DTE iron condor at 8% ROC to a 45-DTE strangle at 12% ROC. Sortable, included in CSV export, tooltip explains the formula.

**Models — big_lizard strategy**

`big_lizard` is now a first-class strategy type with proper label, badge ("BL"), and metadata matching the jade_lizard profile. Previously, imported big_lizard trades silently degraded to "custom" strategy, breaking analytics and compliance checks.

### Fixed

**BPR — OTM-adjusted margin for strangles and naked positions**

`calculateBpr()` now accepts an optional `underlyingPrice` parameter. When provided (from TradeForm's live spot field), uses the Reg T OTM-adjusted formula: `max(20% × underlying − OTM_amount + premium, 10% × underlying + premium)` instead of the conservative flat `20% × strike`. This gives accurate ROC% for wide strangles — previously understated by 15–25% for 5-delta strangles. Stored BPR for existing imported trades is unchanged (conservative is safe). TradeForm now passes underlying price to both the live preview and trade save paths.

**Performance tab — live MTM unrealized P&L**

The Performance → Overview tab now receives `livePortfolioStats` (with real BS mark-to-market unrealized P&L when live prices are available) instead of the static theta-proxy `portfolioStats`. The P&L line also shows "(est.)" for open positions with a hover tooltip explaining the estimation basis.

---

## [3.15.0] — 2026-05-20

### Added

**Analytics — P&L skewness (#169) and max drawdown duration (#170)**

Two new metrics in Performance → Overview right panel:
- **P&L Skewness** — measures the shape of your trade P&L distribution. Negative skewness is normal and expected for premium sellers (many small wins, occasional large losses). Very negative (< -2) may indicate unmanaged losers.
- **DD Duration** — calendar days of the longest drawdown (time from equity peak to recovery). Complements Max DD % with the psychological dimension: 20% DD over 7 days vs. 90 days are completely different experiences. Green < 30d, yellow 30-90d, red > 90d.

**Analytics — Theta capture by DTE bucket (section S, #172)**

New section **S** in Analytics tab. Splits theta capture % into three buckets by DTE at close:
- ≥21 DTE (managed per tastytrade rule)
- 10–21 DTE
- <10 DTE (held into final stretch)

For each bucket: trade count, WR%, theta capture %, avg P&L. Answers empirically: "does the 21-DTE rule actually improve my theta capture?" Theta Capture = actual P&L ÷ theoretical θ-decay (√t model). 100% = kept full premium. >100% = direction/IV helped. Negative = position moved against you.

### Changed

**Alert simplification — removed gamma_risk, pin_risk, delta_extreme (#178)**

Three alert kinds removed from the alert system:
- `gamma_risk` — required stored gamma + ≤7 DTE trigger. Replaced with a plain `manage` alert: "at N DTE — final week, close or roll immediately."
- `pin_risk` — required live spot within 0.5% of short strike at expiry. Belongs in TOS.
- `delta_extreme` (portfolio) — required live beta-weighted delta calculation. Belongs in TOS.

The 12 remaining alert kinds cover all the journal-relevant cases: defense (expires today, tested), max_loss, warning (earnings, calendar), dividend_approach, undefined_drift, drawdown, roll_chain, manage, close, roll, sizing, ok.

**PayoffChart deprecation notice (#179)**

Added a banner above the theoretical payoff chart: "Theoretical expiration payoff — uses stored entry prices, not current market structure. TOS/TastyTrade show this better in real-time. Roadmap: replace with realized P&L timeline."

### Fixed / Polish

**#174** — 21-DTE timeline marker in TradeDetail now has a tooltip explaining √t coordinate space: why the marker isn't at 50% of the bar, and what "theta-decay space" means.

**#175** — IV normalization convention documented in `normalizeIv`/`normalizeIvFrac` in `src/lib/format.ts`. The `> 2` heuristic (decimal vs. percent stored IV) is now explained in a JSDoc contract comment.

**#167, #173** — Already implemented in prior code. Closed: `~` stale Greek indicator was already rendered; ZEBRA strategy guides (czbr/pzbr) were already in StrategyGuidePanel.tsx.

---

## [3.14.0] — 2026-05-20

### Added

**Trade timeline (Gantt) view — GANTT button in Journal tab (#194)**

Click **GANTT** in the Journal header to switch from the table to a horizontal Gantt-style timeline. Click again to return to the table. The detail panel on the right still works — click any bar to load that trade's detail.

**What it shows:**
- Each trade is a horizontal bar spanning its entry → exit date (open trades extend to today)
- Tickers are grouped into rows — trades within the same ticker that overlap in time are packed into separate lanes automatically (greedy interval packing, no overlaps)
- **Bar height ∝ BPR** — thicker bar = more capital allocated to that trade
- **Color:** cyan = open, green = win (P&L > 0), red = loss
- Strategy badge visible inside bars wide enough to show it
- Month grid lines and a TODAY marker for orientation
- Period filter: 3M / 6M / 1Y / All

**What this reveals that the table never could:**
- When you were over-allocated (many thick bars at the same time)
- How your positions overlapped — did you pile into SPX while already heavy on AAPL?
- Campaign visualization — rolling sequences on the same ticker
- Sizing evolution over time — are your bars getting thicker or thinner?

The timeline uses `allFiltered` (all filtered trades, not limited by the 200-trade closed cap) so you see the full picture.

---

## [3.13.9] — 2026-05-20

### Fixed

**Audit bug batch — #159–#166**

Six calculation and display bugs from the Tom Sosnoff audit, plus two already-resolved issues closed:

- **#161 — legPayoffAtPrice undefined for unknown leg types:** Added `default: return 0` to the switch statement. Previously, any unrecognized leg type (imported data with a typo, future leg types) produced `undefined`, propagating NaN through the entire payoff curve and corrupting breakevens, max profit/loss, P50, and PIT.

- **#162 — bsD1 crash when spot ≤ 0:** Added `spot <= 0 || strike <= 0` guard alongside the existing `t <= 0 || sigma <= 0` guard. A stale price feed returning 0 caused `Math.log(0)` = -Infinity, silently poisoning all Greek calculations with NaN.

- **#163 — DTE@Close quality % wrong when findIndex returns -1:** Replaced brittle `findIndex` searching for "21" in bucket label text with a direct edge-comparison filter: `dtecloseBuckets.filter((_, i) => dtecloseEdges[i] >= 21)`. If no label matched, `slice(-1)` returned only the last element, reporting ~0% close quality incorrectly.

- **#164 — Profit Factor color thresholds inconsistent:** OverviewTab line 665 used `≥1.5 = green / ≥1.0 = yellow` while AnalyticsTab used `≥2.0 = green / ≥1.5 = yellow`. Now unified to `≥2.0 green / ≥1.5 yellow / <1.5 red` (tastytrade standard) in both tabs.

- **#165 — CT timezone hardcoded as CST (UTC-6):** Replaced `const CT_OFFSET = 5` with `ctHourOf(d)` using `Intl.DateTimeFormat` with `timeZone: "America/Chicago"`. CDT (UTC-5, March–November) and CST (UTC-6, November–March) are now handled automatically. Previously 6 months/year had entry times misclassified by 1 hour, corrupting first-hour trading analysis.

- **#166 — BE move % divides by zero:** Changed `spotForGreeks` truthiness check to explicit `spotForGreeks != null && spotForGreeks > 0` guard. Prevents the division from running when spot is undefined or zero.

- **#159, #160 — already resolved:** Calmar yearSpan null guards and monthly P&L numeric sort were both already corrected in earlier code. Issues closed as fixed.

---

## [3.13.8] — 2026-05-20

### Added

**Calendar tab enhancements — seasonal patterns + per-month strategy breakdown (#196)**

**Per-month strategy breakdown** (month view, below the day-of-week footer):

When viewing any month, a new section shows every strategy used that month with count, win rate, avg P&L per trade, and total P&L. Sorted by trade count. Answers "what was I running in March?" at a glance.

**Year-over-year comparison grid** (always visible below the 12-month trend bar):

A heatmap table with years as rows and months (Jan–Dec) as columns. Each cell shows that month's total P&L colored by magnitude — green for profitable months, red for losses, intensity proportional to the amount. The TOTAL column shows the full year P&L.

- Only appears when you have ≥ 2 years of closed trade data
- Click any cell to jump directly to that month's calendar view
- The currently selected month is outlined in accent color
- Answers "is October consistently bad for me?" or "do I always make money in Q1?" across the full history

---

## [3.13.7] — 2026-05-20

### Added

**Quick close context menu — right-click any trade row (#195)**

Right-click any trade row in the Journal tab to get a context menu. For open trades:

- **⏳ Expire worthless** — closes the trade immediately with P&L = full credit received, exit date = expiration date, exit reason = expired. Shows a browser confirm with the exact P&L amount before executing. Bypasses the form entirely — the common Friday expiration routine is now two clicks.
- **✕ Close (fill P&L)** — opens the close form pre-selected on this trade
- **↻ Roll** — opens the Roll Advisor for this trade

For all trades (open and closed):
- **✎ Edit** — opens the edit form
- **✕ Delete** — confirms then deletes

Menu auto-dismisses on outside click. Clamped to window edges so it never renders off-screen.

---

## [3.13.6] — 2026-05-20

### Added

**Saved filter presets (#187)**

The filter builder in the Journal tab now has a PRESETS bar above the condition rows.

- **↓ save** button appears when at least one condition is active — click it, type a name, press Enter to save the current filter as a named preset
- Saved presets appear as chips; click any chip to instantly restore that filter
- **×** on each chip to delete it
- Presets are stored in `localStorage` (survive app restarts, no server required)
- Pressing Escape while naming dismisses the input without saving

Example use: save "High IVR Iron Condors" (strategy = iron_condor + IVR > 50) as a one-click query to run whenever you want to review that slice of your journal.

---

## [3.13.5] — 2026-05-20

### Added

**Process quality analytics — section R in Analytics tab (#184)**

Press `R` to open PROCESS QUALITY — powered by the post-mortem data captured at close (#182).

**Thesis outcome matrix (2×2):** Every closed trade with a thesis outcome is placed in one of four cells:
- ✅ Correct + Won — good process, good outcome (the goal)
- 😤 Correct + Lost — good process, bad luck (penalized by variance)
- 😬 Wrong + Won — bad process, good luck (dangerous — looks like skill, isn't)
- ❌ Wrong + Lost — bad process, bad outcome (the one to minimize)

Each cell shows: count, % of total, avg P&L. Header shows overall thesis accuracy rate.

**Quality score impact (1–5 scale):** Three tables side-by-side — Entry Quality, Management Quality, Sizing Quality. For each score level (1=poor → 5=excellent): count, avg P&L, win rate. Reveals whether higher process scores actually correlate with better outcomes in your trading.

Shows "no post-mortem data yet" when neither thesis outcomes nor quality scores have been recorded.

---

## [3.13.4] — 2026-05-20

### Added

**Tag taxonomy + multi-select picker (#183)**

Replaces the free-text tags input in the trade entry/edit form with a structured multi-select tag picker.

Five predefined categories — click any tag to toggle it on/off:
- **Entry:** high_ivr, low_ivr, chased_entry, waited_for_setup, earnings_risk, tested_early
- **Exit:** held_too_long, panic_exit, closed_at_target, early_exit, let_expire, managed_at_21
- **Sizing:** oversized, undersized, max_allocation, appropriate_size
- **Process:** followed_playbook, broke_rules, gut_trade, revenge_trade, best_execution
- **Market:** trending_market, high_vix, low_vix, earnings_crush, vix_spike, sector_rotation

Custom tags still supported — type in the custom field and press Enter or comma to add.

Storage format is unchanged (comma-separated string), so existing imported tags are preserved and the full-text search (`g` key) continues to work across all tags. Predefined tags are now consistently named (underscore-separated), making them reliably queryable via the filter builder and full-text search.

---

## [3.13.3] — 2026-05-20

### Added

**Probability calibration — new section Q in Analytics tab (#192)**

Press `Q` in the Analytics tab to open PROBABILITY CALIBRATION — a feature unique to Theta Vault among retail options journals.

The question: are your probability estimates accurate? If your model says 70% probability, do 70% of those trades actually win?

Two calibration tables side-by-side:

- **POP calibration** — uses the stored POP value at entry, grouped into buckets: <50%, 50–60%, 60–70%, 70–80%, 80–90%, ≥90%. Shows expected win rate (avg POP in bucket) vs actual win rate, with error in percentage points
- **P50 calibration** — computes P50 at entry from stored trade data (legs, IV, DTE, entry price) for each closed trade; same bucketing and comparison

Each row shows: Bucket | N | Expected | Actual | Error (with visual bar) | Avg P&L

Color logic: **green = actual exceeded model** (model is conservative — you outperform your own estimates); **red = actual below model** (model is overconfident — you win less than predicted). Perfect calibration = errors near 0.

Requires ≥3 trades per bucket to display. Cells below threshold show a data-availability notice.

---

## [3.13.2] — 2026-05-20

### Added

**Management adherence analytics — new section P in Analytics tab (#191)**

Press `P` in the Analytics tab to open the MANAGEMENT ADHERENCE section. Answers the question: "do I actually close trades when I say I will?"

Three panels:

- **DTE Management** (vs tastytrade 21-DTE standard): shows what % of your trades were managed at ≥21 DTE (green), held 5–21 DTE (yellow), or held to expiry (red), with avg P&L per bucket — shows whether early management pays off for you specifically
- **Profit Target Adherence**: % closed at target vs held past it, with avg P&L for each group — quantifies whether sticking to your target rule matters
- **Early Exits Warning**: flags trades closed pre-21 DTE without hitting their profit target (discipline failures)
- **Avg DTE@Close by Strategy**: table showing each strategy's average exit DTE — green ≥21d ✓, yellow 10–21d, red <10d

---

## [3.13.1] — 2026-05-20

### Added

**IVR entry analysis — theta capture + best-band recommendation (#190)**

Enhances the IV RANK AT ENTRY section in Analytics:

- **Avg θ/day column:** Shows average entry theta (absolute value) per IVR bucket, so you can see which IV environments generate the most daily decay
- **Best-band highlight:** The IVR bucket with the highest win rate (minimum 3 trades) is marked with ★ and a subtle green background
- **Footer recommendation:** "★ best IVR range for you: 50-60 — 73.1% WR, avg +$142 per trade (11 trades)" — tells you exactly where your edge lives

---

## [3.13.0] — 2026-05-20

### Added

**Ticker analytics panel — sortable, full-stats breakdown (#189)**

The TICKER BREAKDOWN section in Analytics tab is now a 9-column sortable table. Click any column header to sort; click again to reverse.

- **New columns:** Avg DTE at entry, Avg IVR at entry, Avg ROC%, Sharpe (ROC-based, requires ≥ 2 trades)
- **WR% coloring:** green if above portfolio average, red if below, yellow if equal — tells you at a glance where you have edge vs. where you give money back
- **Statistical significance:** tickers with < 5 trades shown in muted color with `·` marker
- Default sort: Total P&L descending (highest-contribution tickers first)

**Learning curve metrics — rolling-50 win rate + quarter-over-quarter (#188)**

Two additions to the Charts tab that answer "am I actually improving?"

*Rolling-50 win rate:* The existing 20-trade rolling win rate line is now joined by a 50-trade rolling window (green line). The 50-trade line only appears once 50+ closed trades exist, providing a smoother signal. Both computed from full trade history (not period-filtered) so the learning curve spans the complete journal.

*Quarter-over-quarter table:* Below the win rate chart, a compact grid shows the last 8 quarters side-by-side — Trades / WR% / Avg P&L / Total P&L per quarter. Respects the period filter. WR% colored green ≥ 60%, yellow ≥ 50%, red < 50%. Answers the question "which quarters were my best, and is the trend improving?"

---

## [3.12.0] — 2026-05-20

### Added

**Full-text search across notes, thesis, and tags (#186)**

Press `g` in the Journal tab to open a "search notes, thesis, tags…" input. The search runs instantly across five text fields on every trade: `notes`, `entry_thesis`, `tags`, `grade_notes`, and `lesson_text`. Case-insensitive substring match.

- Works alongside the existing ticker filter (`f`) and condition filter builder — all three stack as AND logic
- Active query shown as `notes: <query>` in the header (cyan); cleared with Escape
- Keyboard shortcut added to status bar hint and Admin keyboard reference
- No DB query — filters the already-loaded trades array in-memory (fast for any realistic journal size)

---

## [3.11.0] — 2026-05-20

### Added

**Structured post-mortem at close (#182)**

Closes the learning loop opened by the entry thesis (#181). After recording a trade's exit, an optional collapsible "POST-MORTEM" section lets you score your process and capture the lesson while it's fresh.

- **Entry Quality** (1–5): Was your entry timing, IVR environment, and strike selection appropriate?
- **Management Quality** (1–5): Did you follow your management plan?
- **Sizing Quality** (1–5): Was the position sized correctly for the setup?
- **Lesson** (text): One sentence — what would you do differently?
- **Thesis Outcome** (select): Thesis correct + won ✅ / Thesis correct + lost 😤 / Thesis wrong + won 😬 / Thesis wrong + lost ❌

**UI design:** Collapsed by default in the close form — click "▶ POST-MORTEM (optional)" to expand. Score buttons are colored (green=4–5, yellow=3, red=1–2). All fields optional; saving without them works normally.

**TradeDetail HISTORY tab** shows a POST-MORTEM block with dot-scale score display (●●●○○), thesis outcome label, and lesson text.

**DB:** 5 new columns added via idempotent migration: `entry_quality_score`, `management_quality_score`, `sizing_quality_score`, `lesson_text`, `thesis_outcome`.

---

## [3.10.1] — 2026-05-20

### Fixed

**Import: same-day open+close pairs now match correctly (#197)**

When a 0DTE trade is opened and closed within the same account statement, the importer was creating two separate trades (both marked as expired worthless) instead of one closed trade with the correct P&L. Root cause: the Account Trade History is in reverse chronological order, so the 19:45 close was processed before the 17:40 open existed in the pending trades list.

Two fixes applied to `scripts/import.py`:

1. **Chronological sort**: groups are now sorted by exec_time before processing, so opens always precede their same-session closes regardless of CSV order.

2. **Pending-batch close fallback** (`_match_close_to_pending_batch`): when a close fails to match a DB trade, it now also searches the new trades created in the same import batch. If found, the close is applied in-place: `exitTime`, `exitReason`, `pnl`, and leg `closePremium` values are all set correctly before the trade is inserted.

Manual DB correction also applied: trade ID 1720 (SPX 7320/7310 PUT, 5/19/26) was updated from `pnl=$142.56 / reason=expired` to `pnl=$0.12 / reason=closed` with correct leg close premiums.

---

## [3.10.0] — 2026-05-20

### Added

**Entry Thesis field — capture WHY before the outcome is known (#181)**

- New `entry_thesis` text field on the Trade model; idempotent `ALTER TABLE` migration runs on app startup
- Shown as an editable textarea in the New Trade form with placeholder: "Why this trade? What setup needs to hold? What would invalidate the thesis?"
- Read-only italic display in Edit mode (locked after first save — a before-the-fact record)
- Prominently displayed in TradeDetail ENTRY tab above the conditions block when set

**CSV Export — full trade history as downloadable file (#193)**

- "↓ Export CSV" button in Admin tab settings header
- Fetches all trades lazily on click; downloads as `theta-vault-trades-YYYY-MM-DD.csv`
- Includes 31 fields: id, ticker, strategy, entry/exit dates, credit, P&L, BPR, DTE, entry_thesis, notes, tags, grade, sector, IV, Greeks, status, exit reason, and computed held_days
- Flash confirmation: "✓ Exported N trades"
- Blob URL download — no file-system plugin required

**Multi-condition filter builder — slice trade history by any combination of fields (#185)**

- New "FILTER" button in the Journal tab header (highlighted with active condition count when in use)
- Collapsible panel with "Add condition" button; each condition row has field selector, operator selector, and value input
- Supported fields: Strategy, Ticker, IVR, Entry DTE, P&L, Tags, Sector, Earnings Play, DTE at Close
- Operators adapt to field type: `=`, `>`, `<` for numbers; `=`, `contains` for text; `=` for boolean
- Conditions combine with existing status and ticker filters via AND logic
- "Clear all" clears conditions and collapses the panel
- Input keydown events stopped from propagating to the journal keyboard handler

**Manual price refresh — R key on Dashboard (#180)**

- `useMarketData` hook now exposes a `refresh` callback alongside market data
- Pressing `R` on the Dashboard tab triggers an immediate live price + Greeks refresh
- Matches the existing "R:refresh" keyboard shortcut hint already shown in the status bar

---

## [3.8.0] — 2026-05-18

### Added

**Import: now reads all 5 meaningful account statement sections (scripts/import.py)**

`parse_tos_csv()` previously captured only Cash Balance and Account Trade History. It now parses all sections line by line:

- **Account Order History** (`account_order_history`) — order confirmations: strategy type, fill side, qty, pos effect, symbol, expiry, strike, opt type, status. Provides ground-truth strategy classification for every order executed during the statement period.
- **Options** (`options`) — snapshot of all current open equity option positions with signed qty, strike, expiry, and current mark. Used for post-import reconciliation.
- **Futures Options** (`futures_options`) — same snapshot for futures options. Ticker extracted from the composite symbol field; expiry date parsed from the date portion of the symbol string.
- **Profits and Losses** (`profit_loss`) — per-ticker P&L data: P/L Open and P/L YTD. Used for P&L cross-check.

**Import: post-import open position reconciliation (`reconcile_open_positions`)**

After every ThinkorSwim import, the importer cross-references every open leg in the account statement (Options + Futures Options) against every open leg in trades.db — leg by leg, by (ticker, strike, expiry, opt_type, signed_qty). Reports:

- `MISSING FROM DB` — leg exists in the account but not in trades.db
- `PHANTOM IN DB` — leg exists in trades.db but not in the account
- `QTY MISMATCH` — leg present on both sides but signed qty differs
- `✓ All N legs match` — clean state

Rolled legs (closePremium set on a leg within an otherwise-open trade) are excluded from the DB side since they represent a closed side of a still-open rolled trade. Results appear in the RAW LOG and as `[RECONCILE]` entries in the import warnings panel.

**Import: P&L cross-check**

Tickers that show `$0 P/L Open` in the statement (all positions closed) but still have open trades in trades.db are flagged with `[PNL_CHECK]` warnings — catches cases where a trade expired or was closed but the DB was not updated.

**Motivation:** These changes catch the class of bug observed with EWY (wrong put strikes prevented close matching) and TLT (non-standard structure prevented close matching) — both showed as `$0 open P/L` in the account statement while remaining open in trades.db. The reconciliation would have flagged both immediately.

Closes #138

---

## [3.7.5] — 2026-05-18

### Fixed

**Import: updated trades now show correct P&L in review panel (scripts/import.py)**
- When a closing CSV is imported, the UPDATED TRADES panel always showed +$0.00 for all trades
- Root cause: `compute_pnl_and_debit()` correctly calculated P&L and wrote it to the database, but never stored it back on the trade dict — the summary builder then read `t.get('pnl') or 0` (always 0)
- Fix: one line added after the successful DB UPDATE: `trade['pnl'] = round(pnl, 2) if pnl is not None else 0.0`
- P&L values in the DB were always correct — this was a display-only bug in the import review panel

---

## [3.7.4] — 2026-05-17

### Added

**Performance: new CALENDAR sub-tab — full-fledged daily trading calendar (CalendarTab.tsx)**
- New `CalendarTab` component added as a 4th sub-tab in the Performance section alongside Overview, Charts, Analytics
- Full monthly calendar grid (Sun–Sat 7-column, 6-row) with proper day-of-week alignment and prev/next month overflow cells

**Calendar: monthly navigation with inline month picker**
- Prev/next arrows navigate months, clicking the month/year title opens a 12-button inline month picker
- Year navigation buttons (◂ 2024 / 2025 ▸) to jump across years
- `YEAR VIEW` toggle button switches between month and year overview modes
- Fixes #125

**Calendar: day cards with P&L, trade count, and strategy badges**
- Each day cell shows: day number, signed daily P&L (+$1,234 / -$312), trade count, strategy type mini-badges (max 3, +N overflow)
- Today's cell: blue accent border and "TODAY" label
- Selected day: cyan border with elevated background
- Weekend columns (Sun/Sat): subtle dark overlay to visually distinguish non-trading days
- Off-month days (prev/next month): 30% opacity, no trade data shown
- Fixes #126

**Calendar: weekly P&L totals column**
- 8th column to the right of each week row showing that week's cumulative P&L
- Appears automatically when at least one trade exists in the selected month
- Fixes #127

**Calendar: day detail panel — trade breakdown on click**
- Clicking any trade day opens a detail panel below the calendar grid
- Header: date, total daily P&L, wins/losses count
- Per-trade rows: strategy badge, ticker, signed P&L, exit reason (50% profit target / expired worthless / rolled / etc.), DTE range
- Click same day again or ✕ to close
- Fixes #128

**Calendar: year overview mode — 12 mini-month heatmap grid**
- YEAR VIEW toggle shows all 12 months in a 4×3 grid
- Each mini tile: month name, total P&L, trade count; color intensity proportional to max |monthly P&L| in the year
- Current real-world month shown with accent border and ● indicator
- Click any tile to navigate to that month
- Fixes #129

**Calendar: day-of-week avg P&L footer**
- All-time average P&L by day of week (Mon–Fri) shown below the calendar
- Best performing weekday highlighted with ▲ BEST label
- Computed from all closed trades across all time (requires ≥2 trades per day to show)
- Fixes #130

**Calendar: monthly KPI ribbon**
- 6-chip ribbon between navigation header and calendar grid: MONTH P&L, WIN RATE, TRADES, BEST DAY, WORST DAY, AVG/DAY
- Win rate chip colored tastytrade-style: ≥65% green, ≥50% yellow, <50% red
- Updates as user navigates months
- Fixes #131

**Calendar: 12-month P&L trend bar**
- Mini bar chart (48px) at the bottom showing last 12 months P&L
- Current month bar outlined in accent color for easy orientation
- Always visible in both month and year view modes

---

## [3.7.3] — 2026-05-17

### Fixed

**Charts: sector heatmap cell color now reflects profitability (ChartsTab.tsx)**
- Profitable sectors render green cells, unprofitable sectors render red cells
- Previously all cells were cyan regardless of win rate — `isProfitable` was computed but only applied to the sector label text color, not the cell background
- Fixes #102

**Charts: rolling Sharpe and rolling theta capture no longer show broken line gaps (PerformanceCharts.tsx)**
- Added `connectNulls={true}` to both rolling metric Line components
- NaN values (insufficient window data) were converted to `null` which caused Recharts to break the line — visually indistinguishable from a rendering error
- Fixes #103

**Charts: reference line labels no longer clip at right edge of chart panels (PerformanceCharts.tsx)**
- Changed `position: "right"` to `position: "insideTopRight"` on all reference line labels in the Drawdown, Rolling Win Rate, Rolling Sharpe, and Rolling Theta Capture charts
- At minimum window width (1100px), `position: "right"` rendered labels outside the SVG clip rect
- Also alternated Theta Capture labels (100% insideTopRight, 80% insideTopLeft, 50% insideTopRight) to prevent overlap
- As a bonus: Rolling Theta Capture 100% reference line now uses `var(--cyan)` instead of `var(--text-muted)` for proper visual weight
- Fixes #104

**Charts: Monthly P&L and Win Rate Trend x-axis now show deterministic tick distribution (PerformanceCharts.tsx, ChartsTab.tsx)**
- Replaced `interval="preserveStartEnd"` with `interval={Math.max(1, Math.floor(data.length / 6))}` on both charts
- `preserveStartEnd` is Recharts' least reliable interval mode — with 18+ months of data, intermediate labels drop silently
- Now guarantees ~6 evenly spaced labels regardless of dataset size
- Fixes #105

---

## [3.7.2] — 2026-05-17

### Fixed

**Admin: `margin_account` setting now has a UI checkbox (AdminTab.tsx)**
- Added "Margin Account (CSP uses 20% notional BPR)" checkbox in the ACCOUNT section of settings
- The `margin_account` field was added to AppSettings in v3.5.0 and the CSP BPR calculation correctly used it — but there was no way to toggle it from the UI. Margin-account traders were stuck at 5× overstated CSP BPR (100% collateral vs 20% Reg-T)
- Fixes #99

**Admin: `account_size` input now enforces minimum value of 1 (AdminTab.tsx)**
- Added `min="1"` to the account_size number input
- Previously accepted 0 or negative values, causing division-by-zero in `bprPct`, `thetaNetliqRatio`, and all sizing calculations
- Fixes #100

**Admin: `run_import` now has a 5-minute subprocess timeout (lib.rs)**
- Converted `run_import` to async and wrapped with `tokio::time::timeout(300s)`
- Uses `tokio::task::spawn_blocking` for the blocking `cmd.output()` call
- Previously, a hung Python process (Yahoo Finance network timeout, bad CSV) would block the Tauri invoke indefinitely with no escape mechanism
- Added `tokio = { version = "1", features = ["time", "rt"] }` to Cargo.toml
- Fixes #101

**Admin: settings save now shows ✓ Saved confirmation (AdminTab.tsx)**
- Added a 2-second "✓ Saved" flash next to the SETTINGS header after successful save
- Previously the edit panel just closed silently — users had no confirmation the save worked

**Admin: fixed stale closure in useEffect (AdminTab.tsx)**
- Added `settings` to the `useEffect` dependency array `[editing, settings]`
- Previously `startEdit` closed over stale `settings` reference when settings changed between renders

---

## [3.7.1] — 2026-05-17

### Fixed

**Sharpe ratio: population std dev → sample std dev (portfolio.ts)**
- Portfolio Sharpe and rolling Sharpe now divide by `n − 1` instead of `n`
- At 20-trade rolling window, population std dev overstated Sharpe by `√(20/19) ≈ 2.6%` — systematic upward bias across all Sharpe displays
- Fixes #95

**Rolling theta capture: per-trade average → aggregate ratio (portfolio.ts)**
- Rolling theta capture chart now uses `totalActual / totalTheo` (aggregate) matching the portfolio-level theta capture in OverviewTab
- Per-trade average was dominated by outlier trades: one trade with `pnl = −$500` vs `theoretical = $50` = −1000% capture, collapsing the entire rolling chart to a large negative number
- Fixes #96

**Entry timing: UTC hours → CT market hours (OverviewTab.tsx)**
- Entry time analysis now applies `CT_OFFSET = 5` (UTC−5 CST) when bucketing trades into market hours
- `getUTCHours()` on TOS timestamps (stored in CT) would give wrong market hour buckets on any non-UTC machine
- Midnight filter also updated to use CT-adjusted hour for consistency
- Fixes #97

**Strategy Sharpe/Sortino: credit ROC → actual P&L ROC (AnalyticsTab.tsx)**
- Per-strategy Sharpe and Sortino now computed from `(pnl / bpr) × 100` (actual outcome ROC, can be negative) instead of `(credit_received / bpr) × 100` (always positive)
- With credit ROC: all values positive → no downside deviation → Sortino = 9.9 (cap) for every strategy — meaningless
- With actual P&L ROC: losing trades contribute negative values, making Sharpe and Sortino statistically meaningful
- `avgRoc` display still uses credit ROC (entry potential yield) — correct for that column
- Also applies sample std dev (n−1) to strategy Sharpe calculation
- Fixes #98

---

## [3.7.0] — 2026-05-17

### Fixed / Enhanced

**Roll Advisor: debit-roll warning (RollAdvisor.tsx)**
- NET ROLL cell now shows a red `DEBIT` badge when rolling for a debit (`netRoll < 0`)
- A yellow footer warning appears when any scenario requires paying to extend the trade
- Rolling for a debit extends risk without credit — tastytrade advice is to avoid unless thesis requires more time
- Fixes #93

**Actions: fix default collapsed alert groups (ActionsTab.tsx)**
- "roll" removed from the default collapsed set — 21-DTE roll alerts now expand on load
- "close" added to the default collapsed set — profit-target alerts collapse by default (good news, not urgent)
- New default: `["warning", "sizing", "manage", "close"]`
- Fixes #94

---

## [3.6.0] — 2026-05-17

### Fixed

**Actions tab: estPctMax linear → sqrt-of-time (alerts.ts)**
- `alerts.ts` close/manage fallback now uses `1 − √(remaining/entry)` instead of `(elapsed/entry)`
- `portfolio.ts` was fixed in v3.5.0 but the patch was not applied to `alerts.ts` — close and manage alerts were firing 4–6 days late on 45-DTE trades
- Fixes #87

**Wheel assignment breakeven: subtract credit received (ActionsTab.tsx)**
- Assigned positions now show `costBasis − credit_received` as breakeven instead of raw costBasis
- costBasis = assignment strike; true wheel BE = strike − put credit. Previous display overstated BE by the original put credit.
- Fixes #88

**PMCC and calendar strategies: "manage" → "roll" at 21 DTE (alerts.ts)**
- Added `ROLL_ELIGIBLE` set: `pmcc`, `calendar_spread`, `call_calendar_spread`, `put_calendar_spread`, `double_calendar`, `double_diagonal` now get `kind = "roll"` at 21 DTE
- Previously classified as `defined-risk → "manage"` — no "Roll →" button appeared for PMCC traders
- tastytrade: PMCC short call and calendar front month roll at 21 DTE exactly like naked premium
- Fixes #89

**Earnings date UTC parse inconsistency in earnings calendar (ActionsTab.tsx)**
- `buildErEntries` now parses dates as local noon (`"T12:00:00"`) matching `fmtErDate`
- ISO date strings parsed as UTC midnight caused off-by-one-day errors for non-UTC traders (e.g. EDT traders saw May 20 earnings as May 19)
- Fixes #90

**Max-loss alert: fallback for trades with no legs stored (alerts.ts)**
- Max-loss check now runs for all short-premium trades, not just those with leg JSON
- For manually-entered trades missing legs, approximates payoff via `short_strike` (naked put/call intrinsic)
- Previously, the `legs.length > 0` guard silently suppressed the 2× credit hard stop for all manually-entered positions
- Fixes #91

**Earnings 3–5d warning: add missing return (alerts.ts)**
- Added `return alerts` after the 3–5d earnings warning — prevents fall-through to the 21-DTE roll/manage alert
- A trade with earnings in 4d AND 20 DTE was getting both alerts simultaneously; tastytrade logic: earnings alert is exclusive
- Fixes #92

---

## [3.5.0] — 2026-05-16

### Fixed

**Theta proxy: linear → sqrt-of-time decay model (dashboard + unrealized P&L)**
- `portfolio.ts` unrealized P&L estimate now uses `credit × qty × 100 × (1 − √(remaining/entry))` — matches the extrinsic remaining model displayed in TradeDetail
- Linear `θ × days` understated early decay and overstated late; sqrt-of-time is correct for all DTE windows
- Falls back to linear only for debit spreads or trades missing `credit_received`/`entry_dte`

**θ/NLQ denominator: include unrealized P&L**
- Denominator changed from `balance` (realized only) to `balance + unrealized_pnl`
- tastytrade Net Liquidation = realized + unrealized; prior calculation overstated ratio when open positions were underwater
- Fixed in both `portfolio.ts` (snapshot) and `AppShell.tsx` (live re-computation)

**CSP BPR: add `margin_account` setting**
- New `margin_account: boolean` field in `AppSettings` (default: `false` — cash account)
- When `true`, CSP BPR = 20% notional (Reg T margin) instead of 100% (full cash collateral)
- Cascades through `calculateBpr`, `TradeForm`, `JournalTab`; persisted to settings DB

**Per-leg IV in live Greek re-estimation**
- Live Greek re-estimation in `AppShell.tsx` and `TradeDetail.tsx` now passes `leg.iv` to `estimateLegGreeks`
- ICs and strangles on SPY/SPX have 5–10 vol point put skew; single IV for all legs was materially misstating net delta

**"Unrlzd Est." card rename (dashboard)**
- KPI card previously labeled "θ Est." (conflicting with adjacent "Theta/Day" Greek card) renamed to "Unrlzd Est."
- Disambiguates estimated unrealized P&L from the theta Greek

**P50 and PIT: use live spot for open trades (TradeDetail)**
- Both metrics now use `spotForGreeks ?? t.underlying_price` — live price when available
- First-passage probability (P50) and terminal probability (PIT) are spot-dependent; using stale entry price was incorrect

**thetaPace unit mismatch fixed (TradeDetail)**
- `thetaPace` now computes `Math.abs(liveGreeks?.theta ?? (theta × qty × 100)) × remainingDte`
- Stored theta is per-share; comparison against dollar `maxProfit` without scaling caused this metric to always evaluate false

**Jade Lizard leg template corrected (TradeForm)**
- Template changed from `[SP, SC]` (which was a strangle) to `[SP, SC, LC]`
- Jade Lizard = short put + call spread (short call + long call wing at higher strike), not a naked short call

**Close targets: per-contract GTC price (TradeDetail)**
- CLOSE TARGETS section now shows `GTC $X.XX/contract` as primary, with total dollars as secondary
- Traders enter GTC orders at the broker using per-contract debit price, not total P&L

**Delta/theta/vega chips: consistent units when live Greeks unavailable (TradeDetail)**
- When `liveGreeks` is null, chips now scale stored per-share values: `t.delta × qty × 100`
- Previously showed raw per-share value (e.g. `−0.30`) instead of contract-level value (e.g. `−150`)
- T/V ratio numerator/denominator also normalized consistently

---

## [3.4.0] — 2026-05-15

### Added

**IVR × Strategy Heatmap (ChartsTab)**
- New heatmap in Charts tab: strategy rows × IVR band columns (0-10 through 90-100), win rate per cell
- Color-coded: green ≥65%, yellow ≥50%, red <50%; cells with <3 trades show "—" to avoid misleading small samples
- Instantly shows which strategies win/lose in different IV environments (tastytrade-standard view)

**IV Percentile field (TradeForm + TradeDetail)**
- IV Percentile input added to trade entry form alongside existing IV Rank field
- Saved to `iv_percentile` column; displayed in TradeDetail ENTRY tab
- Distinct from IV Rank: measures % of past 252 days where IV was below current level (tastytrade/TOS standard)

**RollAdvisor in Actions tab**
- Roll-kind alerts now include a "Roll →" action button that opens RollAdvisor directly from the Actions tab
- Cross-tab navigation via `journalRollId` state in AppShell: Actions → Journal → RollAdvisor in one click
- No longer requires living in the Journal tab to manage 21-DTE roll candidates

**Per-leg Greeks in TradeDetail RISK tab**
- RISK tab now shows Δ, Θ/d, Vega per individual leg using live `estimateGreeks()` computation
- Detects unbalanced hedges and asymmetric risk at a glance
- Falls back gracefully when spot/IV/DTE data is unavailable

**Assignment tracker in Actions tab**
- New ASN section lists all open positions where `was_assigned = true`
- Shows: ticker, assigned shares, cost basis, breakeven price, `view →` navigation
- Supports the wheel strategy cycle: put assignment → covered call tracking

### Fixed

**Schema: 22 missing columns added to `migrate_db()`**
- `migrate_db()` in `scripts/import.py` now covers all 22 columns that existed in live DBs but were missing from new-install migrations
- Includes: `pop_at_close`, `status`, `roll_count`, `was_assigned`, `iv_percentile`, `sector`, `next_earnings`, `bid_ask_spread_at_entry`, `fill_vs_mid`, `iv_before_earnings`, `iv_after_earnings`, `credit_width_ratio`, `gamma_at_close`, `cached_max_profit`, `cached_max_loss`, `closed_at_target`, `close_notes`, `underlying_price_at_entry`, `underlying_price_at_exit`, `assigned_shares`, `cost_basis`, `ex_dividend_date`
- New installs now match existing installs; no data loss on upgrade

**Trade pagination in Journal tab**
- Closed trades now load in pages of 200 (render-side slicing; no DB query changes)
- "Load more closed trades (N remaining)" button at bottom of closed list
- Prevents memory/render slowdown at 1,000+ closed trades; resets on filter change

---

## [3.3.0] — 2026-05-15

### Added

**Import History**
- Admin tab now shows a persistent **IMPORT HISTORY** section below the import panel
- Lists the last 10 import sessions in reverse-chronological order
- Each row: human-readable date/time, broker (ThinkorSwim / Schwab), source file name, `+N added` (green), `~N updated`, `⚠️ N` warnings badge
- Refreshes automatically after each import; shows "No imports yet." on first run
- Persisted in new `import_sessions` SQLite table (created by `init_db()`, populated at end of each import run)
- `fetchImportSessions()` added to `queries.ts` with graceful fallback for old DBs missing the table

---

## [3.2.0] — 2026-05-15

### Added

**Post-Import Review Panel**
- After broker import completes, a structured review panel appears below the raw log showing: ✅ N ADDED / 🔄 N UPDATED / ⏭ N SKIPPED / ⚠️ N WARNINGS summary bar
- ADDED TRADES table: Ticker, Strategy, DTE (days to expiry), Credit, Qty, and an `Edit →` button per trade
- UPDATED TRADES (closed/rolled): collapsible table with ticker, strategy, exit reason, P&L
- WARNINGS: collapsible list of import warnings
- Raw import log collapses to a `▶ RAW LOG` toggle when the review panel is visible
- `Edit →` on any added trade navigates to the Journal tab with that trade pre-selected for editing
- `scripts/import.py` emits a structured `---IMPORT_SUMMARY_JSON---` block at end of stdout; React parses it without any Rust changes

---

## [3.1.1] — 2026-05-15

### Fixed

**Import — CONDOR ROLL partial-close bug**
- `_build_tos_close_update`: added `all_legs_closed` check; `exitTime` and `exitReason='closed'` are now only emitted when every leg in the position has a `closePremium`. Previously, rolling one side of an Iron Condor (e.g. only the put spread) incorrectly marked the entire IC as `status=closed` because the function unconditionally set `exitReason='closed'` regardless of how many legs were actually transacted.
- CONDOR ROLL block: removed `new_t['exitReason'] = 'rolled'` from newly-opened roll legs. `exitReason` belongs only on closed trades; roll relationships are tracked via `rolled_from_id`.
- Print output now correctly labels CONDOR ROLL events as `Partial-close` (when the IC stays open) vs `Closed` (when all legs are filled).

**Import — Post-import sanity check**
- Added end-of-import scan: any `status=open` trade where ALL legs have `closePremium` is flagged with a `[WARN]` listing. Guards against future regressions from new code paths that skip the `all_legs_closed` guard.

---

## [3.1.0] — 2026-05-14

### Added

**Trade Import System**
- Admin tab: Broker Import panel — select ThinkorSwim or Schwab, pick the native export file, invoke Python importer, stream output to scrollable log
- Admin tab: Generic CSV Wizard (3-step: Upload → Map Fields → Import) — works with any broker CSV; downloadable theta-vault template for manual entry
- `src/lib/csvParse.ts`: CSV parser, template generator, field mapper, leg assembler
- Rust `run_import` command with Nuitka sidecar support (dev falls back to `python3 scripts/import.py`)
- `tauri-plugin-dialog` for native file picker

**Roll Advisor**
- Pressing `r` on an open trade now opens a Roll Advisor panel before the form
- Shows 3 roll scenarios (21 / 30 / 45 DTE) with estimated new credit, net roll cost/credit, and new theta/day — all computed via Black-Scholes at current spot + stored IV
- 30 DTE highlighted as tastytrade default
- Selecting a scenario pre-fills TradeForm with the target expiration; existing roll chain tracking unchanged

**Earnings Calendar — 14-day View**
- Actions tab now shows an EARNINGS panel at the top whenever open positions have earnings within 14 days
- Columns: Ticker (click → journal), Strategy, ER Date, Days to ER (red ≤3d / yellow ≤7d / green ≤14d), Expiry DTE, SAFE/EXPOSED badge
- SAFE = position expires before earnings; EXPOSED = position spans the ER date (IV crush / gap risk)
- Panel persists even when all other alerts are clear

**UI & UX**
- Admin tab: two-column layout (Settings left, Import Trades right)
- Admin tab: real `e`/`Esc` keyboard shortcuts for edit/cancel; status bar hint corrected
- Keyboard shortcuts panel reorganized (GLOBAL / ADMIN / JOURNAL sections)
- Web Worker for payoff curve computation (off main thread, `payoff.worker.ts`)

### Fixed

**Critical Calculation Bugs (Tom Sornoff audit)**
- **CF1 P50/PTouch**: Reflection-principle formula was using `N(+d)` for downside barriers → probabilities >1.0 clamped to 5%. Fixed to correct two-sided formula with full drift μ = r − σ²/2
- **CF2 Theta/Vega 100× inflation**: TradeForm preview multiplied by 100 twice (estimateGreeks already includes ×100). Fixed: stored `greeks.theta / 100` (per-share, consistent with Python importer)
- **CF3 Portfolio normalization**: Removed `Math.abs(th) > 15 ? th/100 : th` threshold that caused 100× error for |theta| ≤ $15 positions
- **CF4 TradeDetail OTM check**: Fixed 2-sided strategies to use inner breakeven range instead of single-sided check
- **Greeks ×100 unit mismatch**: Stored Greek values from Python importer are per-share (raw BS); all aggregation now applies ×100 contract multiplier. `buildPortfolioStats`, live fallback in AppShell, and TradeForm storage normalized
- **θ/NLQ always 0.00%**: `livePortfolioStats` was spreading `theta_netliq_ratio` from static stored theta (per-share, ~$1/day) while displaying live theta. Now recomputed from live theta / balance

**Build**
- Bundle chunk size warning eliminated: vendor chunks split (recharts+d3, react, @tauri-apps, app-playbook); all chunks <500 kB
- Tauri config: removed `"dialog": {}` empty object that caused startup panic (`invalid type: map, expected unit`)

### Changed

- `portfolio.ts` Greek aggregation: `× 100` on all stored Greeks (delta, theta, vega, gamma) for correct contract-dollar units
- TradeForm: stores Greeks as per-share (`greeks.xxx / 100`) for consistency with Python-imported trades
- Status bar Admin hint: `"e:edit  Ctrl+S:save  Esc:cancel"` → `"e:edit settings  Esc:cancel"` (Ctrl+S was unimplemented)

### Database

- Migration 6: `ALTER TABLE trades ADD COLUMN credit_width_ratio REAL`

---

## [3.0.1] — prior

Initial v3 patch release.

---

## [3.0.0] — prior

Initial v3 release.
