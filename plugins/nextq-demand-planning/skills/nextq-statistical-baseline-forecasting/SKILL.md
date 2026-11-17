---
name: nextq-statistical-baseline-forecasting
description: >-
  [Next Quarter Supply Chain Suite] Produces the statistical baseline demand forecast (point + prediction intervals)
  for smooth and erratic series using naive / seasonal-naive benchmarks, ETS and ARIMA selected by AICc, STL+ETS or
  dynamic harmonic regression for weekly data, multi-seasonal models for daily data, and simple combinations, all
  chosen by rolling-origin backtest at the decision lag. Load when the user asks to build, refresh, or backtest a
  baseline / statistical / time-series forecast, compare ETS vs ARIMA, beat the naive forecast, generate a 13-week,
  18-month or S&OP horizon forecast, or produce forecasts with prediction intervals. Inputs: cleaned demand
  (nextq-demand-history-cleansing) and segmentation routes (nextq-demand-pattern-segmentation) at
  SKU-Location-Week/Month/Day. Triggers: "baseline forecast", "statistical forecast", "Holt-Winters", "auto ARIMA",
  "ETS", "forecast next 52 weeks", "backtest", "naive benchmark".
metadata:
  author: "Next Quarter"
  organization: "Next Quarter (nextq)"
  suite: "Next Quarter Supply Chain Intelligence"
  version: "1.0.0"
---

# Statistical Baseline Forecasting
*Part of the **Next Quarter (nextq)** Supply Chain Skill Suite — Governed by SUPPLY_CHAIN_GOLD_STANDARD.md.*

## 1. Role & Objective
**Persona:** Principal Statistical Forecaster.
**Decision supported:** Which statistical model produces the most accurate, least biased **unconstrained** baseline at the decision lag for each smooth or erratic series? What is its forecast distribution over the planning horizon?

**Objective:** A backtested, benchmark-beating baseline forecast (mean + quantiles) per series. It comes with model choice, accuracy by lag, bias, and interval coverage. It is the "Statistical" row of the FVA stair-step (GS-K-07).

**Scope:** routes `R_STAT`, `R_STAT_WEEKLY`, `R_STAT_DAILY`, `R_STAT_PLUS_SBA`, and the statistical arm of `R_CAUSAL`. Intermittent and lumpy series belong to `nextq-intermittent-demand-forecasting`.

