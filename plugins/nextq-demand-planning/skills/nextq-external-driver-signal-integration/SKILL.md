---
name: nextq-external-driver-signal-integration
description: >-
  [Next Quarter Supply Chain Suite] Screens, encodes and validates external and causal demand drivers - weather
  (anomalies vs climatology), holidays and moving events, price and own-price elasticity, macro, search or social
  signals - as leakage-safe forecasting features. Classifies each driver by what is knowable at forecast time
  (known-future, forecasted with archived vintages, realized-only with minimum lags) and admits a driver group only if
  rolling-origin backtests show positive value added by lag with stable coefficient signs. Load when the user asks to
  add weather, holidays, events, price, economic or search data to a forecast, test whether a driver improves
  accuracy, estimate price elasticity, build a driver catalog, or check a model for look-ahead leakage. Inputs: demand
  history, driver series with vintages or publication lags, event and price calendars, geo mapping. Triggers: "weather
  impact", "external drivers", "price elasticity", "holiday effect", "feature leakage".
metadata:
  author: "Next Quarter"
  organization: "Next Quarter (nextq)"
  suite: "Next Quarter Supply Chain Intelligence"
  version: "1.0.0"
---

# External Driver & Signal Integration
*Part of the **Next Quarter (nextq)** Supply Chain Skill Suite — Governed by SUPPLY_CHAIN_GOLD_STANDARD.md.*

## 1. Role & Objective
**Persona:** Demand Data Scientist (feature engineering and causal drivers).
**Decision supported:** Which external signals genuinely improve the forecast at each lag, how should they be encoded, and can they be used in production without look-ahead leakage?

**Objective:**
- A governed **driver catalog** and feature set, with each driver's class, as-of availability, encoding, backtested FVA by lag, sign stability, and an accept/reject decision.
- Price elasticity estimates for the promotion and financial skills.
- A feature-store specification consumed by `nextq-ml-causal-demand-forecasting`, `nextq-statistical-baseline-forecasting` (dynamic regression) and `nextq-short-horizon-demand-sensing`.

## 2. Gold Standard Rules Enforced
| Rule | Inlined standard |
|---|---|
| **GS-M-09** ML / causal controls | Time-ordered rolling-origin validation only (no random K-fold). **No look-ahead leakage:** features must be known at forecast time $t-\ell$. Future promo and price are allowed only if **planned and locked**. A model with drivers must beat NF1/NF2 **and** the best statistical model (FVA ≥ 0). Report importance or coefficients, and bias by segment. |
| **GS-M-16** Backtesting | Rolling origin (expanding or sliding). The test horizon covers the maximum decision horizon. Score **by lag**; never average across lags without reporting each. Minimum 3 origins, or 1 seasonal cycle of origins for seasonal items [RoT]. |
| **GS-K-07** FVA | $\text{FVA}_\ell=\text{WAPE}_\ell(\text{without drivers})-\text{WAPE}_\ell(\text{with drivers})$, same items, lag and window. WAPE $=\sum\lvert A-F\rvert/\sum A$. |
| **GS-D-05** Price / promo | Promo and price effects are estimated with regression terms (or masked baselines). Lift = $A-B$. Own-price elasticity from $\ln Q=a+\varepsilon\ln P+\dots$ Promo lift libraries live in `nextq-promotion-lift-cannibalization-planning`; this skill supplies elasticity and the price feature. |
| **GS-D-10** Calendars & events | 53rd week handled explicitly (drop, merge or dummy). **Moving holidays** (Easter, Lunar New Year, Diwali, Eid, Thanksgiving) need event dummies placed by the actual date in each year, not by week number. Year-over-year features align like weeks. |
| **GS-D-09** Buckets | Daily drivers are aggregated to the demand bucket on the fiscal calendar, using the same week-ending day (mean for intensities such as temperature, sum for counts such as rainfall days). |
| **GS-M-11** Censoring | Driver effects are estimated on unconstrained demand. Stockout periods would bias elasticity and weather effects toward zero. |
| **GS-G-10 / GS-G-11** | Standard output; every accepted feature is reproducible from its as-of snapshot. |

