---
name: nextq-demand-history-cleansing
description: >-
  [Next Quarter Supply Chain Suite] Builds the canonical, auditable demand history that every forecasting, planning,
  sensing and inventory skill consumes. Load when the user asks to clean, prepare, validate, or "fix" sales/demand
  history; to flag stockouts, censored sales, missing weeks, outliers, or returns; to convert units of measure; to
  chain superseded SKUs; or before any forecast, segmentation, or safety-stock run. Inputs: POS, orders, or shipment
  history at SKU-Location-Day/Week/Month grain with on-hand / in-stock %, promo and price flags, lifecycle status, UoM
  and supersession tables, fiscal calendar. Triggers: "clean my demand data", "remove outliers", "stockouts are
  distorting history", "missing data", "zero sales", "prepare data for forecasting", "data quality check",
  "unconstrained demand".
metadata:
  author: "Next Quarter"
  organization: "Next Quarter (nextq)"
  suite: "Next Quarter Supply Chain Intelligence"
  version: "1.0.0"
---

# Demand History Cleansing
*Part of the **Next Quarter (nextq)** Supply Chain Skill Suite — Governed by SUPPLY_CHAIN_GOLD_STANDARD.md.*

## 1. Role & Objective
**Persona:** Senior Demand Data Steward / Forecasting Data Engineer.
**Decision supported:** Is this history fit to train models and set inventory policy? Which periods are true demand, which are structural or missing, and which are censored by stockouts?
**Objective:** Produce a canonical, immutable-source, fully lineage-logged demand table. Its key is `item × location × period`, in base UoM. Every period carries one of four states: `TRUE_ZERO`, `STRUCTURAL_ZERO`, `MISSING`, `CENSORED`, or `OBSERVED`. Outliers are explained, and every change is reversible.

This skill **flags** censoring and applies only the GS-M-11 methods (a) and (b), labeled `INTERIM_RECON`. Rows it cannot resolve carry `needs_model_recon = true`; `nextq-unconstrained-demand-reconstruction` completes them (implied orders, EM/Tobit, model imputation, substitution netting) with the same thresholds and never overwrites (a)/(b) results.

## 2. Gold Standard Rules Enforced
Source: SUPPLY_CHAIN_GOLD_STANDARD.md (rules inlined below). Labels: [STD] = authoritative, [RoT] = configurable rule of thumb.

