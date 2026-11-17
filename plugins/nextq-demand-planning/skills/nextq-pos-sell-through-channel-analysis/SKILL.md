---
name: nextq-pos-sell-through-channel-analysis
description: >-
  [Next Quarter Supply Chain Suite] Compares sell-in (shipments) with sell-out (POS / consumer sales) to measure
  channel inventory, sell-through, forward and backward weeks of supply, channel build or burn against retailer
  targets, transit lag and bullwhip (variance amplification of orders over consumer demand), then derives
  channel-aware shipment forecasts from the sell-out forecast. Reconciles computed vs retailer-reported inventory,
  aligns retailer and fiscal weeks, extrapolates partial POS coverage and flags forward buying. Load when the user
  asks to compare shipments with POS, analyse sell-through or channel inventory, check retailer weeks of supply,
  detect channel stuffing or destocking, measure the bullwhip effect, or forecast shipments from consumer demand.
  Inputs: shipments, POS, retailer inventory, targets, calendar. Triggers: "sell-in vs sell-out", "sell-through",
  "channel inventory", "weeks of supply", "bullwhip".
metadata:
  author: "Next Quarter"
  organization: "Next Quarter (nextq)"
  suite: "Next Quarter Supply Chain Intelligence"
  version: "1.0.0"
---

# POS Sell-Through & Channel Analysis
*Part of the **Next Quarter (nextq)** Supply Chain Skill Suite — Governed by SUPPLY_CHAIN_GOLD_STANDARD.md.*

## 1. Role & Objective
**Persona:** Customer / Channel Demand Analyst (CPFR, retail collaboration).
**Decision supported:** Is consumer demand (sell-out) moving the way our shipments (sell-in) suggest? How much inventory sits in the channel, and is it building or burning? Are orders amplifying demand variability? What should we expect to ship given consumer demand and retailer inventory targets?

**Objective:**
- A reconciled customer × item × week view of sell-in, sell-out and channel inventory.
- Forward and backward weeks of supply (WOS) against target, and a measured transit lag.
- A bullwhip ratio with a confidence interval.
- A channel-aware sell-in forecast for `nextq-consensus-demand-review` and `nextq-supply-demand-balancing`.

## 2. Gold Standard Rules Enforced
| Rule | Inlined standard |
|---|---|
| **GS-D-11** Signal hierarchy | Prefer the signal closest to the consumer: **POS / sell-out → customer orders (requested date) → shipments / sell-in**. Shipments are a **constrained** proxy and are never used as demand when orders or POS exist. Track the sell-in vs. sell-out gap to detect channel build/burn and bullwhip. Variance amplification $=\operatorname{Var}(\text{orders})/\operatorname{Var}(\text{demand})$ (Lee, Padmanabhan & Whang 1997). |
| **GS-D-09** Time buckets | One fiscal calendar table. Retailer weeks often end on a different day than our fiscal week: re-bucket **daily** POS to our week-ending day, or apportion retailer weeks by day using `nextq-time-bucket-calendar-alignment`. Never pair weeks by label. |
| **GS-K-11** Days / weeks of supply | **Forward (gold standard):** weeks until cumulative **forecast** sell-out exhausts channel inventory. **Backward (labeled):** inventory / trailing average sell-out (default 4 weeks), which hides seasonality. Always state forward vs. backward and whether in-transit is included. |
| **GS-M-11** Censoring | Retail weeks with store in-stock < 95% truncate sell-out. Use reconstructed sell-out (`nextq-unconstrained-demand-reconstruction`) for WOS and bullwhip, or flag `CENSORED_SELL_OUT`. |
| **GS-G-03** Exceptions | Each exception has cause, impact (units, value), action and owner, ranked by value at risk = \|units\| × unit margin (cost for excess) × ABC weight (A = 3, B = 2, C = 1 [RoT]). |
| **GS-G-05** Plans | The sell-in forecast is a **derived** shipment plan from unconstrained sell-out demand. Supply constraints are applied in S&OP, not here. |
| **GS-G-10 / GS-G-11** | Standard output; the inventory balance identity is verified in code. |

