---
name: nextq-promotion-lift-cannibalization-planning
description: >-
  [Next Quarter Supply Chain Suite] Measures and plans promotional demand: baseline vs gross lift, pre-promo and
  post-promo dips (pantry loading, forward buying), cannibalization of substitutes, halo on complements, net
  incremental volume and margin, promo ROI, and promo-aware forecasts built from baseline x lift by mechanic (depth,
  display, feature). Load when the user asks to evaluate a past promotion (post-event analysis), forecast an upcoming
  promotion, estimate promo lift or uplift, measure cannibalization or halo, decide whether a promo was incremental or
  profitable, or build a promo calendar forecast. Inputs: cleaned demand (POS preferred) with promo calendar and
  mechanics, prices, costs and trade spend, substitute and complement groups (need states), optional control stores,
  at SKU-Location-Week/Day. Triggers: "promo lift", "uplift", "cannibalization", "halo effect", "post-promo dip",
  "promotion ROI", "was the promotion incremental", "forecast the promotion".
metadata:
  author: "Next Quarter"
  organization: "Next Quarter (nextq)"
  suite: "Next Quarter Supply Chain Intelligence"
  version: "1.0.0"
---

# Promotion Lift & Cannibalization Planning
*Part of the **Next Quarter (nextq)** Supply Chain Skill Suite — Governed by SUPPLY_CHAIN_GOLD_STANDARD.md.*

## 1. Role & Objective
**Persona:** Revenue Growth Management (RGM) / Promotion Analytics Lead.
**Decision supported:**
- Did a promotion create **net incremental** category volume and margin, after dips, cannibalization, and halo?
- What should we forecast for planned promotions?
- Which mechanics are worth repeating?

**Objective:** A deterministic decomposition of every promo event into baseline, gross lift, pre-dip, post-dip, cannibalization, halo, and net incremental (units and margin), plus a lift library by mechanic. It also produces promo forecasts (baseline × lift) that feed `nextq-consensus-demand-review` as **PROMO** inputs with modeled evidence.

