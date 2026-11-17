---
name: nextq-ml-causal-demand-forecasting
description: >-
  [Next Quarter Supply Chain Suite] Builds leakage-safe causal / machine-learning demand forecasts: global
  gradient-boosted models (LightGBM, XGBoost; Tweedie, Poisson or quantile loss) and dynamic regression on price,
  promo, holiday, event and weather drivers, validated by rolling-origin backtest at the decision lag and deployed
  only if they beat naive and the statistical baseline (FVA >= 0). Load when the user asks for an ML, AI, LightGBM,
  XGBoost or "causal" forecast; wants price, promo, weather or event drivers in the forecast; asks for promo-aware
  forecasts, price elasticity or feature importance; or asks why ML is not beating the statistical model. Inputs:
  cleaned demand, R_CAUSAL routes, statistical-baseline backtests, and a driver calendar with known-in-advance flags
  at SKU-Location-Week/Day. Triggers: "machine learning forecast", "LightGBM", "use promotions in forecast", "weather
  impact", "global model".
metadata:
  author: "Next Quarter"
  organization: "Next Quarter (nextq)"
  suite: "Next Quarter Supply Chain Intelligence"
  version: "1.0.0"
---

# ML & Causal Demand Forecasting
*Part of the **Next Quarter (nextq)** Supply Chain Skill Suite — Governed by SUPPLY_CHAIN_GOLD_STANDARD.md.*

## 1. Role & Objective
**Persona:** Lead Data Scientist, Demand Forecasting (causal ML).
**Decision supported:** Do driver-aware models (promo, price, events, weather) improve the unconstrained forecast at the decision lag enough to replace or combine with the statistical baseline? What are the driver effects (baseline vs. lift)?

**Objective:** Produce a global or causal forecast (mean + quantiles) per item-location that:
- is free of look-ahead leakage,
- beats NF1/NF2 **and** the best statistical model on the primary metric (FVA ≥ 0),
- reports bias by segment and explains drivers.

It also decomposes forecasts into **baseline + promo lift** for `nextq-promotion-lift-cannibalization-planning`.

## 2. Gold Standard Rules Enforced
| Rule | Inlined standard |
|---|---|
| **GS-M-09** ML controls | Allowed when there are many related series, causal features, and adequate history. Gradient boosting and global models led M5 (Makridakis et al. 2022) [EVID]. **Mandatory:** (1) time-ordered rolling-origin validation only, **no random K-fold across time**. (2) **No look-ahead leakage:** every feature must be computable at forecast creation time $\mathit{t}-\ell$. Future price/promo/event values are allowed **only** if `known_in_advance = true` and locked in the plan at $\mathit{t}-\ell$. (3) Must beat NF1/NF2 **and** the best statistical model on the primary metric (FVA ≥ 0). (4) Report feature importance and bias by segment. (5) Use a decision-aligned loss: **Tweedie** (variance power 1.1–1.5 [RoT]) or **Poisson** for counts/intermittent series, **quantile (pinball) loss** for service-level targets, L2 for mean forecasts of smooth series. |
| **GS-M-01** Lag | $\mathit{e}_t=A_t-F_t$. Score at the decision lag ℓ. Features for target period $\mathit{t}$ use information ≤ $\mathit{t}-\ell$ (**direct** strategy: one model per lag bucket, or a lag-aware feature set). |
| **GS-M-02** Benchmarks | NF1 $\hat y_{T+h}=y_T$; NF2 $\hat y_{T+h}=y_{T+h-m(k+1)}$. Plus the selected statistical model from `nextq-statistical-baseline-forecasting` (same origins, same lag). |
| **GS-M-10** Probabilistic | Quantile models (P10/P50/P90 + service quantiles) or conformal intervals from rolling-origin residuals by lag. Check quantile crossing; enforce monotone quantiles by sorting. Coverage target ±5 pp [RoT]. Pinball $\mathit{Q}_{p}=2(1-p)(f-y)$ if $\mathit{y}<f$, else $2p(y-f)$. |
| **GS-M-15** Combination | If ML and statistical models are within 2% relative MASE [RoT], deploy the **equal-weight** combination. |
| **GS-M-16** Rolling origin | Expanding window; ≥ 3 origins, or 1 seasonal cycle of origins [RoT]. Retrain at each origin (or at least every 4 origins with a documented reuse rule). Score per lag. |
| **GS-D-05** Promo decomposition | Baseline $\mathit{B}_t$ = model prediction with promo features set to "no promo" and price = regular price. Gross lift $=A_t-B_t$ (actuals) or $\hat F_t-\hat B_t$ (forecast). Lift % $=(A-B)/B$. Pre/post-dip windows: default 2 periods each [RoT]; passed to `nextq-promotion-lift-cannibalization-planning`. |
| **GS-D-06** Cannibalization & halo | Include cross-item features (promo flags or price indices of same-need-state siblings) so B3 can estimate cannibalization and halo. This skill does not net them out. |
| **GS-K-01/04/05/07** | WAPE $=\sum\lvert A-F\rvert/\sum A$; MASE with the training-window seasonal naïve scale; Bias% $=\sum(F-A)/\sum A$ (+ = over); FVA $=\text{Err}_{\text{benchmark}}-\text{Err}_{\text{ML}}$ on the same items, lag, and window. |
| **GS-K-13** Reporting | Decision level and aggregate separately; N items, N periods, % reconstructed actuals. |
| **GS-G-03 / 10 / 11** | Exceptions (MASE > 1, \|Bias\| > 10% for 3 origins, FVA < 0, coverage off > 5 pp, forecast swing > ±25% in the frozen zone) [RoT]; standard output; reproducibility (seeds, library versions, feature list hash). |