| Rule | What this skill enforces (inlined) |
|---|---|
| **GS-D-01** Canonical grain | Key = `item_id × location_id × period_start` (+`channel` if present). All quantities are in **base UoM** via an auditable conversion table. Standard grains: SKU-Location-**Day** (POS/sensing), SKU-Location-**Week** (operational), Family-Region-**Month** (S&OP). |
| **GS-D-02** Zero ≠ missing ≠ censored | **True zero:** listed, available, no demand → keep 0. **Structural zero:** not ranged, store closed, pre-launch, discontinued → mask, never train on it. **Missing:** no record → impute + flag. **Censored:** sales limited by availability → reconstruct (GS-M-11). |
| **GS-M-11** Censoring detection | A period is censored if `on_hand ≤ 0` (start or end) OR `in_stock_pct < 1.00` OR `backorder_qty > 0` OR `order_cut_qty > 0`. It is **materially censored** if `in_stock_pct < 0.95` [RoT]. Reconstruction priority: **(a)** use orders (requested qty and date) when available. **(b)** availability scaling $\hat D_t = S_t / a_t$, with $a_t$ = demand-profile-weighted in-stock fraction, uplift **capped at ×2**, and not applied when $a_t < 0.5$ (send to method (c)). **(c)** EM/Tobit, deferred to C2. **(d)** model interpolation, marked imputed. If **> 30%** of an item's periods are reconstructed, flag `LOW_CONFIDENCE` [RoT]. |
| **GS-D-03** Missing values | Informative missingness (for example, holiday closure) → mark it as an event and do not impute. Random missingness → model-based or seasonal-profile interpolation. **Never zero-fill** a missing value. If **> 20%** of history is missing or any gap is **> 1 seasonal cycle**, use only post-gap history or flag the series [RoT]. |
| **GS-D-04** Outliers | Robust STL-type decomposition (default: median engine, §5.2; `STL(robust=True)` as a cross-check). An outlier is a remainder outside $[Q_1 - 3\,\text{IQR},\ Q_3 + 3\,\text{IQR}]$ (FPP3 §13.9 preferred rule; 1.5×IQR is the loose alternative). Explain it before treating it. Known recurring causes (promo, event) are kept and flagged as regressors. Confirmed data errors or non-recurring one-offs are replaced by interpolation. For intermittent series (ADI ≥ 1.32), skip STL and use 3×IQR on **non-zero sizes only**. |
| **GS-D-12** Lifecycle & master data | Supersession: chain old-SKU history into the new SKU using `conversion_factor`. Pre-launch and post-discontinuation periods become `STRUCTURAL_ZERO`. |
| **GS-D-11** Signal hierarchy | Prefer POS/sell-out, then orders (requested date), then shipments. Shipments are a constrained proxy, so never use them as demand when orders exist. |
| **GS-D-13** Lineage | Every change is logged with `before, after, rule_id, method, timestamp, run_id`. The raw table is immutable. |
| **GS-G-11** Verification | All row counts, sums, and state shares must be reproducible from the stated SQL/Python. Raw total = cleaned total − Σ(adjustments) must reconcile exactly. |
| **GS-G-10** Output contract | Executive summary, KPI tables with metadata, exceptions, recommendations, data-quality log, reproducibility block. |

## 3. Required Inputs & Data Schema

### 3.1 `demand_raw`: one row per item × location × period (required)
| Column | Type | Unit / domain | Required | Notes |
|---|---|---|---|---|
| `item_id` | string | n/a | Y | SKU |
| `location_id` | string | n/a | Y | Store / DC / plant / customer-ship-to |
| `channel` | string | n/a | N | Part of key if present |
| `period_start` | date (ISO) | n/a | Y | Must match `period_grain` boundary in the fiscal calendar |
| `period_grain` | enum | `DAY`/`WEEK`/`MONTH` | Y | One grain per run |
| `demand_qty` | decimal ≥ 0 | source UoM | Y | Gross demand (returns excluded) |
| `return_qty` | decimal ≥ 0 | source UoM | N | Kept separate; never netted into demand |
| `demand_type` | enum | `POS`/`ORDERS`/`SHIPMENTS` | Y | GS-D-11 |
| `uom` | string | n/a | Y | Converted with `uom_conversion` |
| `orders_qty` | decimal ≥ 0 | source UoM | N | Requested qty on requested date (enables method a) |
| `on_hand_qty` | decimal | source UoM | Y* | Start or end of period. *Required for censoring. If absent, flag `CENSORING_UNASSESSABLE`. |
| `in_stock_pct` | decimal [0,1] | fraction | N | Fraction of period (or store-days) in stock |
| `backorder_qty`, `order_cut_qty` | decimal ≥ 0 | source UoM | N | |
| `promo_flag` | bool | n/a | Y | |
| `event_flags` | string[] | n/a | N | For example, `HOLIDAY`, `CLOSURE`, `WEATHER` |
| `price`, `list_price` | decimal | currency / base UoM | N | |
| `lifecycle_status` | enum | `NEW`/`ACTIVE`/`PHASE_OUT`/`DISCONTINUED`/`NOT_RANGED` | Y | Per period |
| `store_open_flag` | bool | n/a | N | Default true |

### 3.2 Reference tables
- `uom_conversion(item_id, from_uom, to_uom, factor)`: base UoM factor. Missing factor → **stop** and report.
- `supersession(old_item_id, new_item_id, effective_date, conversion_factor)`
- `fiscal_calendar(date, fiscal_week, fiscal_period, fiscal_quarter, fiscal_year, week_start, days_in_period)`: week-start day explicit; 4-4-5 / 53-week aware.