**Channel parameters [RoT, configurable]:**
- Inventory reconciliation tolerance: 5% of reported.
- WOS tolerance band: ±1 week around the retailer's target.
- Bullwhip: computed on first differences, with sell-out shifted by the measured lag $\tau$ and a moving-block bootstrap CI (block 4 weeks, 2,000 resamples). Flag `BULLWHIP` when the ratio > 1.5 **and** the CI lower bound > 1.
- Minimum 26 weeks for bullwhip and lag estimation.
- Inventory correction in the sell-in forecast is spread over 4 weeks.
- POS coverage below 60% of shipments → `LOW_COVERAGE`.

## 3. Required Inputs & Data Schema
| Table | Columns | Notes |
|---|---|---|
| `sell_in` | `customer_id, ship_to, item_id, date, shipped_qty, ordered_qty, requested_date, uom` | from ERP; convert to base UoM |
| `sell_out` | `customer_id, store_id, item_id, date or retailer_week, pos_units, in_stock_pct, price` | retailer POS feed |
| `channel_inventory` | `customer_id, location (DC/store), item_id, snapshot_date, on_hand, in_transit` | optional, retailer-reported |
| `returns` | `customer_id, item_id, date, return_qty` | optional |
| `targets` | `customer_id, item_id, target_wos, min_wos, max_wos` | from trading-partner agreement |
| `coverage` | `customer_id, share_of_shipments_with_pos` | for extrapolation |
| `fiscal_calendar` | `date, fiscal_week, week_ending_day` | GS-D-09 |
| `sell_out_forecast` | `customer_id, item_id, fiscal_week, F_sell_out` | from the forecasting skills |

**Grain:** customer × item × fiscal week (store-level detail is optional). **Minimum:** 26 aligned weeks for lag and bullwhip; 13 weeks for channel balance.

## 4. Execution Workflow
1. **Align buckets.**
   - Convert all quantities to base UoM.
   - Re-bucket daily POS and shipments to the fiscal week. If the retailer only provides weekly POS ending on a different day, apportion it by day (DOW-weighted when known).
   - Assert totals are conserved.
2. **Coverage.**
   - If POS covers less than 100% of shipped volume, scale sell-out by 1/coverage for the uncovered share, flagged `EXTRAPOLATED`.
   - Coverage below 60% → `LOW_COVERAGE`. Bullwhip is not computed.
3. **Channel balance.** $\mathit{I}_t=I_{t-1}+\text{SellIn}_t-\text{SellOut}_t-\text{Returns}_t$, from the first reported snapshot $\mathit{I}_0$.
   - Compare with reported inventory when available. A gap above 5% → `RECON_GAP` (shrink, unreported transfers, or a feed issue).
   - Negative computed inventory → `NEGATIVE_COMPUTED_INV` (the data are incomplete). Reported inventory is then authoritative.
4. **Sell-through** $=\text{SellOut}/(I_{\text{begin}}+\text{SellIn})$ per week.
5. **WOS.**
   - Forward WOS = fractional weeks until cumulative $\mathit{F}_{\text{sell out}}$ ≥ inventory. If the inventory outlasts the forecast horizon, report "> H".
   - Backward WOS = inventory / mean of the last 4 weeks of sell-out, labeled.
   - Status from forward WOS vs. target ± 1 week: `CHANNEL_BUILD`, `ON_TARGET` or `CHANNEL_BURN`.
6. **Transit lag** $\tau$ = the lag in 0–4 weeks that maximizes the correlation of first-differenced sell-in with first-differenced sell-out shifted by $\tau$. Report the correlation; below 0.2, use the contractual lead time and flag `LAG_ASSUMED` [RoT].
7. **Bullwhip.** $\mathit{BW}=\operatorname{Var}(\Delta\text{SellIn}_t)/\operatorname{Var}(\Delta\text{SellOut}_{t+\tau})$ with the block-bootstrap CI. $\mathit{BW} > 1$ means amplification. Apply the flag rule from Section 2.
8. **Forward buy / pipeline events.**
   - A sell-in spike (> 3×IQR on the sell-in − sell-out gap remainder) with flat sell-out ahead of a list-price increase or deal → `FORWARD_BUY`.
   - Expect a later burn. Link to the promo skill's `SELL_IN_LIFT`.
