---
name: nextq-forecast-accuracy-diagnostics
description: >-
  [Next Quarter Supply Chain Suite] Measures and diagnoses demand forecast performance with canonical, lag-aware KPIs:
  WAPE, MAPE, sMAPE, MASE, RMSSE, forecast bias, tracking signal, pinball loss, interval coverage, and accuracy by
  lag, level and segment, with a metric-misuse guard and exception flags. Load when the user asks how accurate a
  forecast is, wants a forecast accuracy or bias report or scorecard, asks to compare models, versions or planners,
  sees MAPE that looks wrong, needs tracking-signal alerts, or wants accuracy at lag 1/2/3 or by SKU, customer,
  region. Inputs: frozen forecast snapshots (item, location, target period, creation period/lag, version) and
  unconstrained actuals at SKU-Location-Week/Month. Triggers: "forecast accuracy", "WAPE", "MAPE", "bias", "tracking
  signal", "forecast error report", "which model is more accurate", "accuracy by lag", "forecast KPI dashboard".
metadata:
  author: "Next Quarter"
  organization: "Next Quarter (nextq)"
  suite: "Next Quarter Supply Chain Intelligence"
  version: "1.0.0"
---

# Forecast Accuracy Diagnostics
*Part of the **Next Quarter (nextq)** Supply Chain Skill Suite — Governed by SUPPLY_CHAIN_GOLD_STANDARD.md.*

## 1. Role & Objective
**Persona:** Forecast Performance Analyst (demand planning center of excellence).
**Decision supported:** How good and how biased is each forecast version, at the lag that drives decisions? Where is error concentrated? Which series need intervention?

**Objective:** Produce a deterministic, reproducible accuracy and bias scorecard with mandatory metadata (lag, level, window, weighting). It includes a correct metric choice per series type, tracking-signal control, distributional accuracy, and a ranked exception list. It is the **measurement engine** reused by FVA (`nextq-forecast-value-added-analysis`) and exception management.

## 2. Gold Standard Rules Enforced
Notation: $\mathit{A}_t$ = unconstrained actual, $\mathit{F}_t = F_{t\mid t-\ell}$ = forecast frozen at lag ℓ, $\mathit{e}_t=A_t-F_t$ (**GS-M-01**).