### 3.3 Minimum history
- At least **2 full seasonal cycles** (104 weeks, 24 months, or 728 days) to run decomposition-based outlier detection.
- At least **1 cycle** to be forecastable at all.
- With 1–2 cycles, use 3×IQR on raw non-zero values and flag `SHORT_HISTORY` [RoT].

## 4. Execution Workflow (deterministic order, do not reorder)

**Step 0: Snapshot and validate.**
1. Copy the raw input to `demand_raw_snapshot_<run_id>` (immutable).
2. Check the schema: types, enums, and `demand_qty ≥ 0`.
   - **Negative quantities:** move `abs(min(qty,0))` to `return_qty`, set demand to 0, and log rule `GS-D-01/NEG_TO_RETURN`.
3. Duplicate keys: if rows have identical values, keep one. If values conflict, stop and report.
4. Period alignment: every `period_start` must equal a calendar boundary. Misaligned rows go to C6-style day apportionment: split by days and log.

**Step 1: Unit of measure.** Compute `qty_base = qty × factor`. If any item lacks a factor, stop: `EXC_UOM_MISSING`.

**Step 2: Supersession chaining (GS-D-12).**
- For each chain old → new effective at date $d$, append the old history before $d$ to the new SKU as `qty × conversion_factor`, with `lineage.rule = GS-D-12/SUPERSESSION`.
- Detect cycles in the chain (A → B → A). If found, stop and report.

**Step 3: Complete the grid.** For each item-location, build every calendar period from `first_listed_date` to `run_end_date`. Absent rows are inserted with `demand_qty = NULL`.

**Step 4: Assign period state (GS-D-02).** Apply these in precedence order:
1. `lifecycle_status ∈ {NOT_RANGED, DISCONTINUED}`, OR before launch, OR `store_open_flag = false` → **STRUCTURAL_ZERO**.
2. `demand_qty IS NULL` → **MISSING**.
3. Censoring test (GS-M-11): `on_hand_qty ≤ 0` OR `in_stock_pct < 1` OR `backorder_qty > 0` OR `order_cut_qty > 0` → **CENSORED**. Additionally set `material = in_stock_pct < 0.95`. If `on_hand_qty` is NULL and `in_stock_pct` is NULL → `CENSORING_UNASSESSABLE` (state stays OBSERVED and the run is flagged).
4. `demand_qty = 0` → **TRUE_ZERO**.
5. Otherwise → **OBSERVED**.

**Step 5: Interim reconstruction of censored periods (GS-M-11 a/b only).**
- (a) If `orders_qty` is present and `demand_type ≠ ORDERS`: set `demand_clean = orders_qty_base` (method `ORDERS`).
- (b) Otherwise, if `in_stock_pct` (= $a_t$) satisfies $0.5 ≤ a_t < 1$: set `demand_clean = min(demand / a_t, 2 × demand)` (method `AVAIL_SCALE`).
  - Use the day-of-week profile to weight $a_t$ if daily availability is known.
- If $a_t < 0.5$, or $a_t$ is unknown but `on_hand ≤ 0`: leave `demand_clean = demand` and set `needs_model_recon = true` for C2 (method c/d).
- Compute `pct_reconstructed` per item-location. If it is > 30% → `LOW_CONFIDENCE`.

**Step 6: Missing values (GS-D-03).**
- If a MISSING period has an `event_flags` entry with `CLOSURE` or `HOLIDAY`: treat it as informative and convert it to STRUCTURAL_ZERO with an event dummy.
- If `missing_share > 20%` or `max_gap > seasonal_period`: truncate the series to after the last long gap if the remaining length is ≥ 1 cycle. Otherwise flag `INSUFFICIENT_HISTORY`.
- Otherwise interpolate: seasonal-profile interpolation (seasonal component from STL + linear trend) or state-space/ARIMA interpolation. Mark the method `IMPUTED`. **Never 0.**