9. **Channel-aware sell-in forecast.**
   - $\text{SI}_t=F^{\text{SO}}_{t+\tau}+\big(\text{target\_wos}\cdot\overline{F^{\text{SO}}}_{t+\tau..t+\tau+3}-\hat I_t\big)/4$, floored at 0, then project $\hat I_{t+1}=\hat I_t+\text{SI}_t-F^{\text{SO}}_t$.
   - Submit it to consensus as reason CUSTOMER_INTEL with this report as evidence.
10. **Exceptions and output** (Section 6). Value at risk for build or burn: excess units (WOS − max) × weekly sell-out at cost, or shortage units at margin, × ABC weight.

## 5. Verification Snippets
```python
import numpy as np, pandas as pd

def channel_inventory(i0, sell_in, sell_out, returns=None, reported=None, tol=0.05):
    """GS-D-11 channel balance: I_t = I_{t-1} + SellIn_t - SellOut_t - Returns_t (all base UoM, same aligned bucket).
    If the retailer reports inventory, residual = reported - computed; |residual| > 5% of reported -> RECON_GAP [RoT]."""
    si, so = np.asarray(sell_in, float), np.asarray(sell_out, float)
    rt = np.zeros_like(si) if returns is None else np.asarray(returns, float)
    inv = i0 + np.cumsum(si - so - rt)
    flags = np.where(inv < 0, "NEGATIVE_COMPUTED_INV", "")
    if reported is not None:
        rep = np.asarray(reported, float)
        gap = np.abs(rep - inv) > tol * np.maximum(np.abs(rep), 1e-9)
        flags = np.where(gap, np.char.add(flags.astype(str), "|RECON_GAP"), flags)
    return inv, flags

def sell_through_rate(begin_inv, sell_in, sell_out):
    """Sell-through % = sell-out / (beginning inventory + sell-in) for the period."""
    den = np.asarray(begin_inv, float) + np.asarray(sell_in, float)
    return np.where(den > 0, np.asarray(sell_out, float) / den, np.nan)

def wos_forward(inv, fcst_sell_out):
    """GS-K-11 forward weeks of supply: weeks until cumulative sell-out forecast exhausts inventory (fractional)."""
    cum, prev = 0.0, 0.0
    for w, f in enumerate(np.asarray(fcst_sell_out, float)):
        if cum + f >= inv:
            return w + (inv - cum) / f if f > 0 else float(w)
        cum += f
    return np.inf                                     # beyond forecast horizon: report as '> H'

def wos_backward(inv, trailing_sell_out, n=4):
    """Backward-looking WOS (labelled): inventory / mean of last n weeks of sell-out. Hides seasonality."""
    m = np.mean(np.asarray(trailing_sell_out, float)[-n:])
    return inv / m if m > 0 else np.inf

def best_lag(sell_in, sell_out, max_lag=4):
    """Transit/ordering lag tau (weeks) maximizing corr(diff(sell_in)_t, diff(sell_out)_{t+tau})."""
    a, b = np.diff(np.asarray(sell_in, float)), np.diff(np.asarray(sell_out, float))
    best = (0, -np.inf)
    for tau in range(max_lag + 1):
        x, y = (a, b) if tau == 0 else (a[:-tau], b[tau:])
        if len(x) > 3 and np.std(x) > 0 and np.std(y) > 0:
            c = np.corrcoef(x, y)[0, 1]
            if c > best[1] + 1e-12:
                best = (tau, c)
    return best

def bullwhip(sell_in, sell_out, tau=0, block=4, n_boot=2000, seed=42):
    """Variance amplification Var(sell-in)/Var(sell-out) (Lee, Padmanabhan & Whang 1997), on first differences
    (removes level/trend), sell-out shifted by tau. CI by moving-block bootstrap (block 4 wks) [RoT]."""
    si, so = np.asarray(sell_in, float), np.asarray(sell_out, float)
    if tau:
        si, so = si[:-tau], so[tau:]
    a, b = np.diff(si), np.diff(so)
    ratio = a.var(ddof=1) / b.var(ddof=1)
    rng = np.random.default_rng(seed); n = len(a); k = int(np.ceil(n / block)); bs = []
    for _ in range(n_boot):
        st = rng.integers(0, n - block + 1, k)
        idx = np.concatenate([np.arange(s, s + block) for s in st])[:n]
        bs.append(a[idx].var(ddof=1) / b[idx].var(ddof=1))
    return ratio, tuple(np.percentile(bs, [2.5, 97.5]))

def channel_status(wos, target, tol=1.0):
    """CHANNEL_BUILD if WOS > target + tol; CHANNEL_BURN if < target - tol (weeks) [RoT tol = 1 week]."""
    return "CHANNEL_BUILD" if wos > target + tol else ("CHANNEL_BURN" if wos < target - tol else "ON_TARGET")

def sell_in_forecast(fcst_sell_out, inv0, target_wos, tau=0, correction_weeks=4):
    """Channel-aware shipment forecast: sell-in_t = sell-out_fcst_{t+tau} + (target inv - projected inv)/correction_weeks,
    target inv = target_wos * mean next-4-week sell-out forecast. Floors at 0 (no negative shipments)."""
    f = np.asarray(fcst_sell_out, float); H = len(f) - tau; out = []; inv = inv0
    for t in range(H):
        tgt = target_wos * f[t + tau: t + tau + 4].mean()
        si = max(f[t + tau] + (tgt - inv) / correction_weeks, 0.0)
        inv = inv + si - f[t]
        out.append(si)
    return np.array(out)
```
**Test vector** (verified):
- **Balance:** I₀ = 100, sell-in [50, 80, 0, 60], sell-out [40, 45, 50, 55] → I = [110, 145, 95, 100]. Reported [110, 146, 90, 95] gives `RECON_GAP` in weeks 3 and 4 (5.3–5.6% > 5%).
- **Sell-through:** [0.2667, 0.2368].
- **WOS:**
  - Forward: 95 units at 50/week → **1.9** weeks.
  - Forward: 500 units against a 2-week horizon → ∞, reported as "> H".
  - Backward: 95 / mean(40, 45, 50, 55) → **2.0** (labeled backward).