## 2. Gold Standard Rules Enforced
| Rule | Inlined standard |
|---|---|
| **GS-M-01** Error / lag | $\mathit{e}_t = A_t - F_t$. Every score states **lag ℓ** (forecast frozen at $\mathit{t}-\ell$), **level**, **window**, and **weighting**. Default decision lag = cumulative lead time or frozen zone (for example, lag 2 weeks or lag 1 month) [RoT]. |
| **GS-M-02** Benchmark first | **NF1 naïve:** $\hat y_{T+h\mid T} = y_T$. **NF2 seasonal naïve:** $\hat y_{T+h\mid T} = y_{T+h-m(k+1)}$, with $\mathit{k}=\lfloor (h-1)/m\rfloor$. Optional: 52-week or 12-month moving average. A candidate that does not beat the appropriate naïve model (NF2 if seasonal, else NF1) out of sample at the decision lag is **rejected**. If nothing beats naïve, the naïve model is the production baseline and the series is flagged `NAIVE_WINS`. |
| **GS-M-04** Selection order | Clean → classify → candidate set → rolling-origin backtest → pick on the primary metric → bias check → (reconcile in A6). **Within a family:** min **AICc** (ETS over admissible error/trend/season combos per Hyndman et al. 2002; ARIMA by Hyndman-Khandakar stepwise AICc). **Across families:** rolling-origin CV only. Weekly ($\mathit{m}≈52.18$): STL + non-seasonal ETS, or DHR with Fourier $\mathit{K}$ by AICc. Daily: multi-seasonal (weekly + annual) via STL/MSTL or DHR, plus holiday/moving-event dummies. Short series: min AICc acceptable; no "30-observation" rule. |
| **GS-M-05** ES/ARIMA essentials | SES $\ell_t=\alpha y_t+(1-\alpha)\ell_{t-1}$. ETS(A,A,A): $\hat y_{t+h|t}=\ell_t+hb_t+s_{t+h-m(k+1)}$. Damped trend is allowed and preferred for horizons > 1 year [RoT]. If a Box-Cox/log transform is used, **bias-adjust** back-transformed forecasts to the mean (FPP3 eq. 5.3; log case $\hat y = e^{\hat w}(1+\sigma_h^2/2)$). This is mandatory because forecasts are summed in hierarchies and finance. Report Ljung-Box ($\ell=10$ non-seasonal, $2m$ seasonal, capped at $\mathit{T}/5$) but **never select on it**. |
| **GS-M-10** Probabilistic | Emit quantiles {0.1, 0.5, 0.9} plus any service quantiles requested. Normal PI: $\hat y \pm c\,\hat\sigma_h$ (80% c = 1.28, 95% c = 1.96). Use bootstrapped residual paths when residuals are non-normal. Lead-time totals come from summed **sample paths**, never summed quantiles. Score with pinball loss $\mathit{Q}_{p,t}$ ($2(1-p)(f-y)$ if $\mathit{y}<f$, else $2p(y-f)$) and coverage. |
| **GS-M-15** Combination | If no single model wins by > 2% relative MASE across origins [RoT], use the **equal-weight** mean of the top 2–3 (Bates & Granger 1969; FPP3 §13.4). |
| **GS-M-16** Rolling origin | Expanding window. Horizon ≥ the max decision horizon. ≥ 3 origins, or 1 full seasonal cycle of origins for seasonal items [RoT]. Score **per lag**, and never average across horizons without reporting each lag. Holdout ≈ 20% for a single split. |
| **GS-K-01** WAPE | $\text{WAPE}=\sum\lvert A-F\rvert/\sum\lvert A\rvert \times 100$. Primary **portfolio** metric. Undefined if $\sum A=0$. |
| **GS-K-04** MASE / RMSSE | $\text{MASE}=\text{mean}\lvert e_j\rvert \big/ \frac{1}{T-m}\sum_{t=m+1}^{T}\lvert y_t-y_{t-m}\rvert$, with the scale from the **training window of each origin**. Primary **per-series** metric. MASE < 1 beats in-sample naïve. |
| **GS-K-05** Bias | $\text{Bias\%}=\sum(F-A)/\sum A\times100$ (**positive = over-forecast**); state the direction in words. |
| **GS-K-14** Coverage | Coverage of X% PI should be ≈ X%. Flag deviation > 5 percentage points [RoT]. |
| **GS-G-03** Exceptions | MASE > 1.0; \|Bias%\| > 10% over the last 3 origins; coverage off > 5 pp; final model FVA vs. naïve < 0; forecast change vs. last cycle > ±25% inside the frozen horizon [RoT]. |
| **GS-G-10/11** | Standard output contract; reproducibility (seeded bootstrap; code and params logged). |

**Model-selection score (deterministic):**
1. Primary: mean MASE across origins at the decision lag.
2. Tie-break (< 1% relative): lower |Bias%|.
3. Then: fewer parameters.

**Hard gates:** the model must beat naïve on MASE at the decision lag, and |Bias%| must be ≤ 10% [RoT]. Otherwise take the next candidate.

## 3. Required Inputs & Data Schema
**`demand_clean`** (from `nextq-demand-history-cleansing`). Use `demand_clean` where `train_mask = true`. Structural periods are excluded; set them as gaps and use models that accept missing values, or truncate.

| Column | Type | Unit | Req. |
|---|---|---|---|
| `item_id`, `location_id` | string | n/a | Y |
| `period_start` | date | fiscal bucket | Y |
| `demand_clean` | decimal ≥ 0 | base UoM | Y |
| `train_mask`, `period_state`, `outlier_cause` | n/a | n/a | Y |
| `event_flags`, `promo_flag` | n/a | n/a | N (holiday dummies for daily) |

**`segmentation`** (from `nextq-demand-pattern-segmentation`): `item_id, location_id, route, demand_class, SEASONAL, naive_scale, ABC, XYZ, flags`.

