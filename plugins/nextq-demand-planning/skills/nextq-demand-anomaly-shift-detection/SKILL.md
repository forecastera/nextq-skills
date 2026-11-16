---
name: nextq-demand-anomaly-shift-detection
description: >-
  [Next Quarter Supply Chain Suite] Detects and triages demand anomalies as data arrives: spikes and dips (3xIQR on a
  robust decomposition remainder; size-only checks for intermittent items), level shifts (two-sided CUSUM on
  standardized forecast errors, confirmed by change-point and Welch tests), tracking-signal bias drift, variance
  shifts and demand-occurrence changes. Each alarm gets a deterministic cause (data error, censoring, explained event,
  distribution change, unexplained) and is routed to re-levelling, cleansing or a planner alert ranked by value at
  risk. Load when the user asks to detect anomalies, outliers, spikes, level shifts or structural breaks in demand,
  set up demand alerts, explain a sudden drop or surge, or decide whether to re-level a forecast. Inputs: demand
  history, forecast errors by lag, event calendars, in-stock, store counts, feed metadata. Triggers: "anomaly",
  "outlier", "spike", "level shift", "structural break", "demand alert".
metadata:
  author: "Next Quarter"
  organization: "Next Quarter (nextq)"
  suite: "Next Quarter Supply Chain Intelligence"
  version: "1.0.0"
---

# Demand Anomaly & Shift Detection
*Part of the **Next Quarter (nextq)** Supply Chain Skill Suite — Governed by SUPPLY_CHAIN_GOLD_STANDARD.md.*

## 1. Role & Objective
**Persona:** Demand Signal Monitoring Analyst.
**Decision supported:** Is this movement in demand noise, a one-off event, a data error, a censoring artifact, or a real and persistent change? Should the forecast be re-leveled, the history cleansed, or a planner alerted?

**Objective:**
- Run deterministic detectors on every new period (daily or weekly).
- Classify each alarm by cause and route it: level shifts to model re-level, point anomalies to `nextq-demand-history-cleansing`, censoring to `nextq-unconstrained-demand-reconstruction`, and unexplained alarms to `nextq-demand-exception-management`.
- Rank everything by value at risk, with full lineage.

## 2. Gold Standard Rules Enforced
| Rule | Inlined standard |
|---|---|
| **GS-D-04** Outliers | Detection on the remainder of a robust decomposition: flag values outside $[Q_1-3\,\text{IQR},\ Q_3+3\,\text{IQR}]$ (FPP3 §13.9 preferred rule; 1.5×IQR is the looser alternative). **Explain first** (promo, event, data error, bulk order). Known recurring causes become regressors, not removals. Replacements are logged with the original value and never silent. **Intermittent** series: no decomposition-based detection; use a size-only 3×IQR (or P99) check on non-zero sizes. The default engine is the same median-based robust decomposition as `nextq-demand-history-cleansing`; STL is an optional cross-check. |
| **GS-K-06** Tracking signal | $\mathit{TS}=\sum(A-F)/\text{MAD}$ over the rolling window (13 weeks / 6 months). **Positive = under-forecast.** Out of control at $\lvert TS\rvert>4$ [RoT]. This is the bias-drift detector; it shares one function with `nextq-forecast-accuracy-diagnostics`. |
| **GS-G-03** Exceptions | Each alarm lists cause, impact (units, value) and action. Rank by value at risk = \|units at risk\| × unit margin × ABC weight (**A = 3, B = 2, C = 1** [RoT]). Show the top N. Use the same case and hysteresis rules as the exception skill. |
| **GS-G-01** Human-in-the-loop | Planner responses that change the forecast are overrides: they need a reason code, are subject to materiality and the UP > 20% approval rule, and are scored by FVA. An automatic re-level is a model action and is logged as such, not as an override. |
| **GS-M-11** Censoring | Drops in periods with in-stock < 95% are **censoring**, not demand shifts. Route them to reconstruction and never re-level on them. |
| **GS-D-13** Lineage | Every alarm, classification and action is stored with before/after, rule ID and timestamp. |
| **GS-G-10 / GS-G-11** | Standard output; detectors are reproducible from the stored series snapshot. |

