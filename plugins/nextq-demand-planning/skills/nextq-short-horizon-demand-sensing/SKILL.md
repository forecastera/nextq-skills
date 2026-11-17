---
name: nextq-short-horizon-demand-sensing
description: >-
  [Next Quarter Supply Chain Suite] Re-forecasts the next 0-N weeks (N set by deployment and replenishment lead time)
  from the freshest signals - daily POS or sell-out, open customer orders (order-book pickup curves) and in-stock data
  - blended with the baseline using lag-specific weights chosen by rolling-origin backtests and kept only where
  forecast value added is positive. Applies day-of-week profiles, shrunk and noise-gated run-rate ratios, as-of
  latency guards against leakage, and time-fence routing so sensed changes drive execution without rewriting the
  consensus plan. Load when the user asks to sense demand, adjust the near-term forecast from POS or orders, react to
  this week's sales, build a daily or short-horizon forecast, or test whether sensing adds value. Inputs: daily
  POS/orders, in-stock, baseline versions by lag, order book, calendar. Triggers: "demand sensing", "near-term
  forecast", "POS signal", "order pickup", "run rate".
metadata:
  author: "Next Quarter"
  organization: "Next Quarter (nextq)"
  suite: "Next Quarter Supply Chain Intelligence"
  version: "1.0.0"
---

# Short-Horizon Demand Sensing
*Part of the **Next Quarter (nextq)** Supply Chain Skill Suite — Governed by SUPPLY_CHAIN_GOLD_STANDARD.md.*

## 1. Role & Objective
**Persona:** Demand Sensing Analyst / Execution Planner (deployment and replenishment).
**Decision supported:** Given what sold and what was ordered in the last days, what is the best demand estimate for the next 0–N weeks at SKU-Location-Day/Week, and should replenishment or deployment react?

**Objective:**
- An execution forecast (`SENSED` version) for lags 1 to N days or weeks: baseline × sensed ratio and/or order-book pickup, blended per lag.
- Backtest proof (FVA ≥ 0 vs. baseline at each lag), a leakage-safe as-of design, and routing of material changes per the time fences.

## 2. Gold Standard Rules Enforced
| Rule | Inlined standard |
|---|---|
| **GS-D-11** Signal hierarchy | Prefer the signal closest to the consumer: **POS / sell-out → customer orders (requested date) → shipments / sell-in**. Shipments are a constrained proxy and are never used as demand when orders exist. Track the sell-in vs. sell-out gap (channel inventory) via `nextq-pos-sell-through-channel-analysis`. |
| **GS-M-09** ML / leakage controls | Time-ordered rolling-origin validation only (no random K-fold). Features must be known at forecast time $\mathit{t}-\ell$: POS is usable only after feed latency (default 1 day). Future promo and price only if planned and locked. Must beat the baseline on the primary metric (FVA ≥ 0). Loss aligned to decision (Poisson/Tweedie for counts). |
| **GS-M-15** Combination | Combining forecasts is robust. The blend $\mathit{F}=w_\ell F_{\text{sense}}+(1-w_\ell)F_{\text{base}}$ has weights chosen per lag $\ell$ on a grid {0, 0.25, 0.5, 0.75, 1}. Coarse grids avoid overfitting. Ties go to the **smaller** weight (less change). |
| **GS-M-16** Backtesting | Rolling origin, daily or weekly origins, ≥ 8 weeks of origins [RoT]. Score each lag separately, using only data available at each origin. |
| **GS-K-07** FVA by lag | $\text{FVA}_\ell=\text{WAPE}_\ell(\text{baseline})-\text{WAPE}_\ell(\text{blend})$, same items, same lag, same window. If $\text{FVA}_\ell\le0$ → $\mathit{w}_\ell=0$ (sensing switched off for that lag and segment). WAPE $=\sum\lvert A-F\rvert/\sum A$. |
| **GS-G-04** Time fences | Sensed values are an **execution** forecast. Within the frozen zone they feed deployment and replenishment (not the consensus plan). Changes > ±25% vs. consensus inside the frozen zone are raised as exceptions (`FROZEN_SWING`) for approval. The consensus is never overwritten. Every SENSED version is snapshotted by lag. |
| **GS-M-11** Censoring | Days with in-stock < 95% are excluded from ratio calculation (or reconstructed by `nextq-unconstrained-demand-reconstruction`). Stockout-truncated POS would bias the sensed ratio downward. |
| **GS-D-04 / GS-D-09** | Daily outliers: 3×IQR on the remainder after removing the baseline × DOW profile (bulk buys are explained, not ratioed). Daily → fiscal week aggregation uses the calendar table with an explicit week-ending day. |
| **GS-G-10 / GS-G-11** | Standard output; reproducible from the stored as-of snapshot. |