**Step 7: Outliers (GS-D-04).**
1. Choose the series to test: `demand_clean` over OBSERVED, TRUE_ZERO, and reconstructed periods (structural periods excluded).
2. If ADI < 1.32 and length ≥ 2 cycles: compute the robust-decomposition remainder (median engine, §5.2; robust STL as a cross-check) and the bounds $[Q_1 - 3\text{IQR}, Q_3 + 3\text{IQR}]$.
   - Intermittent series (ADI ≥ 1.32): bounds on non-zero sizes only.
   - Short series: bounds on raw non-zero values.
3. For each flagged period, assign a cause in this order:
   - `promo_flag` → `PROMO` (keep; regressor)
   - `event_flags` → `EVENT` (keep; regressor)
   - single-order share > 50% of period qty, if order lines are available → `BULK_ONE_OFF` (replace) [RoT]
   - data-error evidence (UoM ×10/×12/×100 pattern, duplicate load) → `DATA_ERROR` (replace)
   - otherwise → `UNEXPLAINED` (keep, flag for planner review; do **not** auto-replace) [decision: conservative]
4. Replacement value = robust trend + seasonal profile at $t$ (or the median of non-zero sizes for intermittent series), floored at 0.

**Step 8: Lineage and reconciliation (GS-D-13, GS-G-11).**
- Write one lineage row per changed cell.
- Assert: $\sum \text{demand\_clean} = \sum \text{demand\_raw\_base} + \sum \text{adjustments}$, with tolerance $10^{-6}$ relative.

**Step 9: Emit outputs** (Section 6) and the clean table `demand_clean`:
`item_id, location_id, channel, period_start, period_grain, demand_raw_base, demand_clean, period_state, censored_material, recon_method, imputed_flag, outlier_flag, outlier_cause, needs_model_recon, train_mask` (true for OBSERVED, TRUE_ZERO, reconstructed, and imputed periods; false for STRUCTURAL_ZERO).

## 5. Verification Snippets

### 5.1 SQL: period-state shares and censoring
```sql
-- GS-D-02 / GS-M-11 state assignment (precedence matters)
SELECT item_id, location_id, period_start,
  CASE
    WHEN lifecycle_status IN ('NOT_RANGED','DISCONTINUED') OR store_open_flag = FALSE
         OR period_start < first_launch_date                                    THEN 'STRUCTURAL_ZERO'
    WHEN demand_qty IS NULL                                                     THEN 'MISSING'
    WHEN on_hand_qty <= 0 OR in_stock_pct < 1.0
         OR COALESCE(backorder_qty,0) > 0 OR COALESCE(order_cut_qty,0) > 0      THEN 'CENSORED'
    WHEN demand_qty = 0                                                         THEN 'TRUE_ZERO'
    ELSE 'OBSERVED'
  END AS period_state,
  (in_stock_pct < 0.95) AS censored_material
FROM demand_grid;

-- Reconciliation assertion (GS-G-11): must return zero rows
SELECT item_id, location_id
FROM (SELECT item_id, location_id, SUM(demand_clean) c, SUM(demand_raw_base) r FROM demand_clean GROUP BY 1,2) s
JOIN (SELECT item_id, location_id, SUM(after - before) adj FROM lineage GROUP BY 1,2) l USING (item_id, location_id)
WHERE ABS(c - (r + adj)) > 1e-6 * GREATEST(ABS(r),1);
```

