# Advanced Trend-Building Momentum Screener

This document defines an **early-stage directional momentum screener** designed to find stocks (and 24/7 assets like crypto) that are quietly building trend quality before obvious breakout acceleration.

---

## 1) Design Goals

### Detect
- Gradual accumulation/distribution
- High continuation probability (bar-to-bar directional persistence)
- Progressive relative strength (RS) improvement
- Momentum expansion with controlled pullbacks
- Trend quality and sustainability, not one-off spikes

### Avoid
- One-candle pumps
- News spikes that fade quickly
- High volatility / low directionality tapes
- Already overextended post-impulse names

---

## 2) Core Philosophy

Use a **composite rank** called **Trend Building Quality Score (TBQS)**.

TBQS combines 7 subscores:
1. Directional Smoothness
2. Continuation Probability
3. Progressive Relative Strength
4. Volume Participation Quality
5. Momentum Expansion with Pullback Control
6. Hold Quality (anti-fade behavior)
7. Trend Efficiency

Each subscore is normalized to `[0, 100]`, then weighted.

---

## 3) Feature Set and Formulas

> Notation: `P_t` close, `H_t` high, `L_t` low, `V_t` volume, `ATR_n` ATR over `n`, `EMA_n` EMA over `n`, `VWAP_s` session VWAP.

### 3.1 Directional Smoothness Score (DSS)
Measure whether price is moving in one direction with low zig-zag.

- `NetMove_n = |P_t - P_{t-n}|`
- `Path_n = Σ_{i=t-n+1..t} |P_i - P_{i-1}|`
- **Efficiency Ratio (ER)**: `ER_n = NetMove_n / Path_n` (Kaufman-style)
- **ATR-normalized drift**: `Drift_n = (P_t - EMA_n)/ATR_n`

Long-bias DSS example:
`DSS = 70*z(ER_20) + 30*z(clamp(Drift_20, -2, 2))`
(Then percentile-map to 0–100.)

Why: high ER implies less noisy, steadier trend-building.

### 3.2 Continuation Probability Score (CPS)
Estimate candle-to-candle continuation odds.

- Define directional bar state: `S_i = sign(P_i - P_{i-1})`
- Compute first-order Markov continuation:
  - `p_up = P(S_i=+1 | S_{i-1}=+1)`
  - `p_dn = P(S_i=-1 | S_{i-1}=-1)`
- For long mode: use `p_up`; for short mode use `p_dn`.
- Add run-length persistence bonus from expected streak length:
  - `E[run] = 1/(1-p)`.

`CPS = 80*z(p_dir) + 20*z(min(E[run], cap))`

Why: captures true directional follow-through, not random up/down churn.

### 3.3 Progressive Relative Strength Score (PRSS)
Measure RS trend vs benchmark index/sector.

- Relative line: `RL_t = P_t / Index_t`
- `RS_slope_fast = slope(log(RL), 20)`
- `RS_slope_slow = slope(log(RL), 60)`
- **RS acceleration**: `RS_acc = RS_slope_fast - RS_slope_slow`

`PRSS = 60*z(RS_slope_fast) + 40*z(RS_acc)`

Why: prefers names where outperformance is not only positive but improving.

### 3.4 Volume Participation Quality Score (VPQS)
Differentiate healthy participation from exhaustion spikes.

- **Signed pressure proxy** (bar-level):
  - `BuyFrac_i = (P_i - L_i) / max(H_i - L_i, eps)`
  - `VolDeltaProxy = Σ(V_i * (2*BuyFrac_i - 1))` over lookback
- **Consistency**: fraction of bars with above-median volume and directional close
- **Spike penalty**: penalize if top-1 bar volume dominates too much:
  - `SpikeRatio = max(V_i)/ΣV_i`

`VPQS = 50*z(VolDeltaProxy) + 30*z(Consistency) - 20*z(SpikeRatio)`

Why: sustained participation > single blow-off burst.

### 3.5 Momentum Expansion + Controlled Pullback Score (MEPS)
Capture “compression → expansion” while avoiding chaotic breakouts.

- `ROC_fast = ROC(P, 10)`, `ROC_slow = ROC(P, 30)`
- `ROC_acc = ROC_fast - ROC_slow`
- Compression regime signal: rolling range percentile low before breakout
  - `Comp = pct_rank((HH_20-LL_20)/ATR_20)` (lower is more compressed)
- Pullback depth in trend:
  - `PB = max_peak_to_trough_since_signal / ATR_20`

`MEPS = 45*z(ROC_acc) + 35*z((1-Comp)) + 20*z(1/PB)`

Why: targets early transition from tight base to controlled momentum.

### 3.6 Hold Quality Score (HQS)
Reject immediate reversals after pushes.

- `HoldVWAP = time_above_vwap_ratio` (for longs)
- `CloseLocation = mean((P_i - L_i)/(H_i - L_i + eps))`
- `FadePenalty = mean(max(0, intrabar_high_to_close_drop/ATR))`

`HQS = 50*z(HoldVWAP) + 30*z(CloseLocation) - 20*z(FadePenalty)`

Why: strong names hold gains near highs and above VWAP.

### 3.7 Trend Efficiency Score (TES)
Aggregate low-noise trend mechanics.

- EMA stack alignment score:
  - long: `EMA_8 > EMA_21 > EMA_50`
- Slope acceleration:
  - `d/dt slope(EMA_21)` positive and rising