**Sensing parameters [RoT, configurable]:**
- Look-back for the ratio: 7 clean days (14 for slow movers).
- Shrinkage $\mathit{r}_s=1+(r-1)\frac{n}{n+n_0}$ with $\mathit{n}_0=7$ days. Clip $\mathit{r}_s\in[0.5,2.0]$.
- Noise gate: adjust only if $\lvert r-1\rvert>z\sqrt{\sum B}/\sum B$, with $\mathit{z}=1.64$ (Poisson approximation of the relative noise of a sum).
- Order-book pickup is used only if $\mathit{c}(h)\ge0.30$, capped at 3× booked.
- DOW profile from the last 8 full weeks.
- Horizon N = deployment / replenishment lead time + review period (typically 1–4 weeks). Sensing beyond the point where FVA turns 0 is not used.

## 3. Required Inputs & Data Schema
| Table | Columns | Grain |
|---|---|---|
| `pos_daily` | `item_id, location_id, date, units, in_stock_pct, price, promo_flag, load_ts` | SKU-Location-Day |
| `orders_book` (B2B) | `item_id, customer/location, requested_date, booked_qty, booking_date` | order line |
| `pickup_curve` | `segment, days_ahead h, share_booked c(h)` (median historical share of final qty booked h days before the requested date) | segment × h |
| `baseline_versions` | `item_id, location_id, target_date/week, lag, F_base, version (STAT/CONSENSUS), created_ts` | by lag |
| `promo_baseline` (if promo active) | from `nextq-promotion-lift-cannibalization-planning` | |
| `fiscal_calendar` | `date, fiscal_week, week_ending_day` | day |
| `time_fences` | `item/segment, frozen_days, slushy_days` | |

**Minimum:** 8 weeks of daily clean POS for the DOW profile; ≥ 8 weeks of stored baseline versions by lag for the backtest. A segment without a backtest runs with $\mathit{w}_\ell=0$ (baseline only).

## 4. Execution Workflow
1. **As-of snapshot:**
   - Forecast time $\mathit{t}_0$. Keep POS days with date ≤ $\mathit{t}_0$ − latency (1 day default) and orders booked ≤ $\mathit{t}_0$.
   - Partially loaded days (load_ts after cut-off, or volume < 50% of the DOW-expected volume with no closure event [RoT]) are excluded and flagged `PARTIAL_DAY`.
2. **Clean the recent window:**
   - Drop days with in-stock < 95% (or use reconstructed demand).
   - Detect daily outliers (3×IQR on $\mathit{A}_d - B_d$). Explained bulk orders are excluded from the ratio and logged.
3. **Baseline at daily grain:** $\mathit{B}_d=F^{\text{week}}_{\text{base}}\cdot\text{DOW}_d/7$ (DOW mean = 1), using the **promo baseline** when a promotion is active or planned in the window.
4. **POS ratio sensing:**
   - $\mathit{r}=\sum A_d/\sum B_d$ over the clean look-back days.
   - Apply the noise gate, then shrinkage and clipping (Section 2 parameters).
   - $\mathit{F}_{\text{sense},d}=B_d\cdot r_s$ for future days. The ratio applies to the level only; DOW and seasonality come from the baseline.
5. **Order-book sensing (B2B):**
   - $\mathit{F}_{\text{orders}}(h)=\text{booked}(h)/c(h)$ when $\mathit{c}(h)\ge0.30$ (cap 3× booked).
   - Where both POS and orders exist, $\mathit{F}_{\text{sense}}$ = the method with the better backtest WAPE at that lag (not averaged unless the backtest shows a gain).
6. **Backtest and weights:**
   - Replay steps 1–5 at each historical origin (≥ 8 weeks) using stored baseline versions.
   - Choose $\mathit{w}_\ell$ per lag and segment by minimum WAPE on the grid. If FVA ≤ 0, set $\mathit{w}_\ell=0$.
7. **Blend:** $\mathit{F}_{\text{final},\ell}=w_\ell F_{\text{sense}}+(1-w_\ell)F_{\text{base}}$. Aggregate days to fiscal weeks via the calendar.
8. **Time-fence routing:**
   - Inside the frozen zone, prepare the execution forecast for deployment / replenishment, for the user to load after review. If $\lvert F_{\text{final}}/F_{\text{consensus}}-1\rvert>25\%$, raise `FROZEN_SWING` to `nextq-demand-exception-management`.
   - Slushy / liquid zones: submit as a SENSING input to the next `nextq-consensus-demand-review` with evidence.
9. **Probabilistic band:**
   - Scale baseline quantiles by $\mathit{r}_s$ (level shift) for P10/P90.
   - Report coverage of last cycle's sensed intervals (GS-K-14, flag if off by > 5 pp).
10. **Output** (Section 6).