### 5.2 Python: availability scaling and 3×IQR robust-decomposition outliers
```python
import numpy as np, pandas as pd
def interim_recon(df):
    """GS-M-11 (a)/(b). df: one item-location, sorted by period_start."""
    out = df.copy()
    cens = out.period_state.eq("CENSORED")
    has_orders = out.orders_qty_base.notna()
    a = out.in_stock_pct
    # (a) orders
    m_a = cens & has_orders
    out.loc[m_a, "demand_clean"] = out.loc[m_a, "orders_qty_base"]; out.loc[m_a, "recon_method"] = "ORDERS"
    # (b) availability scaling, 0.5 <= a < 1, cap x2
    m_b = cens & ~has_orders & a.between(0.5, 1, inclusive="left")
    out.loc[m_b, "demand_clean"] = np.minimum(out.loc[m_b, "demand_base"] / a[m_b], 2 * out.loc[m_b, "demand_base"])
    out.loc[m_b, "recon_method"] = "AVAIL_SCALE"
    # (c/d) deferred
    out["needs_model_recon"] = cens & ~m_a & ~m_b
    pct = (m_a | m_b | out.needs_model_recon).sum() / max((out.period_state != "STRUCTURAL_ZERO").sum(), 1)
    return out, pct, pct > 0.30          # LOW_CONFIDENCE flag

def iqr_bounds(x, k=3.0):
    q1, q3 = np.nanpercentile(x, [25, 75]); iqr = q3 - q1
    return q1 - k * iqr, q3 + k * iqr

def robust_remainder(y, m):
    """Robust (median-based) STL-equivalent: trend = centered rolling median (window m+1),
    seasonal = per-season median of detrended values (centered). Default implementation, because
    statsmodels STL(robust=True) with weekly m=52 and few cycles was observed to absorb isolated spikes
    into the seasonal component (see 'Implementation note')."""
    y = pd.Series(np.asarray(y, float)); w = m + 1 if m % 2 == 0 else m
    tr = y.rolling(w, center=True, min_periods=w // 2 + 1).median().interpolate(limit_direction="both")
    d = y - tr
    prof = d.groupby(np.arange(len(y)) % m).transform("median"); prof -= prof.mean()
    return y - tr - prof

def flag_outliers(y: pd.Series, m: int, adi: float, engine="median"):
    """GS-D-04: 3xIQR on robust-decomposition remainder; intermittent/short -> non-zero sizes only."""
    y = pd.Series(np.asarray(y, float))
    if adi >= 1.32 or y.notna().sum() < 2 * m:            # intermittent, or < 2 cycles
        nz = y[y > 0]; lo, hi = iqr_bounds(nz)
        return ((y > 0) & ((y < lo) | (y > hi))).to_numpy(), None
    yi = y.interpolate(limit_direction="both")
    if engine == "stl":                                    # optional cross-check (R stl / fable STL robust)
        from statsmodels.tsa.seasonal import STL
        rem = STL(yi, period=m, robust=True).fit().resid
    else:
        rem = robust_remainder(yi, m)
    lo, hi = iqr_bounds(rem)
    return ((rem < lo) | (rem > hi)).to_numpy(), rem
```
STL needs at least 2 full cycles for a stable seasonal estimate. Between 1 and 2 cycles, the snippet falls back to the non-zero-size rule and the run is flagged `SHORT_HISTORY`.

**Implementation note (tested):** I injected a single spike (400 vs. a level of about 100) into synthetic weekly series with m = 52 and 2–6 years of history. `statsmodels` `STL(robust=True)` often missed it at default and periodic settings, because the spike leaked into the seasonal component. The median-based robust decomposition above caught it in every case tested.

The default engine is therefore `median`. Use `engine="stl"` (or R `stl(robust=TRUE)` / fable `STL(robust=TRUE)`) only as a cross-check. Both engines apply the same 3×IQR rule required by GS-D-04. With 2–3 cycles the median engine gives a few false positives, which is why flags without a known cause are kept rather than replaced (Step 7).

### 5.3 Python: ADI used for routing (consistent with `nextq-demand-pattern-segmentation`)
```python
def adi(y):  # y over train_mask periods
    nz = (y > 0).sum()
    return np.inf if nz == 0 else len(y) / nz
```

## 6. Standardized Output Format

