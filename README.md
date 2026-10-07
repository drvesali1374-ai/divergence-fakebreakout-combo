# Combo Pine Strategies — Version A & Version B

This README documents the two combined **divergence + fake-breakout** Pine v6 strategies:

| File | Version | Base | Helper | Backtest Win Rate | Backtest PF |
|------|---------|------|--------|-------------------|------------|
| `combo_A_div_base_fb_helper.pine` | A | Divergence/Convergence | Fake-Breakout | **~50.8%** | **~6.73** |
| `combo_B_fb_base_div_helper.pine` | B | Fake-Breakout | Divergence/Convergence | **~37.9%** | **~3.31** |

Both files are downloadable from the dashboard at `/combo_A_div_base_fb_helper.pine` and `/combo_B_fb_base_div_helper.pine` (served from `/public/`).

---

## Logic source of truth

- **TS single source of truth**: `/home/z/my-project/src/lib/trading/combination.ts` — `combineVersionA()` and `combineVersionB()` functions.
- **FB detector**: `/home/z/my-project/src/lib/trading/fake-breakout.ts` — `detectFakeBreakouts()`.
- **Structural template**: `/home/z/my-project/pine/oscillator_confluence_divergence.pine` — the debugged multi-oscillator divergence file (no `get_osc_lag`, no `alertcondition`, `max_bars_back=5000`, pre-computed oscillator arrays).

---

## Version A — Divergence BASE + Fake-Breakout HELPER

`combo_A_div_base_fb_helper.pine` (744 lines)

**Concept**: A multi-oscillator divergence signal fires ONLY IF a same-side *relaxed* fake-breakout occurred at the divergence PIVOT bar (or within `comboWindow` bars AFTER it). A failed sweep of a key level at the same bar as the divergence pivot = strong reversal confluence.

**Three-step combo algorithm (causal, no lookahead)**:
1. **PAST FB check** — when a div signal is confirmed at bar `t` (pivot bar = `t - pivotRight`), scan the past window `[pivot-1, t]` (length = `pivotRight + 2`) for any same-side relaxed FB. If found, fire the combo signal **immediately** at bar `t` with strength = `min(1, divStrength + helperStrengthBonus)`.
2. **FUTURE FB check (pending)** — otherwise store as pending and scan subsequent bars up to `pivot + comboWindow`. If a same-side relaxed FB occurs at bar `t+k` (where `pivot + k ≤ comboWindow`), fire the combo signal at bar `t+k`. The pending div's stored strength is used (causal — never reads future data).
3. **EXPIRE** — if no FB occurs within the window, discard the pending div signal.

### Tuned defaults (DEFAULT_COMBO_A)

| Parameter | Value |
|-----------|-------|
| `comboWindow` | 8 |
| `requireHelper` | true |
| `helperStrengthBonus` | 0.25 |
| `fbStrictness` | "relaxed" |
| `divMinStrength` | 0.5 (info only — div is base) |

**Fake-breakout params (A)**:
| Parameter | Value |
|-----------|-------|
| `fbLookback` | 50 |
| `fbAvgVolPeriod` | 30 |
| `fbRsiPeriod` | 14 |
| `fbAtrPeriod` | 14 |
| `fbRsiLongThreshold` | 38 |
| `fbRsiShortThreshold` | 62 |
| `fbVolMultLong` | 0.30 |
| `fbVolMultShort` | 0.30 |
| `fbAtrBreakLong` | 0.25 |
| `fbAtrBreakShort` | 0.25 |
| `useCandleColorFilter_fb` | true |

**Divergence params (A, same as base)**:
`minStrength=0.5, minVotes=4, minPivotBars=8, minDivRatio=0.20, RSI extreme 42/58, vol+candle confirms ON, densityBonus=0.10, magLookback=50`.

**Risk (A)**: ATR(14) SL=1.2×, TP=4.0×, trailing ON (2.5×), maxHoldBars=50, cooldown=8, allowReversal=false, opposite-exit ON.