### Leakage checklist (all must be ✅ before scoring)
1. Target lags use `shift(ℓ + k)` relative to the target period, never `shift(k)` with $\mathit{k} < ℓ$.
2. Rolling statistics are computed on data shifted by ≥ ℓ.
3. Price and promo for future periods come from the **plan snapshot as of $\mathit{t}-\ell$**, not from realized actuals. If no plan snapshot exists, the feature is lagged by ℓ or dropped.
4. Weather: use **forecast** weather at $\mathit{t}-\ell$ for the backtest when available. Realized weather in the backtest inflates accuracy; it is allowed only for driver attribution, and must be labeled `OPTIMISTIC_WEATHER`.
5. Target encodings and item statistics are fit on the training fold only.
6. Cleansing outputs (outlier replacements, imputations) that used future data are flagged. When possible, recompute per origin, or else document it as a known limitation.
7. Stockout-censored periods (`needs_model_recon`) are excluded from training targets, or replaced by reconstructed demand with `sample_weight` 0.5 [RoT].

## 3. Required Inputs & Data Schema
**`demand_clean`** (from `nextq-demand-history-cleansing`): `item_id, location_id, period_start, demand_clean (base UoM ≥0), train_mask, period_state, needs_model_recon, outlier_cause`.

**`drivers`** (one row per item × location × period, history **and** future horizon):

| Column | Type | Unit | Req. | Known-in-advance |
|---|---|---|---|---|
| `price` | decimal | currency / base UoM | Y | if from locked price plan |
| `regular_price` | decimal | currency / base UoM | Y | yes |
| `promo_flag` | bool | n/a | Y | if from locked promo calendar |
| `promo_depth` | decimal [0,1] | (reg − promo)/reg | N | same |
| `display_flag`, `feature_flag` | bool | n/a | N | same |
| `holiday_flag`, `event_id` | bool / str | n/a | N | yes (calendar) |
| `temp_forecast`, `precip_forecast` | decimal | °C / mm | N | yes if forecast snapshot |
| `plan_snapshot_date` | date | n/a | Y for future-valued drivers | defines availability |
| `known_in_advance` | bool per driver column (or a driver metadata table) | n/a | Y | n/a |

**`item_attributes`:** `item_id, category, subcategory, brand, need_state, size, launch_date` (static categoricals for global models).

**`segmentation`:** route, class, ABC/XYZ. **`stat_backtest`:** the selected statistical model's backtest forecasts at the same origins and lags.

**Granularity:** SKU-Location-Week (default) or SKU-Location-Day for sensing. One grain per run.

