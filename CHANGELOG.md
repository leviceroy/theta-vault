# Changelog

All notable changes to Theta Vault are documented here.  
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/), versioning follows [Semantic Versioning](https://semver.org/).

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