**Driver classes and leakage gate (deterministic):**

| Class | Examples | Usable at horizon $h$ | Backtest must use |
|---|---|---|---|
| `KNOWN_FUTURE` | calendar, holidays, store openings, **locked** planned price and promo | any $h$ | the plan as it stood at the origin |
| `FORECASTED` | weather forecasts, macro forecasts | only with **archived vintages** | the vintage issued ≤ origin; realized values are an oracle (`ORACLE_RISK`, not production-eligible) |
| `REALIZED_ONLY` | search trends, social, competitor prices, others' POS | lags ≥ $h$ + publication lag | lagged values only |

**Acceptance rule [RoT]:** a driver group is accepted for a lag if:
- FVA ≥ 0.5 WAPE points,
- it beats the no-driver model in ≥ 2/3 of origins, and
- coefficient signs are stable in ≥ 80% of origins.

Otherwise it is rejected or kept only as a monitored candidate.

**Elasticity plausibility [RoT]:**
- Requires ≥ 3 distinct prices and price CV ≥ 2%.
- Own-price elasticity must fall in [−6, 0). Positive values → `IMPLAUSIBLE_SIGN`; below −6 → `IMPLAUSIBLE_MAGNITUDE`.
- Use category pooling or hierarchical shrinkage when an item fails.

## 3. Required Inputs & Data Schema
| Table | Columns | Notes |
|---|---|---|
| `demand_clean` | `item_id, location_id, period_start, demand_clean, period_state, promo_flag, price` | unconstrained (reconstructed) |
| `driver_catalog` | `driver_id, class, source, unit, publish_lag_periods, has_vintages, locked_flag, geo_level, aggregation (mean/sum/max)` | one row per driver |
| `driver_values` | `driver_id, geo_id, valid_time, issue_time (vintage), value` | `issue_time` is required for FORECASTED |
| `climatology` | `geo_id, day_of_year, temp_normal (≥ 10 reference years [RoT])` | for anomalies |
| `event_calendar` | `event_id, event_type, date, geo_scope, pre_periods, post_periods` | moving holidays per year |
| `geo_map` | `location_id → weather station / region` | nearest station within 50 km [RoT] |
| `baseline_backtest` | stored no-driver forecasts by origin and lag | comparator |

**Minimum:** ≥ 2 seasonal cycles for weather or holiday effects; ≥ 13 origins for acceptance; ≥ 3 distinct price points for elasticity.

## 4. Execution Workflow
1. **Catalog and gate.**
   - Classify each driver and apply the leakage gate per lag.
   - Drivers that fail the gate (`NOT_LOCKED`, `ORACLE_RISK`) are excluded from production features. They can be studied only as oracle upper bounds, labeled as such.
2. **Align.** Map location → geo, then aggregate daily drivers to the fiscal bucket (calendar table; mean or sum per the catalog). Assert there are no missing buckets; gaps longer than 2 buckets → `DRIVER_GAP` (no forward-fill beyond 2).
3. **Encode.**
   - **Weather:** anomaly = value − climatology(day of year), plus optional threshold terms (e.g. heat days above P90 [RoT]).
   - **Events:** pre / event / post dummies placed by the actual date. The 53rd week gets its own dummy.
   - **Price:** $\ln P$ (or the relative price index versus the reference price), plus a promo flag.
   - **Realized-only drivers:** shift by the minimum lag.
   - **Macro:** growth rates rather than levels.
4. **Elasticity** per item (or pooled) with seasonal and promo controls. Apply the plausibility checks and publish $\varepsilon$ with its status.
5. **Screen** each driver by its correlation with the baseline model's residuals at allowed lags (diagnostic only; never an acceptance criterion).
6. **Backtest:**
   - For each driver group, add it to the baseline design and run the rolling-origin backtest at each decision lag.
   - Compute WAPE with and without the drivers, FVA, the share of origins won, and sign stability.
   - Groups are added in a fixed order (calendar/events → price/promo → weather → macro → realized-only) with forward selection; each must pass the acceptance rule given the groups already in.