## 5. Verification Snippets
```python
import numpy as np, pandas as pd

def wape(A, F):
    A, F = np.asarray(A, float), np.asarray(F, float)
    return 100 * np.abs(A - F).sum() / np.abs(A).sum() if np.abs(A).sum() > 0 else np.nan

def dow_profile(daily, weeks=8):
    """Day-of-week index from the last `weeks` full weeks of clean daily POS (mean = 1)."""
    d = daily.iloc[-7 * weeks:]
    idx = d.groupby(d.index.day_of_week).mean()
    return idx / idx.mean()

def sensed_ratio(A_recent, B_recent, n0=7, clip=(0.5, 2.0), z=1.64):
    """Ratio of recent actuals to baseline over the same (clean, in-stock) days.
    Shrink toward 1 with n/(n+n0); clip [RoT]; gate on Poisson noise: |r-1| must exceed z*sqrt(sumB)/sumB."""
    A, B = np.asarray(A_recent, float), np.asarray(B_recent, float)
    n, sA, sB = len(A), A.sum(), B.sum()
    if sB <= 0:
        return 1.0, "NO_BASELINE"
    r = sA / sB
    noise = z * np.sqrt(sB) / sB
    if abs(r - 1) <= noise:
        return 1.0, "WITHIN_NOISE"
    r_s = 1 + (r - 1) * n / (n + n0)
    return float(np.clip(r_s, *clip)), "ADJUSTED"

def pickup_forecast(booked, pickup_share, min_share=0.3, cap=3.0):
    """Order-book sensing: final ~ booked / c(h), c(h) = historical share of final qty booked h days ahead.
    Not used if c(h) < min_share; ratio capped at `cap` x booked [RoT]."""
    if pickup_share < min_share:
        return np.nan
    return min(booked / pickup_share, cap * booked) if booked > 0 else 0.0

def blend(F_sense, F_base, w):
    return w * np.asarray(F_sense, float) + (1 - w) * np.asarray(F_base, float)

def choose_weights(bt, grid=(0.0, 0.25, 0.5, 0.75, 1.0)):
    """bt: backtest rows (lag, A, F_sense, F_base) from rolling origins. Per lag choose w minimizing WAPE;
    GS-K-07: if best blend does not beat the baseline (FVA <= 0) set w=0."""
    out = {}
    for lag, g in bt.groupby("lag"):
        base_err = wape(g.A, g.F_base)
        errs = {w: wape(g.A, blend(g.F_sense, g.F_base, w)) for w in grid}
        w_best = min(errs, key=lambda w: (errs[w], w))          # tie -> smaller weight (less change)
        fva = base_err - errs[w_best]
        out[lag] = dict(w=w_best if fva > 0 else 0.0, wape_base=base_err, wape_best=errs[w_best], fva=fva if fva > 0 else 0.0)
    return pd.DataFrame(out).T

def asof_filter(pos, forecast_time, latency_days=1):
    """Leakage guard (GS-M-09/M-16): only POS days fully available at forecast time."""
    return pos[pos.index <= (forecast_time - pd.Timedelta(days=latency_days)).normalize()]
```
**Test vector** (verified, seed 7):
- **DOW profile:** 8 weeks of Poisson POS (mean 100 × [0.8, 0.9, 1.0, 1.0, 1.1, 1.3, 0.9]) → index [0.831, 0.893, 1.033, 1.009, 1.093, 1.245, 0.897].
- **Ratio sensing:**
  - 7 days at 130 vs. 100 → r = 1.30, shrunk to **1.15** (ADJUSTED).
  - 103 vs. 100 → **1.0** (WITHIN_NOISE; gate 1.64·√700/700 = 0.062).
  - 400 vs. 100 over 14 days → clipped to **2.0**.
- **Pickup:**
  - 60 booked at c = 0.5 → **120**.
  - c = 0.2 is below 0.30 → not used (NaN).
  - 10 booked at c = 0.25 (with min share 0.2) → 40, capped to **30**.
- **Backtest** (40 origins; true level +20% vs. baseline):

  | Lag (days) | w | WAPE base → blend | FVA (pts) |
  |---|---|---|---|
  | 1 | 1.00 | 16.79 → 8.17 | +8.62 |
  | 7 | 1.00 | 17.53 → 6.79 | +10.73 |
  | 14 | 0.75 | 17.24 → 10.74 | +6.50 |
  | 28 (noisy signal) | 0.00 | 17.29 → 17.29 | sensing off |

- **As-of guard:** at 10 Nov 06:00 with 1-day latency, the last usable POS day is **9 Nov**.