- **Lag and bullwhip:** synthetic sell-in leading sell-out by 2 weeks, with extra noise (seed 5).
  - Lag τ = **2** (r = 0.44).
  - BW = **5.66** (95% CI 2.85–9.96) → `BULLWHIP`.
  - Identical series → BW = 1.0.
- **Status:** WOS 6.5 / 4.5 / 2.0 against target 4 → BUILD / ON_TARGET / BURN.
- **Sell-in forecast:** sell-out 50/week, I₀ = 300, target 4 WOS (200) → 25.0, 31.25, 35.94, … converging to 50 as the 100-unit excess burns off.

```sql
-- Weekly channel balance per customer-item on the fiscal calendar (POS re-bucketed from daily)
WITH si AS (SELECT customer_id, item_id, c.fiscal_week, SUM(shipped_qty_base) AS sell_in
            FROM sell_in s JOIN fiscal_calendar c ON c.date = s.date GROUP BY 1,2,3),
     so AS (SELECT customer_id, item_id, c.fiscal_week, SUM(pos_units_base) AS sell_out,
                   MIN(in_stock_pct) AS min_in_stock
            FROM sell_out p JOIN fiscal_calendar c ON c.date = p.date GROUP BY 1,2,3)
SELECT COALESCE(si.customer_id, so.customer_id) AS customer_id, COALESCE(si.item_id, so.item_id) AS item_id,
       COALESCE(si.fiscal_week, so.fiscal_week) AS fiscal_week,
       COALESCE(sell_in,0) AS sell_in, COALESCE(sell_out,0) AS sell_out,
       SUM(COALESCE(sell_in,0) - COALESCE(sell_out,0)) OVER (PARTITION BY COALESCE(si.customer_id, so.customer_id), COALESCE(si.item_id, so.item_id)
           ORDER BY COALESCE(si.fiscal_week, so.fiscal_week)) AS cum_gap,          -- add I0 for the inventory level
       min_in_stock < 0.95 AS censored_sell_out
FROM si FULL JOIN so ON so.customer_id = si.customer_id AND so.item_id = si.item_id AND so.fiscal_week = si.fiscal_week;
```