## 2. Gold Standard Rules Enforced
| Rule | Inlined standard |
|---|---|
| **GS-D-05** Promo lift decomposition [STD practice] | **Baseline** $B_t$ = demand without the promotion. Estimate it by (a) causal model counterfactual (promo off, regular price) from `nextq-ml-causal-demand-forecasting` (preferred), or (b) a statistical forecast through the promo window with promo periods **masked** (excluded from fitting). **Gross lift** $L_t=A_t-B_t$; **Lift %** $=(A_t-B_t)/B_t$. **Net incremental volume:** $\text{NetInc}=\sum_{t\in\text{promo}}(A_t-B_t)-\sum_{t\in\text{post}}(B_t-A_t)-\sum_{t\in\text{pre}}(B_t-A_t)-\text{Cannib}+\text{Halo}$. Default windows [RoT]: pre = 1 period, post = 2 periods (weekly). Extend post to 4 for stockpileable categories. Dips are counted only when positive (no negative dips). |
| **GS-D-06** Cannibalization & halo | **Cannibalization** = loss in the baselines of substitutable items $j$ (same need-state) during the promo of $i$: $\sum_j\sum_{t\in\text{promo}}(B_{j,t}-A_{j,t})^+$. **Halo** = gain in complements $k$: $\sum_k\sum_t(A_{k,t}-B_{k,t})^+$. Estimate them via cross-promo / cross-price regression terms or **test-vs-control** difference-in-differences. Report the **category-level net**. A promo with positive SKU lift but **≤ 0 category net** is flagged `NON_INCREMENTAL_CATEGORY`. Substitutes that were themselves on promo (overlap) are excluded from the cannibalization pool. |
| **GS-M-11** Censoring | Stockouts during the promo truncate lift. Promo periods with in-stock < 95% are **materially censored**: reconstruct demand (orders, or availability scaling $\hat D=S/a$ capped at ×2 for $a ≥ 0.5$), and flag the event `CENSORED_PROMO` (lift is a lower bound). Never fit a lift library on censored events without the flag. |
| **GS-D-11** Signal | Use **POS/sell-out** for consumer lift. Shipments include retailer forward-buy, so a shipment-based lift must be labeled `SELL_IN_LIFT` and the forward-buy dip measured over a longer post window (default 4–8 weeks) [RoT]. |
| **GS-M-09 / GS-M-16** | Lift models (regression or ML) are validated by rolling-origin over events. Features must be known in advance (planned depth, display, feature). Look-ahead leakage is prohibited. |
| **GS-K-07 / GS-K-01 / GS-K-05** | Promo forecast accuracy is scored on promo periods separately: WAPE $\sum\lvert A-F\rvert/\sum A$, and Bias% $\sum(F-A)/\sum A$ (+ = over). The **FVA of the modeled lift** vs. a naïve lift (the item's last comparable event lift, or the category-average lift) must be ≥ 0, or the library is not used for that cell. |
| **GS-G-06** Financials | Per event: incremental gross margin $=\text{NetInc}\times m_{\text{reg}}-\text{(promo units)}\times(\text{P}_{\text{reg}}-\text{P}_{\text{promo}})_{\text{funded}}-\text{fixed fees}$, where $m_{\text{reg}}$ = regular unit margin. **Promo ROI** $=\dfrac{\text{incremental gross margin}}{\text{total trade spend}}$. Cannibalized units are valued at the **substitute's** margin; halo at the complement's margin. Value conversions use net prices consistently (volume-to-value per GS-G-06). |
| **GS-G-01** Handoff to consensus | Promo forecasts enter `nextq-consensus-demand-review` as PROMO overrides with `evidence_link` = this report. UP > 20% vs. base requires this evidence. |
| **GS-G-03 / 10 / 11** | Exceptions; standard output; reproducible decomposition (sum check: $A_{\text{promo}}=B+L$ exactly). |

**Lift-library pooling rule** [RoT]:
- **Item-mechanic** level if ≥ 3 clean (uncensored) events.
- Else **subcategory-mechanic** (≥ 5 events).
- Else **category-mechanic**, flagged `POOLED_LIFT`.
- Mechanic key = depth band {< 15%, 15–25%, 25–35%, > 35%} × display × feature × (multi-buy flag).

## 3. Required Inputs & Data Schema
**`demand_clean`** (from `nextq-demand-history-cleansing`; POS preferred): `item_id, location_id, period_start, demand_clean, period_state, in_stock_pct, demand_type`.

**`promo_calendar`**: one row per event × item × location.

| Column | Type | Unit | Req. |
|---|---|---|---|
| `promo_id` | string | n/a | Y |
| `item_id`, `location_id` (or `location_group`) | string | n/a | Y |
| `start_period`, `end_period` | date | fiscal bucket | Y |
| `regular_price`, `promo_price` | decimal | currency / base UoM | Y |
| `depth` | decimal | (reg − promo)/reg | derived |
| `display_flag`, `feature_flag`, `multibuy_flag` | bool | n/a | Y |
| `funding_share` | decimal [0,1] | share of discount funded by us | Y |
| `fixed_fees` | decimal | currency | N |
| `status` | enum | PLANNED/RUNNING/COMPLETED/CANCELLED | Y |

**`item_groups`**: `item_id, need_state_id, role ∈ {SUBSTITUTE_POOL, COMPLEMENT_OF:<need_state>}` (defines the cannibalization and halo sets).

**`costs`**: `item_id, location_id, unit_cost, regular_net_price` → regular unit margin $m_{\text{reg}}$.

**`baseline_forecasts`**: `item_id, location_id, period, baseline_qty, method ∈ {CAUSAL_CF, MASKED_STAT}` (from `nextq-ml-causal-demand-forecasting` / `nextq-statistical-baseline-forecasting`).

**`control_map`** (optional): `test_location_id, control_location_id` (similar stores without the promo) for DiD.

**Granularity:** SKU-Location-Week (default) or Day. Event-level results roll up to promo_id × item.

**Minimum:**
- Baseline history ≥ 1 cycle (52 weeks) with promo periods masked.
- ≥ 8 pre-period weeks for DiD control matching [RoT].
- Library pooling thresholds as above.

## 4. Execution Workflow
1. **Event windows.** For each event, set promo periods $[s,e]$, pre = $[s-1]$ (or $s-k_{\text{pre}}$), and post = $[e+1, e+k_{\text{post}}]$. Mark overlapping events on the same item (merged windows, flagged `OVERLAP`) and windows hitting other promos of the same item (dip windows truncated).
2. **Censoring check (GS-M-11):** in-stock < 95% in any promo period → reconstruct and flag `CENSORED_PROMO`.
3. **Baseline:** use `CAUSAL_CF` if available, else `MASKED_STAT`. The baseline must exclude promo periods **and** dip windows from fitting (no contamination). Log the method.
4. **Own-item decomposition:**
   - $L=\sum_{\text{promo}}(A-B)$
   - $\text{PreDip}=\sum_{\text{pre}}(B-A)^+$
   - $\text{PostDip}=\sum_{\text{post}}(B-A)^+$
   - Verify $\sum_{\text{promo}}A=\sum B+L$ exactly.
5. **Cannibalization and halo:**
   - Substitute pool $J$ = same need-state, not on promo in the window, available (in-stock ≥ 95%).
   - $\text{Cannib}=\sum_{j\in J}\sum_{\text{promo}}(B_j-A_j)^+$
   - Complements $K$: $\text{Halo}=\sum_{k}\sum_{\text{promo}}(A_k-B_k)^+$
   - If `control_map` exists, use **DiD** instead: $\widehat{\text{effect}}=(\bar A^{\text{test}}_{\text{promo}}-\bar A^{\text{test}}_{\text{pre8}})-(\bar A^{\text{ctrl}}_{\text{promo}}-\bar A^{\text{ctrl}}_{\text{pre8}})$ per period × number of promo periods, for the own item, the pool, and complements. Report both methods when available.
6. **Net incremental:** $\text{NetInc}=L-\text{PreDip}-\text{PostDip}-\text{Cannib}+\text{Halo}$. Category net = NetInc in category units (UoM-harmonized).
7. **Financials:**
   - Incremental margin $=L\cdot m_i-\text{PreDip}\cdot m_i-\text{PostDip}\cdot m_i-\sum_j\text{Cannib}_j m_j+\sum_k\text{Halo}_k m_k-\text{funded discount}-\text{fees}$, where funded discount $=\sum_{\text{promo}}A\cdot(P_{\text{reg}}-P_{\text{promo}})\cdot\text{funding\_share}$.
   - ROI = incremental margin / (funded discount + fees).
   - Flag `NEGATIVE_ROI` and `NON_INCREMENTAL_CATEGORY`.
8. **Lift library:** per mechanic key and pooling level, compute the median and IQR of lift multiplier $\lambda=\sum_{\text{promo}}A/\sum_{\text{promo}}B$, post-dip ratio, and cannibalization ratio $=\text{Cannib}/L$. Exclude `CENSORED_PROMO` and `OVERLAP` events from the median (report them separately).
9. **Validate the library (rolling-origin over events):** predict each event's $\lambda$ from events before it (same pooling rule), then compute WAPE and Bias% of promo-period forecasts vs. the naïve lift (the last comparable event's λ, or the category median). FVA < 0 → do not use that cell; fall back to the next pooling level.
10. **Forecast planned promos:**
    - $F_{\text{promo},t}=B_t\cdot\lambda_{\text{mech}}$
    - $F_{\text{post},t}=B_t\cdot(1-\text{postdip ratio}_{t})$
    - Substitute adjustments $F_{j,t}=B_{j,t}-\text{cannib ratio}\cdot L_t\cdot \text{share}_j$, where $\text{share}_j$ = the substitute's share of pool baseline.
    - Output P10/P90 from the library IQR (scaled).
    - Produce consensus submissions (PROMO reason, evidence link).