**Detector parameters [STD where noted, otherwise RoT]:**
- **CUSUM** (Page, tabular, two-sided) on standardized one-step errors $z_t=e_t/\hat\sigma$, with $\hat\sigma$ = RMSE on the training window.
  - $S^+_t=\max(0,S^+_{t-1}+z_t-k)$ and $S^-_t=\max(0,S^-_{t-1}-z_t-k)$. Alarm when $S>h$.
  - Defaults: $k=0.5$ (tuned to about a 1σ shift) and $h=5$. These are standard SPC choices: in-control average run length ≈ 465 periods for $h=5$ and ≈ 168 for $h=4$ (Montgomery's tables).
  - **A CUSUM alarm is a trigger, not a verdict:** it must be confirmed by the change-point test.
- **Change-point confirmation:**
  - Single mean change at the $\tau$ minimizing piecewise-constant SSE, with both segments ≥ 8 periods.
  - Confirmed only if Welch $p<0.01$ **and** |relative change| ≥ 20%.
  - Unconfirmed alarms are logged as `SHIFT_CANDIDATE` and re-tested each period.
- **Variance shift:** MAD of the last 13 errors / MAD of history > 2.
- **Occurrence shift** (intermittent, ADI ≥ 1.32): exact binomial test of the recent (13-period) non-zero count against the historical occurrence rate, $p<0.01$.
- **UoM / data-error heuristic:** the level ratio after/before within 5% of a pack multiple (6, 10, 12, 24, 48) or its inverse → `UOM_SUSPECT`.

## 3. Required Inputs & Data Schema
| Table | Columns | Notes |
|---|---|---|
| `demand_clean` | `item_id, location_id, period_start, demand_clean, demand_raw_base, period_state, in_stock_pct, promo_flag, event_flags` | from cleansing; raw kept for data-error checks |
| `forecast_errors` | `item_id, location_id, period_start, lag, A, F, e = A − F` | lag 1 for CUSUM; decision lag for TS |
| `segmentation` | `item_id, location_id, ADI, CV2, demand_class, abc, m (seasonal period)` | routing |
| `store_counts` | `item_id, location_id, period_start, stores_ranged` | distribution-change triage |
| `feed_meta` | `source, period_start, load_ts, row_count, expected_row_count` | feed-failure triage |
| `margins` | `item_id, unit_margin, unit_cost` | value at risk |

**Minimum:** ≥ 2 seasonal cycles for decomposition-based point detection (otherwise size-only); ≥ 26 error periods to estimate $\hat\sigma$; ≥ 16 periods for change-point confirmation.

## 4. Execution Workflow
1. **Ingest** the new period(s). If the feed row count is below 90% of expected [RoT] → `FEED_PARTIAL`, and all detectors pause for that source and period.
2. **Point anomalies** (GS-D-04):
   - Smooth/erratic series with ≥ 2 cycles: robust remainder → 3×IQR bounds.
   - Intermittent series or short history: size-only check on non-zero sizes.
3. **Bias drift:** $\lvert TS\rvert>4$ on the rolling window → `BIAS_DRIFT` (direction stated in words).
4. **Level shift:**
   - Run CUSUM on lag-1 errors.
   - On an alarm, run change-point confirmation on the last 52 periods (or all available ≥ 16): confirmed → `LEVEL_SHIFT`, otherwise `SHIFT_CANDIDATE`.
   - Reset the CUSUM after a confirmed shift.
5. **Variance shift** and, for intermittent items, **occurrence shift**.
6. **Triage** each alarm with deterministic cause precedence:

   | Precedence | Cause | Evidence |
   |---|---|---|
   | 1 | `DATA_ERROR` | `UOM_SUSPECT`, `FEED_PARTIAL`, or a raw vs. clean mismatch |
   | 2 | `CENSORING` | in-stock < 95% in the alarm window |
   | 3 | `EXPLAINED_EVENT` | promo or event flag in the window |
   | 4 | `DISTRIBUTION_CHANGE` | stores ranged changed by > 10% [RoT] |
   | 5 | `UNEXPLAINED` | none of the above |

7. **Route:**

   | Alarm / cause | Action |
   |---|---|
   | DATA_ERROR | data owner; hold forecasting for the series |
   | CENSORING | `nextq-unconstrained-demand-reconstruction` |
   | EXPLAINED_EVENT | no forecast action; the event is added as a regressor if it recurs (`nextq-external-driver-signal-integration`) |
   | DISTRIBUTION_CHANGE | re-level per store (rate-of-sale × stores) |
   | Confirmed LEVEL_SHIFT, UNEXPLAINED | model re-level: re-initialize the level at the post-break mean, or train on post-break data only if ≥ 1 cycle exists; otherwise add a step dummy. Log it as a model action. |
   | Point anomaly, UNEXPLAINED | `nextq-demand-history-cleansing` outlier treatment (replace via interpolation, logged) |
   | BIAS_DRIFT, VARIANCE or OCCURRENCE shift | `nextq-demand-exception-management` (occurrence decline on a TSB item → obsolescence review in `nextq-new-product-lifecycle-forecasting`) |

8. **Quantify** value at risk:
   - Level shift: $\lvert\text{after}-\text{before}\rvert$ × periods in the lead time.
   - Point anomaly: |remainder|.
   - Bias: $\lvert\sum(F-A)\rvert$ over the window.

   Multiply by margin and the ABC weight 3/2/1.
9. **Persist** with hysteresis (an alarm closes after 2 clear periods) and deterministic IDs `ANM-<period>-<item>-<loc>-<type>`.
10. **Output** (Section 6).

## 5. Verification Snippets
```python
import numpy as np, pandas as pd
from scipy import stats

# --- identical to nextq-demand-history-cleansing (GS-D-04 default engine) ---
def iqr_bounds(x, k=3.0):
    q1, q3 = np.nanpercentile(x, [25, 75]); iqr = q3 - q1
    return q1 - k * iqr, q3 + k * iqr

def robust_remainder(y, m):
    """Median-based robust decomposition: trend = centered rolling median (window m+1), seasonal = per-season
    median of detrended values (centered); remainder = y - trend - seasonal."""
    y = pd.Series(np.asarray(y, float)); w = m + 1 if m % 2 == 0 else m
    tr = y.rolling(w, center=True, min_periods=w // 2 + 1).median().interpolate(limit_direction="both")
    d = y - tr
    prof = d.groupby(np.arange(len(y)) % m).transform("median"); prof -= prof.mean()
    return y - tr - prof

def point_anomalies(y, m, adi):
    """Spikes / dips: 3xIQR on the remainder; intermittent (ADI >= 1.32) or < 2 cycles -> non-zero sizes only."""
    y = pd.Series(np.asarray(y, float))
    if adi >= 1.32 or len(y) < 2 * m:
        nz = y[y > 0]; lo, hi = iqr_bounds(nz)
        return ((y > 0) & ((y < lo) | (y > hi))).to_numpy()
    r = robust_remainder(y, m); lo, hi = iqr_bounds(r)
    return ((r < lo) | (r > hi)).to_numpy()

def tracking_signal(A, F, w=13, alpha=None):
    """APICS TS (identical to nextq-forecast-accuracy-diagnostics): rolling sum of (A-F) / MAD; positive = under-forecast."""
    e = pd.Series(np.asarray(A, float) - np.asarray(F, float))
    if alpha is None:
        rsfe = e.rolling(w, min_periods=w).sum(); mad = e.abs().rolling(w, min_periods=w).mean()
    else:
        rsfe = e.cumsum(); mad = e.abs().ewm(alpha=alpha, adjust=False).mean()
    return rsfe / mad.replace(0, np.nan)

def cusum(e, sigma, k=0.5, h=5.0):
    """Two-sided tabular CUSUM on standardized forecast errors z=e/sigma (Page). Returns first alarm index and side.
    k = 0.5 (detects ~1 sigma shifts), h = 5 [STD SPC defaults; h = 4 is more sensitive]."""
    z = np.asarray(e, float) / sigma; sp = sn = 0.0
    for t, v in enumerate(z):
        sp = max(0.0, sp + v - k); sn = max(0.0, sn - v - k)
        if sp > h: return t, "UP"
        if sn > h: return t, "DOWN"
    return None, None

def level_shift(y, min_seg=8, min_rel=0.20, alpha=0.01):
    """Single change point in the mean: tau minimizing piecewise-constant SSE with both segments >= min_seg.
    Confirm with Welch t-test p < alpha AND relative change >= 20% [RoT]."""
    y = np.asarray(y, float); n = len(y); best = (None, np.inf)
    for tau in range(min_seg, n - min_seg + 1):
        a, b = y[:tau], y[tau:]
        sse = ((a - a.mean()) ** 2).sum() + ((b - b.mean()) ** 2).sum()
        if sse < best[1]: best = (tau, sse)
    tau = best[0]
    if tau is None: return None
    a, b = y[:tau], y[tau:]
    p = stats.ttest_ind(a, b, equal_var=False).pvalue
    rel = (b.mean() - a.mean()) / abs(a.mean()) if a.mean() != 0 else np.inf
    return dict(tau=tau, before=a.mean(), after=b.mean(), rel_change=rel, p=p,
                confirmed=bool(p < alpha and abs(rel) >= min_rel))

def variance_shift(e, recent=13, ratio=2.0):
    """MAD(recent errors)/MAD(history) > 2 -> VARIANCE_SHIFT [RoT]."""
    e = np.asarray(e, float); h, r = e[:-recent], e[-recent:]
    mh = np.median(np.abs(h - np.median(h))); mr = np.median(np.abs(r - np.median(r)))
    q = mr / mh if mh > 0 else np.inf
    return q, q > ratio

def occurrence_shift(y, recent=13, alpha=0.01):
    """Intermittent: has the demand-occurrence probability changed? Exact binomial test of recent nonzero count
    against the historical rate."""
    y = np.asarray(y); h, r = y[:-recent], y[-recent:]
    p0 = (h > 0).mean(); k = int((r > 0).sum())
    p = stats.binomtest(k, len(r), p0).pvalue if 0 < p0 < 1 else np.nan
    return dict(p_hist=p0, p_recent=k / len(r), p=p, shift=bool(np.isfinite(p) and p < alpha))

def uom_jump(before, after, packs=(6, 10, 12, 24, 48), tol=0.05):
    """Data-error heuristic: level ratio within 5% of a pack multiple (or its inverse) -> UOM_SUSPECT [RoT]."""
    if before <= 0 or after <= 0: return None
    r = after / before
    for pk in packs:
        if abs(r / pk - 1) < tol or abs(r * pk - 1) < tol: return pk
    return None

def triage(flags):
    """Deterministic cause precedence: DATA_ERROR > CENSORING > EXPLAINED_EVENT > DISTRIBUTION_CHANGE > UNEXPLAINED."""
    for cause, key in (("DATA_ERROR", "uom_or_feed"), ("CENSORING", "in_stock_lt_95"), ("EXPLAINED_EVENT", "promo_or_event"),
                       ("DISTRIBUTION_CHANGE", "store_count_change")):
        if flags.get(key): return cause
    return "UNEXPLAINED"
```
**Test vector** (executed, separate seeds per test):
- **Point anomalies:** 156 weeks of seasonal demand (amplitude 20, noise σ = 3) with +60 injected at t = 80 and −50 at t = 120 → flagged exactly **[80, 120]**.
- **CUSUM:**
  - Errors with a +1.5σ shift at t = 30 → alarm **UP at t = 32**.
  - A 100-period in-control series (seed 23) raised a false alarm (DOWN at t = 71). Change-point confirmation rejected it (relative change −0.4%), so it stays a `SHIFT_CANDIDATE`. This shows why confirmation is mandatory.
- **Change point:** mean 100 → 130 at t = 30 → τ = 30, 99.35 → 130.47 (+31.3%), p = 2.3e−20, **confirmed**. A no-shift series → not confirmed.
- **Variance shift:** σ 1 → 3 over the last 13 periods → MAD ratio 2.57 (**flagged**). In-control → 1.12 (not flagged).
- **Occurrence shift:** occurrence 0.29 → 0.85 → p = 4.9e−5 (**flagged**).
- **UoM:** 10 → 121 and 120 → 10.1 → pack 12 (`UOM_SUSPECT`). 10 → 15 → none.
- **Triage:** promo + stockout → **CENSORING** (precedence). Nothing → UNEXPLAINED.

```sql
-- Censoring and event context for open alarms (triage inputs), last 4 periods around each alarm
SELECT a.alarm_id, a.item_id, a.location_id, a.period_start, a.alarm_type,
       MIN(d.in_stock_pct) < 0.95                                     AS in_stock_lt_95,
       BOOL_OR(d.promo_flag OR d.event_flags IS NOT NULL)             AS promo_or_event,
       MAX(s.stores_ranged) / NULLIF(MIN(s.stores_ranged),0) - 1 > 0.10 AS store_count_change,
       BOOL_OR(f.row_count < 0.9 * f.expected_row_count)              AS feed_partial
FROM alarms a
JOIN demand_clean d ON d.item_id=a.item_id AND d.location_id=a.location_id
                   AND d.period_start BETWEEN a.period_start - INTERVAL '3 weeks' AND a.period_start
LEFT JOIN store_counts s ON s.item_id=a.item_id AND s.location_id=a.location_id AND s.period_start=d.period_start
LEFT JOIN feed_meta f ON f.period_start=d.period_start
GROUP BY 1,2,3,4,5;
```

## 6. Standardized Output Format
```markdown
# Demand Anomaly & Shift Monitor — period <YYYY-WW> — run ANM_<run_id>

## Executive Summary
- <N> alarms: <p> point, <l> confirmed level shifts (<c> candidates), <b> bias drift, <v> variance, <o> occurrence.
- Causes: DATA_ERROR <d>, CENSORING <s>, EXPLAINED_EVENT <e>, DISTRIBUTION_CHANGE <g>, UNEXPLAINED <u>.
- <r> series re-levelled (model action); top VaR alarm <item> (<value>).
- Feeds: <n_partial> partial loads paused.

## Metadata (mandatory)
| Level | Grain | Time window | Error lag (CUSUM / TS) | Decomposition engine | Thresholds | Weighting | Snapshot |
|---|---|---|---|---|---|---|---|
| SKU-Location | WEEK | <start–end> | 1 / <decision lag> | median robust (STL cross-check) | 3×IQR; CUSUM k=0.5 h=5; |TS|>4; p<0.01 & ≥20% | VaR = units × margin × ABC 3/2/1 | <id> |

## KPI Table
| Metric | Value | Rule |
|---|---|---|
| Alarms per 100 series | | GS-G-03 |
| Share confirmed (level shift) | | n/a |
| Share by cause | | GS-D-04 |
| Median detection delay (confirmed shifts) | | n/a |
| False-alarm rate (closed as noise within 4 periods) | | n/a |

## Alarms
| ID | item | loc | type | period | magnitude | direction (words) | cause | confirmed | VaR | route | action | owner | status |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|

## Exceptions
| ID | Trigger (rule) | item | metric | value | action | owner |
|---|---|---|---|---|---|---|
| | LEVEL_SHIFT unexplained (GS-D-04) | | rel change | | re-level / review | Demand planner |
| | BIAS_DRIFT |TS|>4 (GS-K-06) | | TS | | exception queue | Demand planner |
| | UOM_SUSPECT / FEED_PARTIAL (GS-D-13) | | | | fix data | Data owner |
| | OCCURRENCE decline (GS-M-06) | | p | | obsolescence review | Lifecycle planner |

## Recommendations
1. …

## Data-Quality Log
| series | issue (short history → size-only, σ̂ unstable, missing store counts) | treatment |
|---|---|---|

## Reproducibility
Detector parameters, seeds (none needed; deterministic), engine, snapshot, code §5.
```

## 7. Edge Cases
- **Seasonal peaks** mistaken for spikes: the seasonal component removes them only if ≥ 2 cycles exist. With shorter history, use the size-only check and lower confidence.
- **Two shifts close together:** after a confirmed shift, CUSUM resets and the next change-point test uses only post-break data (still ≥ 8 periods per segment).
- **Gradual drift** (trend change, not a step): CUSUM alarms repeatedly with no confirmed step. After 3 unconfirmed alarms in 13 periods [RoT], flag `TREND_CHANGE` for model re-selection.
- **Planned events with unexpected magnitude** (a promo that failed): classed as EXPLAINED_EVENT, but the lift deviation is passed to `nextq-promotion-lift-cannibalization-planning`.
- **Calendar artifacts** (53rd week, Easter moving): check alignment with `nextq-time-bucket-calendar-alignment` before triage. A shift exactly one week long at a year boundary is usually calendar.
- **Daily streaming:** run detectors on the daily remainder with $m=7$. Confirm weekly before any re-level, so a single noisy day never re-levels a model.
- **Phantom inventory** (zero sales with stock on hand): an occurrence or level drop with in-stock = 100%. Route it as `PHANTOM_SUSPECT` to store operations; never re-level until counted.