- Noise filter:
  - `ATR_14 / P_t` too high => penalty

`TES = 40*EMA_align + 35*z(slope_acc) - 25*z(vol_noise)`

---

## 4) Composite Ranking Logic (TBQS)

Example long-side weighted model:

`TBQS = 0.20*DSS + 0.15*CPS + 0.15*PRSS + 0.15*VPQS + 0.15*MEPS + 0.10*HQS + 0.10*TES`

Then apply gates:
- Minimum liquidity (ADV, spread, trade count)
- Regime fit (market trend filter)
- Not overextended (`distance_from_EMA21 <= k*ATR`)
- Anti-spike filter (`single_bar_return <= threshold`)

Output ranked list + metadata:
- `TBQS`
- Subscores
- Stage label: `Build`, `Early Trend`, `Expansion`, `Mature`

---

## 5) Reducing Lag (Without Overfitting)

1. Use **slopes/accelerations** of features, not only levels.
2. Blend fast+slow windows (e.g., 10/30, 20/60) to keep responsiveness and stability.
3. Prefer **incremental updates** (streaming EMA, rolling sums) intrabar.
4. Use **event-based recompute** (new high, VWAP reclaim, volatility compression break).
5. Apply **Kalman/EMA smoothing** only to noisy sub-features, not final rank exclusively.

---

## 6) Avoiding False Momentum Signals

Use hard reject filters and confidence penalties:

- **One-bar impulse reject**:
  - if `|return_1bar| > x*ATR` and subsequent 3-bar hold < threshold, discard.
- **News shock decay filter**:
  - detect abnormal gap + volume spike + weak VWAP hold.
- **Chop filter**:
  - low ER + high ATR% + low continuation probability.
- **Late extension filter**:
  - overextension vs EMA21/50 and exhausted volume profile.
- **Cross-timeframe disagreement penalty**:
  - if 5m bullish but 30m/1h trend state bearish/flat.

---

## 7) Multi-Timeframe Model

For intraday equities:
- Trigger TF: 1m/2m/5m
- Structure TF: 15m
- Context TF: 1h + daily

For crypto 24/7:
- Trigger TF: 3m/5m
- Structure TF: 15m/1h
- Context TF: 4h

Scoring approach:
- Compute TBQS per timeframe
- Weighted fusion (example):
  - `TBQS_fused = 0.45*TBQS_trigger + 0.35*TBQS_structure + 0.20*TBQS_context`
- Require minimum alignment (e.g., 2 of 3 timeframes directionally consistent)

---

## 8) Real-Time Scanner Architecture

### Data Layer
- Ingest ticks/trades/quotes/bars (WebSocket + backup polling)
- Normalize and timestamp-align feed
- Maintain rolling feature store in memory

### Feature Engine
- Incremental computations:
  - EMA/VWAP/ATR/ER/ROC/RS/volume-pressure
- Stateful trend regime machine (`Build -> Early -> Expansion -> Mature -> Failure`)

### Ranking Engine
- Compute subscores and TBQS every bar close (and optionally mid-bar preview)
- Liquidity/slippage constraints
- Symbol-level cooldown to avoid repeated stale top-ranks

### Alerting/UI
- Ranked board with sparkline and subscore decomposition
- “New Entrant” boost for symbols crossing TBQS threshold first time in lookback
- Alert types:
  - Early Build Detected
  - Continuation Reinforcement
  - Trend Failure Risk

### Storage/Backtest
- Persist raw bars + feature snapshots + TBQS decisions
- Version scoring models for reproducibility

---

## 9) Backtesting Methodology

### Labeling / Target Definition
Evaluate if a detected signal leads to directional follow-through:
- Horizon returns: `R_{+5}, R_{+15}, R_{+30}` bars
- MFE/MAE in ATR units
- Continuation success if:
  - forward return > threshold
  - MAE bounded
  - no early failure pattern

### Validation Framework
- Walk-forward (rolling train/validation/test)
- Regime-segmented testing (trend, chop, high-vol news weeks)
- Universe-segmented testing (large/mid/small cap, sectors, crypto majors/alts)

### Metrics
- Precision@K on top-ranked symbols
- Information Coefficient (rank vs future return)
- Hit rate conditioned on TBQS decile
- Average MFE/MAE and expectancy
- Turnover and capacity constraints
- Slippage-adjusted PnL simulation

### Ablation
Remove one subscore at a time to measure contribution:
- `TBQS - DSS`, `TBQS - VPQS`, etc.
Keep only components that add robust out-of-sample edge.

---

## 10) Practical Defaults (Starting Point)

- Lookbacks: 10, 20, 30, 60 bars
- ATR length: 14
- EMA set: 8/21/50
- RS benchmark: SPY (or sector ETF), BTC for crypto alts
- TBQS alert threshold: 70+
- “Early trend” condition:
  - TBQS rising 3 consecutive bars
  - Not overextended (`dist_EMA21 < 1.8 ATR`)
  - VPQS and CPS both above 60

---

## 11) Implementation Notes

- Use robust normalization (percentile rank or robust z-score with MAD).
- Keep long/short symmetry by sign-flipping directional features.
- Use adaptive thresholds by volatility regime.
- Log every score component for explainability.

This screener is designed to surface names *before* obvious breakout headlines by rewarding consistent, efficient, and increasingly supported directional behavior.