**Minimum data:**
- ≥ 2 seasonal cycles at portfolio level.
- ≥ 30 related series sharing features for a global model [RoT].
- ≥ 3 promo events per item **or** pooling across category for lift estimates. Otherwise lift is estimated at category level and flagged `POOLED_LIFT`.
- If fewer than ~30 series: use **dynamic regression** (regression + ARIMA errors) per series instead of a global ML model.

## 4. Execution Workflow
1. **Scope:** take series with route `R_CAUSAL` (plus any user-requested set), and pull the statistical backtest for the same series.
2. **Feature build (per decision lag ℓ):**
   - Target lags: $\mathit{y}_{t-\ell}, y_{t-\ell-1},\dots$, and seasonal $\mathit{y}_{t-m}$ only if $\mathit{m} ≥ \ell$.
   - Rolling mean/SD/median over windows {4, 8, 13, 52} computed on `y.shift(ℓ)`.
   - Calendar: week-of-year, fiscal period, holiday distance, week-53 flag.
   - Price: log price, price index vs. regular price (= price/regular_price), price change vs. last period.
   - Promo: depth, display, feature, promo lead/lag dummies ±2 periods for pre/post dips.
   - Static categoricals.
   - Sibling features: share of need-state items on promo, and the min price index among siblings.
   - **Run the leakage checklist.**
3. **Loss selection:**
   - Intermittent share (ADI ≥ 1.32) > 30% of series, or counts → Tweedie (power tuned in {1.1, 1.3, 1.5}).
   - Otherwise L2 on raw values, or L2 on log1p with bias correction.
   - Quantiles: separate quantile models, or conformal residual quantiles by lag.
4. **Rolling-origin backtest (GS-M-16):** at each origin, train on periods ≤ origin − ℓ (targets known by then). Hold out the last 10% of the training time span for early stopping (time-ordered).
   - LightGBM default params [RoT]: learning_rate 0.05, num_leaves 63, min_data_in_leaf 50, feature_fraction 0.8, bagging_fraction 0.8, bagging_freq 1, ≤ 2,000 rounds with early stopping at 100.
   - Seeds fixed.
5. **Score** per series and lag: MASE, WAPE, Bias%, pinball, coverage. **FVA vs. NF1/NF2 and vs. the statistical model** (same origins).
6. **Decision per series** (deterministic):
   - If ML beats the statistical model by > 2% relative MASE, has |Bias%| ≤ 10%, and FVA_vs_naïve > 0 → `ML`.
   - Else if within ±2% → `COMBO` (equal weights).
   - Else → `STAT` (keep the baseline; ML not deployed for that series).
   - Also report the portfolio-level decision.
7. **Driver attribution:**
   - Gain/split importance and SHAP mean |value| on the last origin.
   - **Price elasticity** check: partial-dependence slope of log demand on log price index. Flag `ELASTICITY_SIGN` if it is positive for non-Veblen categories.
   - Promo lift = $\hat F - \hat B$, with $\hat B$ computed by counterfactual (promo off, regular price).
8. **Final training** on all data. Forecast $\mathit{h}=1..H$ using only known-in-advance future drivers. Drivers not known in advance are set to their lagged or seasonal value, and are flagged.
9. **Bias by segment:** ABC, class, category, promo vs. non-promo periods, new vs. mature items.
10. **Output** (Section 6). Persist `forecast_version = ML_<run_id>`, the feature list hash, and the per-origin snapshots for FVA (B1).