11. **Output** (Section 6).

## 5. Verification Snippets
```python
import numpy as np, pandas as pd

def decompose_event(A, B, promo, pre, post):
    """A, B: pd.Series indexed by period for the promoted item; promo/pre/post: lists of periods."""
    L = (A[promo] - B[promo]).sum()
    pre_dip = (B[pre] - A[pre]).clip(lower=0).sum() if len(pre) else 0.0
    post_dip = (B[post] - A[post]).clip(lower=0).sum() if len(post) else 0.0
    assert abs(A[promo].sum() - (B[promo].sum() + L)) < 1e-9          # sum check (GS-G-11)
    lift_mult = A[promo].sum() / B[promo].sum() if B[promo].sum() > 0 else np.nan
    return dict(gross_lift=L, pre_dip=pre_dip, post_dip=post_dip, lift_mult=lift_mult,
                lift_pct=100 * (lift_mult - 1) if np.isfinite(lift_mult) else np.nan)

def cross_effects(A_df, B_df, promo, subs, comps):
    """A_df/B_df: DataFrames (periods x items). Cannibalization on substitutes, halo on complements (positive parts only)."""
    cannib = (B_df.loc[promo, subs] - A_df.loc[promo, subs]).clip(lower=0).sum().sum() if subs else 0.0
    halo = (A_df.loc[promo, comps] - B_df.loc[promo, comps]).clip(lower=0).sum().sum() if comps else 0.0
    return cannib, halo

def net_incremental(ev, cannib, halo):
    return ev["gross_lift"] - ev["pre_dip"] - ev["post_dip"] - cannib + halo

def did(test_pre, test_promo, ctrl_pre, ctrl_promo, n_promo_periods):
    """Difference-in-differences effect in units over the promo window."""
    return ((np.mean(test_promo) - np.mean(test_pre)) - (np.mean(ctrl_promo) - np.mean(ctrl_pre))) * n_promo_periods

def promo_financials(ev, cannib_by_item_margin, halo_by_item_margin, m_own, A_promo_units,
                     p_reg, p_promo, funding_share, fees=0.0):
    funded = A_promo_units * (p_reg - p_promo) * funding_share
    inc_margin = (ev["gross_lift"] - ev["pre_dip"] - ev["post_dip"]) * m_own \
                 - sum(u * m for u, m in cannib_by_item_margin) + sum(u * m for u, m in halo_by_item_margin) \
                 - funded - fees
    spend = funded + fees
    return dict(incremental_margin=inc_margin, trade_spend=spend, roi=inc_margin / spend if spend > 0 else np.nan)

def pooling_level(n_item_mech, n_subcat_mech):
    return "ITEM" if n_item_mech >= 3 else ("SUBCATEGORY" if n_subcat_mech >= 5 else "CATEGORY_POOLED_LIFT")
```
**Test vector** (executed):
- Weekly, B = 100 every week; A = pre 95, promo 180 / 170, post 80 / 90.
- Own item: gross lift = 150, pre-dip = 5, post-dip = 30, λ = 1.75.
- Substitutes lose 40 units and complements gain 10, so NetInc = 150 − 5 − 30 − 40 + 10 = **85**.
- With m_own = 2.0, substitute margin 1.5, complement margin 1.0, P_reg 5, P_promo 4, funding 50%, promo units 350:
  - incremental margin = 115×2 − 40×1.5 + 10×1 − 175 = **5.0**
  - ROI = 5/175 = **0.029**