7. **Decide and publish.**
   - Accepted groups go into the feature-store spec (driver, encoding, lag, as-of rule). Rejected groups are listed with the reason.
   - Re-run acceptance quarterly [RoT] or when a source changes.
8. **Monitor.** Track driver drift (mean or variance shift of the driver > 3×IQR versus the training window) and source outages. If a production driver is missing at run time, fall back to the no-driver model and flag `DRIVER_FALLBACK`.
9. **Output** (Section 6).

## 5. Verification Snippets
```python
import numpy as np, pandas as pd

DRIVER_CLASSES = ("KNOWN_FUTURE", "FORECASTED", "REALIZED_ONLY")

def usable(driver_class, h, publish_lag=0, has_vintages=False, locked=False):
    """GS-M-09 leakage gate for horizon h (periods ahead of origin t0).
    KNOWN_FUTURE (calendar, holidays, *locked* planned price/promo): usable at any h.
    FORECASTED (weather, macro forecasts): usable only with archived as-of vintages; else ORACLE_RISK.
    REALIZED_ONLY (search, social, POS of others): only lags >= h + publish_lag, i.e. value known at t0."""
    if driver_class == "KNOWN_FUTURE":
        return (True, "OK") if locked or publish_lag == 0 else (False, "NOT_LOCKED")
    if driver_class == "FORECASTED":
        return (True, "VINTAGE") if has_vintages else (False, "ORACLE_RISK")
    if driver_class == "REALIZED_ONLY":
        return True, f"MIN_LAG_{h + publish_lag}"
    raise ValueError(driver_class)

def lagged_feature(x, min_lag):
    """Shift a realized-only driver so row t only sees x_{t-min_lag}."""
    return pd.Series(x).shift(min_lag)

def weather_anomaly(temp, doy, years_clim):
    """Temperature anomaly vs climatology (mean by day-of-year over reference years), not raw temperature."""
    clim = pd.Series(years_clim["temp"].values, index=years_clim["doy"].values).groupby(level=0).mean()
    return np.asarray(temp, float) - clim.reindex(np.asarray(doy)).to_numpy()

def fit_elasticity(q, p, controls=None):
    """Log-log OLS: ln Q = a + e ln P + controls. Requires >= 3 distinct prices and CV(price) >= 2% [RoT].
    Plausible own-price elasticity window [-6, 0) [RoT]; positive -> IMPLAUSIBLE_SIGN."""
    q, p = np.asarray(q, float), np.asarray(p, float)
    if len(np.unique(np.round(p, 4))) < 3 or p.std() / p.mean() < 0.02:
        return np.nan, "INSUFFICIENT_PRICE_VARIATION"
    X = np.c_[np.ones(len(q)), np.log(p)] if controls is None else np.c_[np.ones(len(q)), np.log(p), controls]
    b, *_ = np.linalg.lstsq(X, np.log(q), rcond=None)
    e = b[1]
    return e, ("OK" if -6 <= e < 0 else "IMPLAUSIBLE_SIGN" if e >= 0 else "IMPLAUSIBLE_MAGNITUDE")

def rolling_origin_fva(y, X_base, X_drv, origins, h=1):
    """Rolling-origin backtest (expanding window, OLS dynamic regression proxy). Returns WAPE base, WAPE with drivers,
    FVA = WAPE_base - WAPE_drv, share of origins where drivers helped, and coefficient-sign stability of the drivers."""
    y = np.asarray(y, float); eb, ed, wins, signs = [], [], 0, []
    for t0 in origins:
        tr, te = slice(0, t0), t0 + h - 1
        Xb, Xd = X_base, np.c_[X_base, X_drv]
        bb, *_ = np.linalg.lstsq(Xb[tr], y[tr], rcond=None)
        bd, *_ = np.linalg.lstsq(Xd[tr], y[tr], rcond=None)
        fb, fd = Xb[te] @ bb, Xd[te] @ bd
        eb.append(abs(y[te] - fb)); ed.append(abs(y[te] - fd)); wins += abs(y[te] - fd) < abs(y[te] - fb)
        signs.append(np.sign(bd[X_base.shape[1]:]))
    A = np.abs(y[[t0 + h - 1 for t0 in origins]]).sum()
    wb, wd = 100 * np.sum(eb) / A, 100 * np.sum(ed) / A
    S = np.array(signs); stab = (S == np.sign(S.sum(axis=0))).mean(axis=0)
    return dict(wape_base=wb, wape_drivers=wd, fva=wb - wd, win_share=wins / len(origins), sign_stability=stab)

def accept_driver(res, min_fva=0.5, min_win=2/3, min_stab=0.8):
    """[RoT] keep a driver group only if FVA >= 0.5 WAPE pts, helps in >= 2/3 of origins, signs stable >= 80%."""
    return bool(res["fva"] >= min_fva and res["win_share"] >= min_win and np.all(res["sign_stability"] >= min_stab))

def event_dummies(dates, event_dates, pre=1, post=1, freq="W"):
    """Event window dummies (pre / event / post) aligned to period starts; moving holidays come from the event table."""
    idx = pd.DatetimeIndex(dates); out = pd.DataFrame(0, index=idx, columns=["pre", "event", "post"])
    step = pd.Timedelta(weeks=1) if freq == "W" else pd.Timedelta(days=1)
    for d in pd.DatetimeIndex(event_dates):
        pos = idx.searchsorted(d, side="right") - 1          # period containing the event
        if pos < 0: continue
        out.iloc[pos, 1] = 1
        for k in range(1, pre + 1):
            if pos - k >= 0: out.iloc[pos - k, 0] = 1
        for k in range(1, post + 1):
            if pos + k < len(idx): out.iloc[pos + k, 2] = 1
    return out
```
**Test vector** (executed, seed 11):
- **Leakage gate:**
  - Locked planned price → OK. Unlocked → `NOT_LOCKED`.
  - Weather forecast without vintages → `ORACLE_RISK`; with vintages → VINTAGE.
  - Search data at h = 3 with a 1-period publication lag → `MIN_LAG_4`.