## 5. Verification Snippets
```python
import numpy as np, pandas as pd
import lightgbm as lgb   # if unavailable, use sklearn HistGradientBoostingRegressor

def make_features(df, lag, m=52):
    """df sorted by [item_id, location_id, period_start]; target y. All target-derived features shifted by >= lag."""
    g = df.groupby(["item_id", "location_id"])["y"]
    for k in range(lag, lag + 4):
        df[f"y_lag{k}"] = g.shift(k)
    if m >= lag:
        df["y_lag_season"] = g.shift(m)
    base = g.shift(lag)
    for w in (4, 8, 13, 52):
        df[f"roll_mean_{w}"] = base.groupby([df.item_id, df.location_id]).transform(lambda s: s.rolling(w, min_periods=1).mean())
    df["price_idx"] = df.price / df.regular_price
    df["log_price"] = np.log(df.price.clip(lower=1e-6))
    for k in (1, 2):   # pre/post promo dummies: future flags allowed only if known_in_advance
        df[f"promo_lead{k}"] = df.groupby(["item_id", "location_id"]).promo_flag.shift(-k)
        df[f"promo_lag{k}"]  = df.groupby(["item_id", "location_id"]).promo_flag.shift(k)
    return df

def leakage_assert(feature_cols, lag, known_in_advance):
    """Static guard. Target-derived features must be shifted >= lag; future-valued drivers must be
    known_in_advance (dict keyed by base driver column, e.g. {'price': True, 'promo_flag': True})."""
    base_of = {"price_idx": "price", "log_price": "price"}
    for c in feature_cols:
        if c.startswith("y_lag") and c != "y_lag_season":
            k = int(c[len("y_lag"):]); assert k >= lag, f"{c}: shift {k} < lag {lag}"
        if c == "y_lag_season":
            assert True  # built only when m >= lag (see make_features)
        if c.startswith("promo_lag") and int(c[len("promo_lag"):]) < lag:
            assert known_in_advance.get("promo_flag", False), f"{c}: shift < lag {lag} and promo plan not known in advance"
        if c.startswith("promo_lead") or c in ("price", "price_idx", "log_price", "promo_flag", "promo_depth"):
            b = "promo_flag" if c.startswith("promo_lead") else base_of.get(c, c)
            assert known_in_advance.get(b, False), f"{c}: driver '{b}' not known in advance -> lag it by {lag} or drop"

def rolling_origin_lgb(df, feats, origins, lag, H, params, seed=42):
    out = []
    for o in origins:                                   # o = last period whose actual is known at forecast time
        tr = df[(df.period_start <= o) & df.y.notna() & df.train_mask]
        te = df[(df.period_start > o + pd.Timedelta(weeks=lag - 1)) &
                (df.period_start <= o + pd.Timedelta(weeks=lag - 1 + H))]
        cut = tr.period_start.quantile(0.9)             # time-ordered early stopping split
        dtr = lgb.Dataset(tr[tr.period_start <= cut][feats], tr[tr.period_start <= cut].y)
        dva = lgb.Dataset(tr[tr.period_start > cut][feats], tr[tr.period_start > cut].y)
        mdl = lgb.train({**params, "seed": seed}, dtr, 2000, valid_sets=[dva],
                        callbacks=[lgb.early_stopping(100, verbose=False)])
        out.append(te.assign(origin=o, fcst=np.clip(mdl.predict(te[feats]), 0, None)))
    return pd.concat(out)

params_tweedie = dict(objective="tweedie", tweedie_variance_power=1.3, learning_rate=0.05, num_leaves=63,
                      min_data_in_leaf=50, feature_fraction=0.8, bagging_fraction=0.8, bagging_freq=1, verbose=-1)
params_q90 = {**params_tweedie, "objective": "quantile", "alpha": 0.9}
```
In this sketch, `make_features` is built for a single lag. For multi-lag scoring, build one feature frame per lag (the direct strategy), or restrict each frame to its own lag bucket.

```python
# FVA and decision rule (GS-K-07, step 6)
def decide(mase_ml, mase_stat, mase_naive, bias_ml):
    if mase_ml < 0.98 * mase_stat and abs(bias_ml) <= 10 and mase_ml < mase_naive: return "ML"
    if abs(mase_ml - mase_stat) / mase_stat <= 0.02 and mase_ml < mase_naive:     return "COMBO"
    return "STAT"
```
```sql
-- FVA stair-step (same items, origins, lag)
SELECT model,
       100.0*SUM(ABS(actual-fcst))/NULLIF(SUM(actual),0) AS wape_pct,
       100.0*SUM(fcst-actual)/NULLIF(SUM(actual),0)      AS bias_pct
FROM backtest_all
WHERE lag = :lag AND origin IN (SELECT origin FROM common_origins)
  AND (item_id, location_id) IN (SELECT item_id, location_id FROM scope)
GROUP BY model ORDER BY CASE model WHEN 'NF1' THEN 1 WHEN 'NF2' THEN 2 WHEN 'STAT' THEN 3 WHEN 'ML' THEN 4 ELSE 5 END;
```