**Run parameters:**
- `grain` ∈ {DAY, WEEK, MONTH}; `m` = 7 (daily, weekly cycle; annual 365.25 added if ≥ 2 years) / 52 (weekly; 52.18 for Fourier) / 12.
- `horizon_H` (for example, 26 or 52 weeks, 18 months); `decision_lag`; `n_origins`; `quantiles`; `seed`.

**Minimum history:**
- ≥ 2 full seasonal cycles for seasonal ETS/ARIMA or STL. With 1–2 cycles, use non-seasonal ETS on deseasonalized data or NF2, and flag `SHORT_HISTORY`.
- < 1 cycle: route back to `R_NEW_ANALOG`.

## 4. Execution Workflow
1. **Pre-checks.**
   - Reject series whose route is not statistical.
   - Carry over the `LOW_CONFIDENCE` / `needs_model_recon` flags. If > 30% of history is reconstructed, still forecast but mark the output `LOW_CONFIDENCE`.
   - Confirm there are no `MISSING` periods without imputation.
2. **Candidate set by route:**
   - `R_STAT` (monthly): NF1, NF2, MA12, ETS(auto, AICc, damped allowed), ARIMA(auto, AICc), and the combination of the best 2–3.
   - `R_STAT_WEEKLY`: NF1, NF2 (lag 52), MA52, STL(m=52, robust) + ETS(non-seasonal), DHR = ARIMA errors + Fourier(K ∈ 1..10 by AICc, period 52.18), ETS on non-seasonal data if not SEASONAL, and the combination.
   - `R_STAT_DAILY`: NF2 (lag 7), MSTL(7, 365.25) + ETS, DHR with weekly and annual Fourier + holiday dummies, and the combination.
   - `R_STAT_PLUS_SBA`: the above plus SBA (via `nextq-intermittent-demand-forecasting` §5 function).
3. **Transformations.**
   - If variance grows with level (correlation of rolling mean and rolling SD > 0.5 [RoT]), also test log(y+1) or Box-Cox variants.
   - Back-transform with bias adjustment (GS-M-05).
4. **Rolling-origin backtest (GS-M-16).**
   - Set origins $\mathit{T}_0, T_0+s, \dots$ ($\mathit{s}$ = step, default 1 period for monthly or 4 for weekly), with the last origin at $\mathit{T}-H_{\mathrm{bt}}$, where $\mathit{H}_{\mathrm{bt}}=\max(\text{decision\_lag}, \text{backtest horizon})$.
   - At each origin, refit on data ≤ origin only (**no look-ahead**: outlier flags and imputation must use only data available at that origin when causal; otherwise document) and forecast $\mathit{h}=1..H_{\mathrm{bt}}$.
   - Store `(origin, h, F, quantiles)`.