- **Lagging:** lag 2 of [1..5] → [NaN, NaN, 1, 2, 3].
- **Weather anomaly:** temperatures 15 and 9 against normals 11 and 12 → [+4, −3].
- **Elasticity:** 80 weeks with prices {1.8, 2.0, 2.2, 2.5} and true ε = −1.8 → **−1.733 (OK)**. A constant price → `INSUFFICIENT_PRICE_VARIATION`; a positive true slope → `IMPLAUSIBLE_SIGN`.
- **Backtest:** 156 weeks with a +3 units/°C weather effect, 48 one-step origins. The temperature series is treated as a vintage-available driver here.
  - WAPE 6.24 → 1.40 (FVA **+4.84**), wins 83%, sign stability 100% → **accepted**.
  - A pure-noise driver: FVA −0.13, wins 40% → **rejected**.
- **Events:** Easter on 5 Apr 2026 against weekly periods from 2 Mar → event dummy in the week starting 30 Mar, with pre and post weeks around it.

```sql
-- Leakage audit: any FORECASTED driver value used in a backtest must have been issued at or before the origin
SELECT f.model_run_id, f.origin_time, f.driver_id, f.valid_time, v.issue_time
FROM feature_usage f
JOIN driver_values v ON v.driver_id = f.driver_id AND v.geo_id = f.geo_id AND v.valid_time = f.valid_time
                    AND v.issue_time = f.issue_time_used
JOIN driver_catalog c ON c.driver_id = f.driver_id
WHERE (c.class = 'FORECASTED'    AND v.issue_time > f.origin_time)                                -- future vintage
   OR (c.class = 'REALIZED_ONLY' AND f.valid_time > f.origin_time - c.publish_lag_periods * INTERVAL '1 week')
   OR (c.class = 'KNOWN_FUTURE'  AND c.locked_flag = FALSE AND f.valid_time > f.origin_time);      -- unlocked plan
-- Expected result: zero rows. Any row = LEAKAGE (model run invalid).
```

