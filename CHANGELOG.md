# Changelog

All notable changes to Theta Vault are documented here.  
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/), versioning follows [Semantic Versioning](https://semver.org/).

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