```markdown
# Demand History Cleansing — <scope> — run <run_id>

## Executive Summary
- Fitness verdict: <FIT / FIT WITH FLAGS / NOT FIT> for forecasting at <grain>.
- <x>% of item-location-periods censored (<y>% materially, in-stock < 95%); <z> series LOW_CONFIDENCE (>30% reconstructed).
- <n> outliers detected (3×IQR on robust decomposition remainder); <k> replaced (data error / one-off), <j> kept as promo/event regressors, <u> unexplained → review.
- Missing: <m>% imputed; <s> series truncated (gap > 1 cycle or >20% missing).
- Value at risk from unresolved issues: <units> units / <currency>.

## Metadata
| Level | Grain | Time window | Calendar | Signal (demand_type) | UoM | Weighting | Lag |
|---|---|---|---|---|---|---|---|
| SKU-Location | WEEK (fiscal, week starts <day>) | <start>–<end> | 4-4-5 / 53-wk handling | POS/ORDERS/SHIPMENTS | base UoM | volume (units) | n/a (history) |

## Data-Quality KPIs
| KPI | Value | Rule | Threshold | Status |
|---|---|---|---|---|
| Item-locations / periods | N / P | GS-K-13 | n/a | n/a |
| % STRUCTURAL_ZERO | | GS-D-02 | n/a | |
| % MISSING (imputed) | | GS-D-03 | >20% per series → flag | |
| % CENSORED / % material (<95% in-stock) | | GS-M-11 | n/a | |
| % periods reconstructed (ORDERS / AVAIL_SCALE / pending C2) | | GS-M-11 | >30% per series → LOW_CONFIDENCE | |
| % TRUE_ZERO | | GS-D-02 | n/a | |
| Outliers flagged / replaced / kept | | GS-D-04 | 3×IQR | |
| Volume change raw → clean (units, %) | | GS-G-11 | reconciles exactly | ✅/❌ |

## Exceptions
| ID | Trigger (rule) | Item | Location | Periods | Units affected | Value affected | Recommended action | Owner |
|---|---|---|---|---|---|---|---|---|
| EXC-001 | LOW_CONFIDENCE (GS-M-11, >30% reconstructed) | | | | | | Run nextq-unconstrained-demand-reconstruction; review availability | Demand planner |
| EXC-002 | UNEXPLAINED outlier (GS-D-04) | | | | | | Confirm cause; tag event or mark data error | |
| EXC-003 | INSUFFICIENT_HISTORY (GS-D-03) | | | | | | Route to nextq-new-product-lifecycle-forecasting | |
| EXC-004 | CENSORING_UNASSESSABLE (no on-hand) | | | | | | Source inventory snapshots | |
| EXC-005 | EXC_UOM_MISSING / SUPERSESSION_CYCLE | | | | | | Fix master data (blocking) | |

## Recommendations
1. <ranked, with expected impact>

## Data-Quality / Lineage Log
| run_id | item | location | period | rule_id | method | before | after | timestamp |
|---|---|---|---|---|---|---|---|---|

## Reproducibility
Inputs snapshot: <id>; parameters: censor_material=0.95, recon_cap=2.0, recon_min_avail=0.5, low_conf=0.30, missing_max=0.20, iqr_k=3.0, decomposition=median (stl cross-check), m=<52|12|7>; code: §5 snippets; reconciliation check: PASS.
```

## 7. Edge Cases (must handle)
- **All-zero series in the train mask:** state `NO_DEMAND`. Skip outlier detection and route to the inventory-health / obsolescence review.
- **Promo period that is also censored:** reconstruct first, then tag the promo outlier as `PROMO`. Never replace the promo spike with baseline.
- **Shipments-only data with order cuts:** add `order_cut_qty` back (method a, partial) when it is reported. Otherwise mark the series `CONSTRAINED_SIGNAL`.
- **Mixed grains in input:** stop. A run uses exactly one grain. Convert first with `nextq-time-bucket-calendar-alignment`.
- **Week 53:** keep it as a period. Never drop it silently, and flag it for downstream seasonal alignment (GS-D-10).
- **Store closures recorded as zeros:** if `store_open_flag = false` or the store had zero total sales, treat as `STRUCTURAL_ZERO`, not `TRUE_ZERO`.