## 6. Standardized Output Format
```markdown
# External Driver Integration — <scope> — run DRV_<run_id>

## Executive Summary
- <N> drivers evaluated: <a> accepted (groups: <list>), <r> rejected, <o> excluded by leakage gate (ORACLE_RISK / NOT_LOCKED).
- Best gain: <group> FVA +<x> WAPE pts at lag <ℓ> (wins <w>%); weather effect <β> units/°C anomaly.
- Price elasticity median <ε> (<n_ok> OK, <n_bad> implausible → pooled).
- Leakage audit: <0 rows ✔ / n violations ✖>.

## Metadata (mandatory)
| Level | Grain | Time window | Origins | Lags scored | Model for test | Weighting | Driver vintages | Geo mapping |
|---|---|---|---|---|---|---|---|---|
| SKU-Location (pooled by <segment>) | WEEK | <start–end> | <n> (expanding) | <list> | dynamic regression / GBM | volume-weighted WAPE | archived since <date> | station ≤50 km |

## Driver Catalog & Decisions
| driver | class | gate status | encoding | min lag | FVA lag1 | FVA lagL | win % | sign stability | decision | reason |
|---|---|---|---|---|---|---|---|---|---|---|

## Elasticity
| item / pool | ε | status | N weeks | distinct prices | price CV | controls |
|---|---|---|---|---|---|---|

## KPI Table (per lag)
| Lag | WAPE no drivers % | WAPE with accepted drivers % | FVA pts | Bias % (+=over) | vs NF1 FVA |
|---|---|---|---|---|---|

## Exceptions
| ID | Trigger (rule) | driver / item | metric | value | action | owner |
|---|---|---|---|---|---|---|
| | LEAKAGE (GS-M-09) | | audit rows | | invalidate run | Data scientist |
| | ORACLE_RISK (GS-M-09) | | | | archive vintages before use | Data eng |
| | IMPLAUSIBLE elasticity (GS-D-05) | | ε | | pool / review price data | Pricing analyst |
| | DRIVER_GAP / DRIVER_FALLBACK | | | | fix source | Data eng |
| | Driver drift >3×IQR | | | | re-validate | Data scientist |

## Recommendations
1. …

## Data-Quality Log
| driver | issue (missing vintages, gaps, station distance, unit changes) | treatment |
|---|---|---|

## Reproducibility
Catalog version, vintage snapshot, origins, lags, acceptance thresholds, seed, code §5.
```

## 7. Edge Cases
- **Weather at national level:** never average temperatures across regions with opposite seasons or climates. Model at the regional grain, then reconcile (`nextq-hierarchical-forecast-reconciliation`).
- **Holidays falling across a week boundary:** apportion the event dummy by day share, or use daily modeling.
- **Collinear drivers** (temperature and month seasonality): weather enters as an **anomaly**, so seasonality stays in the seasonal terms. If the condition number of the design is above 1,000 [RoT], drop or combine terms.
- **Price endogeneity** (price cut in response to weak demand): flag `ENDOGENEITY_RISK` when price changes follow demand drops, and prefer planned, exogenous price events for estimation.
- **Search or social data rescaled by the provider** (relative indices): re-download a consistent window and use growth rates.
- **New items:** use pooled category elasticities and driver effects; never fit item-level effects on fewer than 26 weeks.
- **COVID-type regime breaks:** exclude the break window or add regime dummies (`nextq-demand-anomaly-shift-detection`), and do not let it dominate driver coefficients.