## 6. Standardized Output Format
```markdown
# ML / Causal Demand Forecast — <scope> — run ML_<run_id>

## Executive Summary
- <N> series in scope; ML deployed on <a>%, COMBO <b>%, STAT retained <c>% (FVA rule).
- Portfolio WAPE (lag ℓ): NF <x>% → STAT <y>% → ML/COMBO <z>% (FVA vs STAT <y−z> pts); Bias <d>% (<over/under>).
- Top drivers: <price index, promo depth, holiday, …>; median price elasticity <ε> (<n> categories sign-checked).
- Leakage checklist: PASS (7/7) | flagged: <OPTIMISTIC_WEATHER …>.

## Metadata
| Level | Grain | Lag | Time window | Origins | Horizon | Weighting | Loss | Actuals basis |
|---|---|---|---|---|---|---|---|---|
| SKU-Location | WEEK | <ℓ> | <start–end> | <n> | <H> | WAPE volume; MASE median | Tweedie p=1.3 / quantile | unconstrained; censored excluded <p>% |

## FVA Stair-Step (same items / origins / lag)
| Step | WAPE % | Median MASE | Bias % (+=over) | Pinball P90 | 80% coverage | FVA vs prior (pts) | FVA vs naïve (pts) |
|---|---|---|---|---|---|---|---|
| Naïve (NF1/NF2) | | | | | | — | — |
| Statistical (selected) | | | | | | | |
| ML (global) | | | | | | | |
| Combination | | | | | | | |
| **Deployed** | | | | | | | |

## Bias & Accuracy by Segment
| Segment | N series | WAPE % | Bias % | MASE | Promo-period WAPE % | Non-promo WAPE % |
|---|---|---|---|---|---|---|

## Driver Attribution
| Feature | Gain share % | Mean |SHAP| | Direction | Known-in-advance | Notes |
|---|---|---|---|---|---|---|

## Promo Baseline/Lift Handoff (to nextq-promotion-lift-cannibalization-planning)
| item | loc | period | forecast | baseline (counterfactual) | lift units | lift % | promo_depth | pre/post window |
|---|---|---|---|---|---|---|---|---|

## Exceptions
| ID | Trigger (rule, threshold) | item/segment | metric | value at risk | action | owner |
|---|---|---|---|---|---|---|
| | FVA<0 vs STAT (GS-K-07) | | | | keep STAT; review features | |
| | |Bias|>10% 3 origins (GS-K-05) | | | | recalibrate / check censoring | |
| | MASE>1 (GS-G-03) | | | | | |
| | LEAKAGE_SUSPECT (GS-M-09) | | | | block deployment | |
| | ELASTICITY_SIGN (positive own-price) | | | | investigate confounding (promo×price) | |
| | Coverage off >5pp (GS-K-14) | | | | conformal recalibration | |
| | POOLED_LIFT (<3 promo events) | | | | category-level lift; low confidence | |

## Recommendations
1. …

## Data-Quality Log
| item | loc | censored periods excluded | reconstructed used (weight) | drivers not known-in-advance (handled how) | notes |

## Reproducibility
Feature list hash <h>; params; seeds; origins; library versions (lightgbm x.y); training windows; code §5.
```

## 7. Edge Cases
- **Promo and price collinear** (every promo = price cut): use promo depth only, or price index only, and document the choice. Do not report separate effects (`CONFOUNDED_DRIVERS`).
- **New items with no history:** a global model with static attributes may forecast them, but the result is routed to and approved by `nextq-new-product-lifecycle-forecasting`, and flagged `COLD_START`.
- **Structural breaks** (for example, COVID-like periods): add a regime indicator, or exclude the window from training with documentation. Never silently down-weight it.
- **Daily grain:** day-of-week, payday, and holiday-proximity features. Lag ℓ is in days. Aggregate to weekly for S&OP via sample-path or simple sums of means (GS-M-10).
- **Intermittent items inside a global model:** Tweedie loss. Compare against SBA/TSB from `nextq-intermittent-demand-forecasting` on RMSSE; MAE-type metrics reward zero forecasts.
- **ML beats the statistical model only at aggregate level:** the deployment decision is made at **decision level** (item-location-lag). Report the aggregate separately; never mix the two.