```sql
-- Own-item decomposition per event (weekly), baseline from masked/causal model
SELECT e.promo_id, e.item_id, e.location_id,
       SUM(CASE WHEN w.win='PROMO' THEN d.demand_clean - b.baseline_qty END)                AS gross_lift,
       SUM(CASE WHEN w.win='PRE'   THEN GREATEST(b.baseline_qty - d.demand_clean,0) END)    AS pre_dip,
       SUM(CASE WHEN w.win='POST'  THEN GREATEST(b.baseline_qty - d.demand_clean,0) END)    AS post_dip,
       SUM(CASE WHEN w.win='PROMO' THEN d.demand_clean END)
         / NULLIF(SUM(CASE WHEN w.win='PROMO' THEN b.baseline_qty END),0)                    AS lift_mult,
       MIN(CASE WHEN w.win='PROMO' THEN d.in_stock_pct END) < 0.95                           AS censored_promo
FROM promo_events e
JOIN promo_windows w ON w.promo_id = e.promo_id
JOIN demand_clean d  ON d.item_id=e.item_id AND d.location_id=e.location_id AND d.period_start=w.period
JOIN baseline_forecasts b ON b.item_id=e.item_id AND b.location_id=e.location_id AND b.period=w.period
GROUP BY 1,2,3;
```