5. **Score per series, per lag.**
   - Compute MASE (scale from the origin's training window), Bias%, pinball, and coverage.
   - Compute WAPE at portfolio level.
   - Compute **FVA vs. naïve** = MASE_naïve − MASE_model (positive = adds value).
6. **Select** with the deterministic score and gates (Section 2).
   - Apply the combination rule (GS-M-15).
   - If no candidate passes the gates, use the production model = naïve (NF2 if seasonal else NF1) and flag `NAIVE_WINS`.
7. **Final fit and forecast.**
   - Refit the winner on all history and forecast $\mathit{h} = 1..H$.
   - Produce the mean (bias-adjusted), quantiles, and 1,000 sample paths (seeded) for lead-time aggregation.
   - Floor negatives at 0 and log the count.
8. **Diagnostics:** Ljung-Box p-value, residual mean (should be ≈ 0; otherwise bias), and change vs. last cycle's forecast within the frozen horizon.
9. **Exceptions and output** (Section 6). Persist `forecast_version = STAT_<run_id>` and lag snapshots for FVA (B1).

## 5. Verification Snippets
```python
# Rolling-origin backtest with Nixtla statsforecast (if available; otherwise statsmodels ETS/ARIMA)
import numpy as np, pandas as pd
from statsforecast import StatsForecast
from statsforecast.models import Naive, SeasonalNaive, AutoETS, AutoARIMA, WindowAverage, MSTL

m, H, lag = 52, 13, 2                      # weekly, 13-week evaluation horizon, decision lag 2
df = demand[["unique_id", "ds", "y"]]       # unique_id = item|loc, ds = week start, y = demand_clean
models = [Naive(), SeasonalNaive(season_length=m), WindowAverage(window_size=m),
          # weekly: STL + non-seasonal ETS (GS-M-04); ETS with m=52 is not used
          MSTL(season_length=[m], trend_forecaster=AutoETS(model="ZZN"), alias="STL_ETS"),
          AutoARIMA(season_length=m)]       # AICc stepwise; for DHR add Fourier exog (K by AICc) via X_df
# monthly runs: m=12 and add AutoETS(season_length=12) (AICc over admissible ETS models)
sf = StatsForecast(models=models, freq="W-SAT", n_jobs=-1)
cv = sf.cross_validation(df=df, h=H, step_size=4, n_windows=13)   # 13 origins >= 1 quarter; expanding window
cv["h"] = cv.groupby(["unique_id", "cutoff"]).cumcount() + 1

def mase_scale(train_y, m):
    y = np.asarray(train_y, float)
    return np.mean(np.abs(y[m:] - y[:-m])) if len(y) > m else np.mean(np.abs(np.diff(y)))

def score(cv, df, model_cols, lag, m):
    rows = []
    for (uid, cutoff), g in cv[cv.h == lag].groupby(["unique_id", "cutoff"]):
        s = mase_scale(df[(df.unique_id == uid) & (df.ds <= cutoff)].y, m)
        for mc in model_cols:
            e = g.y - g[mc]
            rows.append(dict(unique_id=uid, cutoff=cutoff, model=mc, ae=np.abs(e).sum(), a=g.y.sum(),
                             f_minus_a=(g[mc] - g.y).sum(),
                             q=(np.abs(e) / s).mean() if s > 1e-9 * max(abs(g.y.mean()), 1) else np.nan))  # SCALE_ZERO -> NaN, use WAPE
    r = pd.DataFrame(rows)
    per = r.groupby(["unique_id", "model"]).agg(MASE=("q", "mean"), AE=("ae", "sum"), A=("a", "sum"), FA=("f_minus_a", "sum"))
    per["Bias%"] = 100 * per.FA / per.A
    wape = r.groupby("model").apply(lambda x: 100 * x.ae.sum() / x.a.sum())
    return per, wape
```
**Weekly ETS caveat:** many ETS implementations cap the seasonal period (for example, at 24). For weekly data, use STL + ETS(non-seasonal) or DHR as GS-M-04 prescribes, not ETS with m = 52.

```python
# Bias-adjusted back-transform for log models (GS-M-05)
def backtransform_log_mean(w_hat, sigma2_h):
    return np.exp(w_hat) * (1 + sigma2_h / 2)

# Interval coverage (GS-K-14)
def coverage(y, lo, hi): return np.mean((y >= lo) & (y <= hi))
```
```sql
-- Portfolio WAPE and Bias% at decision lag (GS-K-01/05)
SELECT model,
       100.0 * SUM(ABS(actual - fcst)) / NULLIF(SUM(ABS(actual)),0) AS wape_pct,
       100.0 * SUM(fcst - actual)      / NULLIF(SUM(actual),0)      AS bias_pct_over_positive
FROM backtest WHERE lag = :decision_lag AND period BETWEEN :w_start AND :w_end
GROUP BY model;
```
**Sanity tests:**
- On a pure random walk, NF1 should win (MASE ≈ 1).
- On deterministic seasonal data with noise, NF2 MASE ≈ 1 out of sample, and a correct seasonal model should be < 1.

## 6. Standardized Output Format
```markdown
# Statistical Baseline Forecast — <scope> — run STAT_<run_id>

## Executive Summary
- <N> series forecast for <H> <periods>; decision lag ℓ=<lag>; <x>% beat naïve (FVA>0), <y>% NAIVE_WINS.
- Portfolio WAPE <a>% vs naïve <b>% (FVA <b−a> pts); median MASE <c>; Bias <d>% (<over/under>-forecast).
- 80% PI coverage <e>% (target 80±5); <k> series flagged; value at risk <currency>.
- Model mix: ETS a%, ARIMA b%, STL+ETS c%, DHR d%, Combination e%, Naïve f%.

## Metadata (mandatory on every KPI table)
| Level | Grain | Lag | Time window (evaluation) | Origins | Horizon | Weighting | Actuals basis |
|---|---|---|---|---|---|---|---|
| SKU-Location | WEEK (W-SAT) | 2 | <start–end> | 13 | 13 | WAPE volume-weighted; MASE unweighted median | unconstrained (cleansed), % reconstructed <r>% |

## Accuracy by Model (portfolio, lag ℓ)
| Model | WAPE % | Median MASE | Bias % (+=over) | Pinball P10/P50/P90 | 80% PI coverage | FVA vs naïve (WAPE pts) | # series selected |
|---|---|---|---|---|---|---|---|
| NF1 / NF2 | | | | | | 0 | |
| ETS | | | | | | | |
| ARIMA | | | | | | | |
| STL+ETS / DHR | | | | | | | |
| Combination | | | | | | | |
| **Selected (final)** | | | | | | | |

## Accuracy by Lag (selected models)
| Lag | WAPE % | Median MASE | Bias % | Coverage 80% |
|---|---|---|---|---|

## Series Detail (top N by value + exceptions)
| item | loc | route | selected model | spec (e.g., ETS(M,Ad,N)) | AICc | MASE ℓ | Bias % | FVA vs naïve | LB p | flags |
|---|---|---|---|---|---|---|---|---|---|---|

## Forecast Output (table/file)
| item | loc | period | h | mean | P10 | P50 | P90 | model | version |
|---|---|---|---|---|---|---|---|---|---|

## Exceptions
| ID | Trigger (rule, threshold) | item | loc | metric value | units at risk | value at risk | action | owner |
|---|---|---|---|---|---|---|---|---|
| | MASE>1.0 (GS-G-03) | | | | | | review data/route; consider ML or intermittent | |
| | \|Bias%\|>10% 3 origins (GS-K-05) | | | | | | investigate level shift / censoring | |
| | NAIVE_WINS (GS-M-02) | | | | | | use naïve in production; model review | |
| | Coverage off >5pp (GS-K-14) | | | | | | bootstrap intervals / transform | |
| | Δ vs last cycle >±25% in frozen zone (GS-G-03) | | | | | | planner confirmation | |
| | LOW_CONFIDENCE (>30% reconstructed) | | | | | | reconstruct demand (C2) | |

## Recommendations
1. …

## Data-Quality Log
| item | loc | % reconstructed | imputed periods | negatives floored | transform | notes |

## Reproducibility
Inputs: cleansing <id>, segmentation <id>; params m, H, lag, step, n_windows, quantiles, seed=<s>; library versions; code §5.
```

## 7. Edge Cases
- **Constant series** (naïve scale = 0): MASE undefined. Use WAPE and the absolute error, and flag `SCALE_ZERO`.
- **Level shift detected** (for example, CUSUM or a structural-break test, or a persistent residual mean ≠ 0): fit on post-break data if ≥ 1 cycle remains, and flag `REGIME_CHANGE`.
- **Week 53:** for NF2, use the restated like-week mapping (week 53 maps to week 52 of the prior year). For DHR, period 52.18 handles it naturally (GS-D-10).
- **Moving holidays** (Easter, Lunar New Year, Diwali): ETS cannot take regressors. Use DHR/ARIMA with dummies or STL with holiday-adjusted data.
- **Negative forecasts:** floor at 0, or model log(y+1) with bias adjustment. Log the count of floored values.
- **Very long series (> ~200 periods for monthly data):** consider fitting on recent windows (FPP3 §13.7) and compare by backtest.
- **Promo-driven series:** the baseline must exclude promo lift. Use promo-masked training or hand off to A5/B3. Never let promo spikes inflate the baseline.