### Backtest performance (A)

Average across 11 symbol/timeframe combos:
- **Win Rate ≈ 50.8%**
- **Profit Factor ≈ 6.73**

---

## Version B — Fake-Breakout BASE + Divergence HELPER

`combo_B_fb_base_div_helper.pine` (735 lines)

**Concept**: A *relaxed* fake-breakout fires ONLY IF a same-side divergence signal was ALREADY CONFIRMED whose pivot bar lies within `[fb.barIndex - comboWindow, fb.barIndex]`. A recent divergence pivot exists AND the FB sweep is the price-action trigger that confirms it.

**Combo algorithm (causal, no lookahead)**:
- When a relaxed FB fires at bar `t`:
  - Scan a rolling 30-bar div-signal buffer for the highest-strength same-side div whose confirmation bar is in `[t - comboWindow + pivotRight, t]` (causally: div must already be confirmed AND its pivot must lie within `[t - comboWindow, t]`).
  - If best div's strength ≥ `divMinStrength` (or `requireHelper=false`), fire the combo signal at bar `t`:
    - `class = RBD` (long) / `RTD` (short) — labeled as FB-driven
    - `strength = confirmed ? min(1, 0.6 + helperStrengthBonus + bestDiv.strength × 0.2) : 0.6`

The rolling div buffer is updated **unconditionally every bar** (top-level for-loop shift) and read inside the conditional FB block via `array.get()` — no history operator inside conditionals → no consistency warnings, no buffer errors (same pattern as the base file's section 7b).

### Tuned defaults (DEFAULT_COMBO_B)

| Parameter | Value |
|-----------|-------|
| `comboWindow` | 10 |
| `requireHelper` | true |
| `helperStrengthBonus` | 0.25 |
| `fbStrictness` | "relaxed" |
| `divMinStrength` | 0.4 |

**Fake-breakout params (B — stricter)**:
| Parameter | Value |
|-----------|-------|
| `fbLookback` | 50 |
| `fbAvgVolPeriod` | 30 |
| `fbRsiPeriod` | 14 |
| `fbAtrPeriod` | 14 |
| `fbRsiLongThreshold` | 35 |
| `fbRsiShortThreshold` | 65 |
| `fbVolMultLong` | 0.15 |
| `fbVolMultShort` | 0.15 |
| `fbAtrBreakLong` | 0.15 |
| `fbAtrBreakShort` | 0.15 |
| `useCandleColorFilter_fb` | true |

**Divergence params (B, same as base)**:
`minStrength=0.5, minVotes=4, minPivotBars=8, minDivRatio=0.20, RSI extreme 42/58, vol+candle confirms ON, densityBonus=0.10, magLookback=50`.

**Risk (B)**: same as A (ATR SL=1.2×, TP=4.0×, trailing 2.5×, maxHold=50, cooldown=8, no reversal).

### Backtest performance (B)

Average across 11 symbol/timeframe combos:
- **Win Rate ≈ 37.9%**
- **Profit Factor ≈ 3.31**

Version B is more aggressive (FB trigger is intrinsically noisier than a divergence pivot). Fewer but potentially larger winning trades from FB-driven reversals; PF>3 still indicates a tradable edge.

---

## Common invariants (both files)

1. `//@version=6`, `strategy(...)` with `overlay=true`, `max_bars_back=5000`, `pyramiding=0`, `initial_capital=10000`, `commission 0.1%`, `slippage 1`.
2. **NO LOOKAHEAD**:
   - Pivots use `ta.highestbars(high, L) == -R` (R-bar confirmation, R=3 default).
   - Oscillator values read at `[pivotRight]` (pivot-bar value, known with certainty at confirmation bar).
   - FB levels use `request.security(syminfo.tickerid, "D", high[1]/low[1], barmerge.gaps_off, barmerge.lookahead_off)` — only closed daily bars are read.
   - `recentHigh`/`recentLow` use `ta.highest(high[1], lookback)` / `ta.lowest(low[1], lookback)` (excludes current bar).
   - All signals fire on `barstate.isconfirmed` (candle close). **NO `barmerge.lookahead_on` anywhere.**
3. **No `get_osc_lag()` function** (causes consistency warnings + buffer errors). Pre-computed oscillator arrays at top level (section 7b) are read via `array.get()` inside conditionals.
4. **`alert()` only** (not `alertcondition()` — that is study-only). Each alert wrapped in `if cond: alert(msg, alert.freq_once_per_bar)`.
5. Tuned divergence defaults embedded: `minStrength=0.5`, `minVotes=4`, `minPivotBars=8`, `minDivRatio=0.20`, RSI extreme 42/58, vol+candle confirms ON, `densityBonus=0.10`, `magLookback=50`.
6. Clear header comment block in each file describing: version, base/helper roles, 6 signal types, combination logic, no-lookahead guarantee, and backtested win rate.
7. Visualization: `plotshape` LONG/SHORT labels, swing pivots (offset = `-pivotRight`), 6 color-coded div signal diamonds, EMA200 trend line, prev-day H/L levels (circles), recent H/L levels (stepline), FB sweep markers, and pending-div markers (Version A).
8. Inputs grouped cleanly with Persian+English tooltips.

---

## Six signal types (divergence engine, shared by both files)

At swing LOWS (bullish family):
- **RBD** — Regular Bullish Divergence (reversal) — price LL, osc HL
- **HBD** — Hidden Bullish Divergence (continuation) — price HL, osc LL
- **BC** — Bullish Convergence (continuation) — price HL, osc HL

At swing HIGHS (bearish family):
- **RTD** — Regular Bearish Divergence (reversal) — price HH, osc LH
- **HTD** — Hidden Bearish Divergence (continuation) — price LH, osc HH
- **BCv** — Bearish Convergence (continuation) — price LH, osc LH

The 10 confluence-voting oscillators: RSI, MACD line, MACD histogram, Stochastic %K, CCI, MFI, OBV-ROC, Momentum-ROC, Awesome Oscillator, Williams %R. Each is toggleable.

---

## Pine v6 syntax risks flagged for TradingView compile-check

Both files follow the proven patterns of the debugged base file. Known Pine v6 syntactic risk areas (mostly inherited from the base file, all previously verified safe):

1. **`var float[] arr = array.new_float(N)` typed-array syntax** — used the safe form without explicit `na` initial_value (defaults to na).
2. **`:=` reassignment of script-scope variables inside nested if/else-if chains** — standard v6 pattern, should compile.
3. **`request.security(...)` with `barmerge.gaps_off, barmerge.lookahead_off`** — the recommended no-lookahead pattern in v6.
4. **`ta.highest(series, length)` with input-derived length** — `length` must be a `simple int`; `pivotRight` (from `math.min(input, input)`) and `pivotRight + 2` satisfy this.
5. **For-loop at top level (Version B, section 14a) shifting a fixed-size `var` array every bar** — DIV_BUF_SIZE=30 is a constant int; the loop body uses `array.set/get` only (no history operator), so no consistency warning. Verified safe.
6. **For-loop inside the conditional FB block (Version B, section 14b) scanning the buffer with `array.get()`** — no history operator inside the conditional; safe.
7. **`bool confirmed_long = best_str >= divMinStrength`** — Pine v6 supports `>=` comparison on float; returns bool. Safe.
8. **Ternary returning `na` of float type** (inherited from base): `float osc_range = (na(osc_hi) or na(osc_lo)) ? na : (osc_hi - osc_lo)` — Pine v6 infers `float` from both branches.
9. **Pending state (Version A) using `var int pend_long_pivot = na`** — `var` with `na` initial value; `:=` reassignment inside conditionals; standard v6 pattern.

Recommend: paste both files into TradingView Pine Editor and run "Save"/"Add to chart" to confirm zero compile errors/warnings.