| Rule | Exact definition (inlined) | Use / guardrail |
|---|---|---|
| **GS-M-01** | Every number states **lag ℓ, level, time window, weighting**. A forecast without a stored creation date (unknown lag) is **not scored**. | Default decision lag = cumulative lead time or frozen zone [RoT]. |
| **GS-K-01** WAPE (= WMAPE, IBF) | $\text{WAPE}=\dfrac{\sum\lvert A_t-F_t\rvert}{\sum\lvert A_t\rvert}\times100$. Value-weighted variant: weight by price, $\frac{\sum p\lvert A-F\rvert}{\sum p\lvert A\rvert}$. | **Primary portfolio** metric. Undefined if $\sum A=0$. "Accuracy" $=\max(0,100-\text{WAPE})$ only when labeled. |
| **GS-K-02** MAPE | $\frac{100}{n}\sum\left\lvert\frac{A_t-F_t}{A_t}\right\rvert$ over $\mathit{A}_t>0$ only. | Legacy/communication only. **Prohibited** for intermittent series (ADI ≥ 1.32). Count and disclose excluded zeros. |
| **GS-K-03** sMAPE | $\frac{100}{n}\sum\frac{2\lvert A_t-F_t\rvert}{\lvert A_t\rvert+\lvert F_t\rvert}$, with 0/0 := 0. | Reported only for external comparability; never used for decisions (Hyndman & Koehler 2006). |
| **GS-K-04** MASE / RMSSE | $\mathit{q}_j=\dfrac{e_j}{\frac{1}{T-m}\sum_{t=m+1}^{T}\lvert y_t-y_{t-m}\rvert}$; $\text{MASE}=\text{mean}\lvert q_j\rvert$; $\text{RMSSE}=\sqrt{\text{mean}(e_j^2)/\frac{1}{T-m}\sum(y_t-y_{t-m})^2}$. $\mathit{m}=1$ for non-seasonal and intermittent series. Training window = history before the forecast **origin**. | **Primary per-series** metric. < 1 beats the in-sample one-step (seasonal) naïve. Scale ≈ 0 → `SCALE_ZERO` and fall back to WAPE. RMSSE is used for intermittent series (MAE-type metrics reward zero forecasts). |
| **GS-K-05** Bias | $\text{Bias\%}=\dfrac{\sum(F_t-A_t)}{\sum A_t}\times100$ (**positive = over-forecast**); $\text{ME}=\text{mean}(e_t)$. | Compute at decision level and roll up. Also report **absolute-bias-weighted** roll-up $\frac{\sum_i\lvert\sum_t(F-A)\rvert_i}{\sum A}$ because offsets hide at aggregate. State the direction in words. |
| **GS-K-06** Tracking signal (APICS) | $\mathit{TS}_t=\dfrac{\sum_{i=t-w+1}^{t}(A_i-F_i)}{\text{MAD}_t}$, where $\text{MAD}_t=\frac1w\sum\lvert A_i-F_i\rvert$ over the same rolling window $\mathit{w}$ (default 13 weeks / 6 months [RoT]). Smoothed alternative: $\text{MAD}_t=\alpha\lvert e_t\rvert+(1-\alpha)\text{MAD}_{t-1}$, α = 0.1. | **Out of control** at $\lvert \mathit{TS}\rvert>4$ [RoT; 3.75–6 used]. **Positive TS = under-forecasting** (opposite sign to Bias%). |
| **GS-K-07** FVA (hook) | $\text{FVA}_s=\text{Err}(\text{prior step})-\text{Err}(s)$, same metric, items, lag, window. | Computed here on request. The stair-step and significance testing live in `nextq-forecast-value-added-analysis`. |
| **GS-K-13** Reporting rules | Decision level and aggregates reported **separately**. Volume-weighted WAPE + **median** MASE across items. Disclose N items, N periods, % reconstructed actuals, % excluded zeros. | Aggregate accuracy always looks better (errors cancel), so never present it as SKU accuracy. |
| **GS-K-14** Probabilistic | Pinball $\mathit{Q}_{p,t}=2(1-p)(f_{p,t}-y_t)$ if $\mathit{y}_t<f_{p,t}$, else $2p(y_t-f_{p,t})$. Winkler $\mathit{W}_\alpha=(u-l)+\frac2\alpha(l-y)\mathbb 1[y<l]+\frac2\alpha(y-u)\mathbb 1[y>u]$. Coverage $=\frac{\#\{A_t\in \mathit{PI}\}}{n}$. | Flag coverage off nominal by > 5 pp [RoT]. Use CRPS / skill scores vs. naïve for cross-series comparison. |
| **GS-M-16** | Score **per lag**. Never average different horizons without showing each lag. | Rolling-origin backtests and production snapshots are both allowed; label which. |
| **GS-M-11 / GS-D-02** | Score against **unconstrained** actuals (cleansed and reconstructed). Report sales-basis (constrained) accuracy separately. Exclude structural-zero periods. | If > 30% of actuals are reconstructed → `LOW_CONFIDENCE`. |
| **GS-G-03** Exception triggers [RoT] | \|TS\| > 4; \|Bias%\| > 10% over the last 3 cycles (15% for intermittent); MASE > 1.0 on the rolling window; FVA of final vs. naïve < 0; forecast change vs. last cycle > ±25% inside the frozen zone; coverage off > 5 pp; > 30% reconstructed actuals. | Rank by **value at risk** = $\lvert\sum(F-A)\rvert\times$ unit margin (or cost) × ABC weight (A = 3, B = 2, C = 1 [RoT]). Show the top N. |
| **GS-G-10 / 11** | Standard output; reproducible SQL/Python. | |

### Metric-misuse guard (runs automatically)
| Condition | Action |
|---|---|
| MAPE requested on a series with any $\mathit{A}_t=0$ or ADI ≥ 1.32 | Replace with WAPE + MASE/RMSSE; note `MAPE_SUPPRESSED` |
| Comparison of forecasts frozen at different lags | Block the comparison; `LAG_MISMATCH` |
| Aggregated-then-scored WAPE presented as item accuracy | Relabel the level; compute item-level WAPE |
| Accuracy averaged across items as an unweighted mean of WAPE | Show the volume-weighted WAPE instead; unweighted only as a secondary |
| Actuals = shipments while orders exist | Warn `CONSTRAINED_ACTUALS` (GS-D-11) |
| Different item sets across compared versions | Restrict to the intersection; `ITEMSET_ALIGNED` |

## 3. Required Inputs & Data Schema
**`forecast_snapshots`**: one row per version × item × location × target period × creation period.

| Column | Type | Unit | Req. | Notes |
|---|---|---|---|---|
| `item_id`, `location_id` (`customer_id`, `channel`) | string | n/a | Y | |
| `target_period` | date | fiscal bucket | Y | |
| `created_period` | date | fiscal bucket | Y | Lag $\ell$ = periods between created and target (fiscal calendar) |
| `forecast_version` | enum | `NF1`/`NF2`/`STAT`/`INT`/`ML`/`REC`/`OVERRIDE`/`CONSENSUS`/`FINAL` | Y | |
| `fcst_qty` | decimal ≥ 0 | base UoM | Y | Mean forecast |
| `q10`, `q50`, `q90` (+ `pi_lo`, `pi_hi`, `pi_level`) | decimal | base UoM | N | |
| `model`, `owner` | string | n/a | N | |

**`actuals`**: `item_id, location_id, period, actual_unconstrained, actual_sales, pct_reconstructed_flag, period_state` (from `nextq-demand-history-cleansing`). Structural periods are excluded.

**`history_for_scale`**: the training history per series for the MASE/RMSSE denominator (same as `demand_clean`).

**`attributes`**: `ABC, XYZ, demand_class, unit_margin or unit_cost, price` (from `nextq-demand-pattern-segmentation` / master data).

**Granularity:** one grain per run (default SKU-Location-Week). Lags are in the same buckets.

**Minimum:** ≥ 3 target periods per series per lag for series-level metrics. ≥ 1 seasonal cycle of history before the first origin for the MASE scale; otherwise use $\mathit{m}=1$ and flag.

## 4. Execution Workflow
1. **Validate snapshots.** Compute `lag` from the fiscal calendar. Drop rows with lag < 1 and flag forecasts with no creation date. Deduplicate: keep the **last** snapshot per (version, key, target, lag) before the freeze cut-off, and log the drop count.
2. **Join actuals.** Use the unconstrained basis. Exclude STRUCTURAL periods. Attach `pct_reconstructed`.
3. **Align scope.** For any multi-version comparison, intersect items × periods × lags (`ITEMSET_ALIGNED`).
4. **Compute the MASE/RMSSE scale per series and origin** from history ≤ created_period. Use $\mathit{m}$ = the seasonal period if the series is classified SEASONAL with ≥ 2 cycles, else $\mathit{m}=1$. Intermittent series use $\mathit{m}=1$.
5. **Compute metrics** per (version, series, lag): Σ|e|, ΣA, Σ(F−A), MASE, RMSSE, MAPE (eligible only), sMAPE, pinball per quantile, and coverage. Roll up per level and segment with the GS-K-13 weighting rules.
6. **Tracking signal** per series on the decision lag. Use a rolling window $\mathit{w}$ (default 13 for weekly, 6 for monthly) and also the smoothed variant. Flag $\lvert \mathit{TS}\rvert>4$ and record its sign.
7. **Run the metric-misuse guard** (table above). Apply substitutions and record them.
8. **Diagnostics:**
   - Error decomposition $\text{MSE}=\text{Bias}^2+\text{Var}(e)$ per series. A bias share > 50% means systematic error.
   - Concentration (Pareto): % of total absolute error from the top 20% of series.
   - Accuracy by lag curve.
   - Error by segment (ABC × XYZ, class, region, customer).
9. **Model / version comparison** (optional): a paired Diebold-Mariano-style test on the per-period loss differential $\mathit{d}_t=\lvert e^{(1)}_t\rvert-\lvert e^{(2)}_t\rvert$ pooled per series, or a bootstrap over items (seeded, 1,000 resamples) for the WAPE difference. Report the 95% CI.
10. **Exceptions:** apply the GS-G-03 triggers, compute value at risk, rank, and keep the top N (default 50).
11. **Output** (Section 6), with the reproducibility block.

## 5. Verification Snippets
```python
import numpy as np, pandas as pd

def kpis(A, F, y_train, m=1, q=None, A_for_q=None):
    """Canonical GS-K metrics for one series at one lag. A, F aligned arrays; y_train for scale."""
    A, F = np.asarray(A, float), np.asarray(F, float); e = A - F
    yt = np.asarray(y_train, float)
    d1 = np.abs(yt[m:] - yt[:-m]) if len(yt) > m else np.array([np.nan])
    s1, s2 = np.nanmean(d1), np.nanmean(d1 ** 2)
    pos = A > 0
    out = dict(
        n=len(A), sumA=A.sum(), sumAE=np.abs(e).sum(),
        WAPE=100 * np.abs(e).sum() / np.abs(A).sum() if np.abs(A).sum() > 0 else np.nan,
        MAPE=100 * np.mean(np.abs(e[pos] / A[pos])) if pos.any() else np.nan, MAPE_excluded_zeros=int((~pos).sum()),
        sMAPE=100 * np.mean(np.where((np.abs(A) + np.abs(F)) == 0, 0, 2 * np.abs(e) / np.where((np.abs(A) + np.abs(F)) == 0, 1, np.abs(A) + np.abs(F)))),
        MASE=np.mean(np.abs(e)) / s1 if s1 > 1e-9 else np.nan,
        RMSSE=np.sqrt(np.mean(e ** 2) / s2) if s2 > 1e-12 else np.nan,
        Bias_pct=100 * (F - A).sum() / A.sum() if A.sum() > 0 else np.nan,   # + = over-forecast
        ME=e.mean())
    return out

def pinball(y, f, p):
    y, f = np.asarray(y, float), np.asarray(f, float)
    return np.mean(np.where(y < f, 2 * (1 - p) * (f - y), 2 * p * (y - f)))

def coverage(y, lo, hi): return float(np.mean((np.asarray(y) >= lo) & (np.asarray(y) <= hi)))

def tracking_signal(A, F, w=13, alpha=None):
    """APICS TS: rolling sum of (A-F) / MAD over same window; positive = under-forecast."""
    e = pd.Series(np.asarray(A, float) - np.asarray(F, float))
    if alpha is None:
        rsfe = e.rolling(w, min_periods=w).sum(); mad = e.abs().rolling(w, min_periods=w).mean()
    else:
        rsfe = e.cumsum(); mad = e.abs().ewm(alpha=alpha, adjust=False).mean()
    return rsfe / mad.replace(0, np.nan)

def bootstrap_wape_diff(df, v1, v2, n=1000, seed=42):
    """Item-level bootstrap CI of WAPE(v1) - WAPE(v2). df: item, A, F_v1, F_v2 summed per item."""
    rng = np.random.default_rng(seed); items = df.index.to_numpy(); diffs = []
    for _ in range(n):
        s = df.loc[rng.choice(items, len(items), replace=True)]
        diffs.append(100 * ((s.A - s[v1]).abs().sum() - (s.A - s[v2]).abs().sum()) / s.A.sum())
    return np.percentile(diffs, [2.5, 50, 97.5])
```
**Test vectors** (verified):
- `A=[10,0,10,20]`, `F=[12,1,8,20]`, `y_train=[8,10,12,10,11]`: WAPE = 12.5%, Bias% = +2.5% (over), MAPE = 13.33% with 1 zero excluded, MASE ≈ 0.714 (scale 1.75), and RMSSE ≈ 0.832.
- A constant under-forecast of 2 units for 13 periods (|e| = 2) gives TS = +13.
- `pinball([10],[8],0.9)` = 3.6.

```sql
-- Portfolio scorecard at decision lag (GS-K-01/05/13); weighting = volume
WITH s AS (
  SELECT f.forecast_version, f.item_id, f.location_id, f.target_period,
         a.actual_unconstrained AS A, f.fcst_qty AS F
  FROM forecast_snapshots f
  JOIN actuals a ON a.item_id=f.item_id AND a.location_id=f.location_id AND a.period=f.target_period
  WHERE f.lag = :lag AND a.period_state <> 'STRUCTURAL_ZERO'
    AND f.target_period BETWEEN :w_start AND :w_end)
SELECT forecast_version,
       COUNT(DISTINCT item_id||'|'||location_id)                 AS n_series,
       100.0*SUM(ABS(A-F))/NULLIF(SUM(ABS(A)),0)                 AS wape_pct,
       100.0*SUM(F-A)/NULLIF(SUM(A),0)                           AS bias_pct_pos_over,
       100.0*SUM(CASE WHEN A=0 THEN 1 ELSE 0 END)/COUNT(*)       AS pct_zero_actual_periods
FROM s GROUP BY forecast_version;
```

## 6. Standardized Output Format
```markdown
# Forecast Accuracy Diagnostics — <scope> — run ACC_<run_id>

## Executive Summary
- At decision lag ℓ=<lag> (<level>, <window>): WAPE <x>% (volume-weighted), median MASE <y>, Bias <z>% (<over/under>-forecast).
- Error concentration: top 20% of series = <p>% of absolute error; bias share of MSE <b>%.
- <k> series out of control (|TS|>4: <u> under, <o> over); value at risk <currency>.
- Metric guard: <MAPE_SUPPRESSED on n series | LAG_MISMATCH blocked | …>.

## Metadata (mandatory)
| Level | Grain | Lag(s) | Time window | Weighting | Actuals basis | Versions | N series | N periods | % reconstructed | % excluded zeros (MAPE) |
|---|---|---|---|---|---|---|---|---|---|---|
| SKU-Location | WEEK | 1,2,4 (decision=2) | <start–end> | WAPE volume; MASE median; Bias Σ | unconstrained | STAT, FINAL | | | | |

## Scorecard by Version (lag ℓ)
| Version | WAPE % | Median MASE | RMSSE (intermittent) | Bias % (+=over) | Abs-bias roll-up % | MAPE % (eligible) | sMAPE % (ref) | Pinball P90 | 80% coverage |
|---|---|---|---|---|---|---|---|---|---|

## Accuracy by Lag
| Lag | WAPE % | Median MASE | Bias % | Coverage 80% | N |
|---|---|---|---|---|---|

## Accuracy by Segment (decision lag)
| Segment (ABC×XYZ / class / region) | N | Volume share % | WAPE % | Median MASE | Bias % | % series \|TS\|>4 |
|---|---|---|---|---|---|---|

## Version Comparison (optional)
| Comparison | ΔWAPE pts | 95% CI (bootstrap items) | Significant? |
|---|---|---|---|

## Exceptions (top N by value at risk)
| ID | Trigger (rule, threshold) | item | loc | lag | metric | value | units at risk | value at risk | likely cause | action | owner |
|---|---|---|---|---|---|---|---|---|---|---|---|
| | \|TS\|>4 (GS-K-06) | | | | TS | | | | level shift / bias | re-baseline, review overrides | |
| | \|Bias\|>10% ×3 cycles (GS-K-05) | | | | | | | | | | |
| | MASE>1 (GS-G-03) | | | | | | | | | route review | |
| | Coverage off >5pp (GS-K-14) | | | | | | | | | recalibrate PIs | |
| | LOW_CONFIDENCE actuals (>30% reconstructed) | | | | | | | | | reconstruct / caveat | |

## Recommendations
1. …

## Data-Quality Log
| issue | count | treatment |
|---|---|---|
| snapshots without created date (not scored) | | |
| duplicate snapshots (kept last before freeze) | | |
| structural periods excluded | | |
| MAPE zero exclusions | | |
| SCALE_ZERO series (MASE→WAPE) | | |

## Reproducibility
Snapshot extract id, freeze cut-off rule, lag calendar, TS window/α, bootstrap seed, code §5.
```

## 7. Edge Cases
- **ΣA = 0 for a series in the window:** WAPE, Bias%, and MAPE are undefined. Report the absolute error (units) and RMSSE, and flag `NO_ACTUAL_DEMAND`. If F > 0 persistently, this is an obsolescence/over-forecast signal.
- **Mixed lags in one scorecard:** each table is single-lag, or has lag as an explicit column. Never pool.
- **Promo periods:** report accuracy for promo and non-promo periods separately. Promo error usually dominates and should be owned by `nextq-promotion-lift-cannibalization-planning`.
- **Item supersession:** score the successor on chained actuals (cleansing GS-D-12). The predecessor's snapshots after the effective date are excluded.
- **Aggregation level effect:** WAPE at brand-month is always ≤ the weighted SKU-week WAPE (triangle inequality). Label the level explicitly.
- **Forecasts revised after freeze:** use only the snapshot at the freeze. Later revisions are scored under their own (shorter) lag.
- **Negative forecasts or actuals** (returns): actuals use gross demand (returns separate). Negative forecasts are an upstream error; floor them at 0 for scoring and flag them.