```sql
-- Clean look-back ratio per item-location (7 in-stock days before the as-of cut-off)
WITH lb AS (
  SELECT p.item_id, p.location_id, p.date, p.units,
         b.f_base_week * d.dow_index / 7.0 AS b_day
  FROM pos_daily p
  JOIN baseline_daily_profile d ON d.item_id = p.item_id AND d.location_id = p.location_id AND d.dow = EXTRACT(DOW FROM p.date)
  JOIN baseline_versions b ON b.item_id = p.item_id AND b.location_id = p.location_id
                          AND b.fiscal_week = (SELECT fiscal_week FROM fiscal_calendar c WHERE c.date = p.date)
                          AND b.created_ts <= :asof
  WHERE p.date BETWEEN :asof_date - 7 AND :asof_date - 1          -- 1-day latency
    AND p.in_stock_pct >= 0.95 AND p.load_ts <= :asof)
SELECT item_id, location_id, COUNT(*) AS n_days, SUM(units) / NULLIF(SUM(b_day),0) AS raw_ratio
FROM lb GROUP BY 1,2;
```

## 6. Standardized Output Format
```markdown
# Demand Sensing — as-of <YYYY-MM-DD HH:MM> — run SENSE_<run_id>

## Executive Summary
- <N> item-locations sensed; <n_adj> adjusted (median r_s <r>), <n_noise> within noise; sensing active for lags <1..k> (FVA > 0).
- Next <N> weeks: sensed <units> vs baseline <units> (<x>%); <n_swing> FROZEN_SWING exceptions raised (>±25% in frozen zone).
- Backtest FVA (WAPE pts) lag 1 / 7 / 14: <a> / <b> / <c>. Confidence: <clean-day share>%, <partial days excluded>.

## Metadata (mandatory)
| Level | Grain | Signal | As-of / latency | Look-back | Lags scored | Backtest window (origins) | Weighting | Baseline version |
|---|---|---|---|---|---|---|---|---|
| SKU-Location | DAY (rolled to fiscal WEEK) | POS / orders | <ts> / 1 day | 7 clean days | 1–N | <start–end> (<n>) | volume-weighted WAPE | STAT / CONSENSUS <id> |

## KPI Table (backtest, per lag)
| Lag | Segment | N | WAPE base % | WAPE sensed % | FVA pts | w chosen | Bias % sensed (+=over) | PI coverage (nominal 80%) |
|---|---|---|---|---|---|---|---|---|

## Sensed Forecast (execution)
| item | loc | week | F_base | r_s | status | F_orders | w | F_final | Δ vs consensus % | zone | route |
|---|---|---|---|---|---|---|---|---|---|---|---|

## Exceptions
| ID | Trigger (rule) | item | loc | metric | value | action | owner |
|---|---|---|---|---|---|---|---|
| | FROZEN_SWING >25% (GS-G-04) | | | Δ% | | approve deployment change | S&OP / execution |
| | Ratio clipped at 0.5 / 2.0 | | | r | | investigate (event? data?) | Demand planner |
| | FVA ≤ 0 all lags (GS-K-07) | | | | | sensing off for segment | Analyst |
| | PARTIAL_DAY / stale feed (GS-M-09) | | | | | fix feed | Data team |
| | Clean days < look-back (GS-M-11) | | | | | use reconstruction / baseline | |

## Recommendations
1. …

## Data-Quality Log
| item / feed | issue (latency, partial day, censored days, outliers removed, missing baseline version) | treatment |
|---|---|---|

## Reproducibility
As-of timestamp, latency, look-back, n0, clip, z, grid, backtest origins, baseline version IDs, code §5.
```

## 7. Edge Cases
- **Promotion in the look-back or horizon:** ratio against the **promo-adjusted** baseline. A promo spike against a plain baseline would wrongly raise the level for post-promo days (where a dip is expected).
- **Moving holiday or calendar shift** (Easter, Thanksgiving week): the DOW profile excludes holiday weeks. The baseline must carry the event dummy; otherwise flag `CALENDAR_EVENT_UNMODELED` and freeze $\mathit{r}_s=1$ for those days.
- **Stockouts in the look-back** reducing clean days below 4: use reconstruction output or fall back to $\mathit{w}=0$ (`INSUFFICIENT_CLEAN_DAYS`).
- **New items** (no baseline): sensing does not apply; use early-read re-scaling in `nextq-new-product-lifecycle-forecasting`.
- **Intermittent items** (ADI > 1.32): the ratio method is unstable. Use weekly look-back of 4+ weeks and $\mathit{n}_0=28$, or turn sensing off unless the backtest shows FVA > 0.
- **Retailer POS shared weekly only:** grain becomes week and look-back 2–4 weeks; the latency guard applies to the share date.
- **Order cancellations / duplicates in the order book:** net cancellations by booking date and deduplicate by order line ID before pickup.
- **Feed outage:** if no POS for > 2 days, the last sensed ratio decays to 1 over 7 days [RoT] and the run is flagged `STALE_SIGNAL`.