## 6. Standardized Output Format
```markdown
# Promotion Lift & Cannibalization — <scope> — run PROMO_<run_id>

## Executive Summary
- <N> events analysed (<c>% CENSORED_PROMO, <o>% OVERLAP); median lift ×<λ> (<p>%); median post-dip <d>% of lift; cannibalization <k>% of lift.
- Net incremental: <u> units (<v>% of gross lift); <n_neg> events NON_INCREMENTAL_CATEGORY; <r_neg> NEGATIVE_ROI.
- Best mechanics: <depth band × display × feature> (ROI <x>); worst: <…>.
- Planned promos (<M>): forecast <units> vs base <units>; submitted to consensus as PROMO with evidence.

## Metadata (mandatory)
| Level | Grain | Signal | Time window | Baseline method | Windows (pre/post) | Weighting | Lag (forecasts) | Price/margin basis |
|---|---|---|---|---|---|---|---|---|
| SKU-Location (event roll-up promo_id × item) | WEEK | POS (sell-out) | <start–end> | CAUSAL_CF / MASKED_STAT | 1 / 2 weeks | volume; value at net price | <ℓ> | regular net margin |

## Event Decomposition
| promo_id | item | loc | mechanic | depth | baseline | actual | gross lift | lift ×/% | pre-dip | post-dip | cannib | halo | net incremental | category net | inc. margin | trade spend | ROI | flags |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|

## Lift Library (clean events only)
| mechanic key | pooling level | N events | median λ | IQR λ | post-dip ratio | cannib ratio | FVA vs naïve lift (WAPE pts) | usable? |
|---|---|---|---|---|---|---|---|---|

## Promo Forecast Accuracy (promo periods, lag ℓ)
| Method | WAPE % | Bias % (+=over) | FVA vs naïve lift | N events |
|---|---|---|---|---|

## Planned Promotion Forecasts (to consensus)
| promo_id | item | loc | period | baseline | λ | forecast | P10 | P90 | post-dip adj | substitute adj (total) | evidence |
|---|---|---|---|---|---|---|---|---|---|---|---|

## Exceptions
| ID | Trigger (rule) | promo_id / item | metric | value | action | owner |
|---|---|---|---|---|---|---|
| | NON_INCREMENTAL_CATEGORY (GS-D-06) | | category net | | redesign / drop mechanic | |
| | NEGATIVE_ROI (GS-G-06) | | ROI | | renegotiate funding / depth | |
| | CENSORED_PROMO (GS-M-11) | | in-stock % | | lift is lower bound; fix allocation | |
| | SELL_IN_LIFT (shipments) (GS-D-11) | | forward-buy | | use POS / extend post window | |
| | POOLED_LIFT (<3 events) | | | | low confidence forecast | |
| | Library FVA<0 | | | | fall back to next pooling level | |

## Recommendations
1. …

## Data-Quality Log
| event | issue (overlap / censoring / missing baseline / control mismatch) | treatment |
|---|---|---|

## Reproducibility
Baseline version ids, window lengths, need-state map version, control map, pooling thresholds, depth bands, code §5.
```

## 7. Edge Cases
- **Overlapping promos on substitutes:** exclude promoted substitutes from the cannibalization pool and evaluate the joint event at need-state level (`OVERLAP`).
- **Multi-buy and BOGO mechanics:** measure in consumer units. A depth equivalent = 1 − (paid units / received units). Pull-forward is usually larger, so the default post window is 4 weeks.
- **Promo with zero baseline** (new item launched on promo): λ is undefined. Route to `nextq-new-product-lifecycle-forecasting` and exclude it from the library.
- **Negative gross lift:** report it, do not floor it. Flag `NEGATIVE_LIFT` and investigate execution (no display, out of stock, price not applied).
- **Price-only events** (TPR without display/feature): the mechanic key separates them. Do not pool them with display or feature events.
- **Retailer forward buy** (sell-in spike, flat POS): compute both sell-in and sell-out lift. Use the sell-out lift for consumer incrementality and the sell-in lift for shipment planning, and show the dip in shipments over 4–8 weeks.
- **Cannibalization exceeding the substitute baseline gap:** cap each $(B_j-A_j)^+$ at $B_j$, and flag it if the pool's baseline model has MASE > 1 (the baseline is unreliable).