## 6. Standardized Output Format
```markdown
# Sell-In vs Sell-Out & Channel Health — <customer/scope> — run CHN_<run_id>

## Executive Summary
- Sell-out <x>% vs sell-in <y>% (13-wk YoY like-for-like); channel inventory <I> units = <wos_f> forward WOS vs target <t> → <status>.
- Bullwhip <BW> (95% CI <lo>–<hi>) at lag τ=<τ> wks; <n_fb> FORWARD_BUY events.
- Derived sell-in next 13 wks <units> (<Δ>% vs current shipment forecast).
- Confidence: POS coverage <c>%, <r> RECON_GAP weeks, <z>% censored sell-out weeks.

## Metadata (mandatory)
| Level | Grain | Time window | Signal | Calendar alignment | WOS basis | Lag τ | Weighting | Coverage |
|---|---|---|---|---|---|---|---|---|
| customer × item | fiscal WEEK | <start–end> | POS sell-out vs shipments | retailer wk → fiscal wk (<method>) | forward (forecast), in-transit <incl/excl> | <τ> | volume | <c>% |

## KPI Table
| customer | item | sell-in 13w | sell-out 13w | gap | channel inv | sell-through % | WOS fwd | WOS bwd (labelled) | target | status | BW (CI) |
|---|---|---|---|---|---|---|---|---|---|---|---|

## Derived Sell-In Forecast
| customer | item | week | sell-out fcst | projected inv | target inv | sell-in fcst | vs current shipment fcst |
|---|---|---|---|---|---|---|---|

## Exceptions
| ID | Trigger (rule) | customer / item | metric | value | units | VaR | action | owner |
|---|---|---|---|---|---|---|---|---|
| | CHANNEL_BUILD (GS-K-11) | | WOS fwd | | | | slow shipments / sell-out support | KAM |
| | CHANNEL_BURN (GS-K-11) | | WOS fwd | | | | expedite / allocation check | Supply |
| | BULLWHIP (GS-D-11) | | BW | | | | order smoothing / VMI / CPFR | Planning |
| | FORWARD_BUY (GS-D-11) | | spike | | | | plan the burn; exclude from baseline | Demand planner |
| | RECON_GAP / NEGATIVE_COMPUTED_INV | | % | | | | fix feed / shrink review | Data owner |
| | LOW_COVERAGE (<60%) | | % | | | | request POS / treat as indicative | KAM |

## Recommendations
1. …

## Data-Quality Log
| customer | issue (week-end mismatch, UoM, missing POS weeks, censored sell-out, extrapolated coverage) | treatment |
|---|---|---|

## Reproducibility
Calendar version, τ search range, bootstrap seed/blocks, tolerances, targets version, code §5.
```

## 7. Edge Cases
- **Retailer DC vs. store inventory:** WOS is computed for both, and the channel total = DC + stores + in transit (state which). Store-only WOS hides DC build.
- **New store openings / closures:** use same-store (comparable) sell-out for trend and bullwhip. Pipeline fill for new stores is a legitimate one-off build (`PIPELINE_FILL`), excluded from bullwhip.
- **Returns spikes** (post-season returns): include them in the balance, but exclude them from sell-out demand.
- **Seasonal items:** use forward WOS only; backward WOS before a peak understates cover.
- **Missing POS weeks:** do not zero-fill. Interpolate for balance continuity only, flagged `IMPUTED_POS`, and exclude those weeks from bullwhip.
- **Direct-to-consumer / marketplace channels** with no intermediary inventory: sell-in equals sell-out, so channel analysis does not apply.
- **Promotions:** retailer forward buy ahead of a deal appears as BUILD followed by BURN. Evaluate WOS against the deal calendar before escalating.
