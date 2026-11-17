---
name: nextq-demand-financial-reconciliation
description: >-
  [Next Quarter Supply Chain Suite] Reconciles the unconstrained consensus demand plan with the financial plan or
  budget each S&OP cycle: volume-to-value at net price (list less discounts, trade and rebates), currency and
  fiscal-calendar alignment (4-4-5, calendar months, 53-week years), plan-vs-budget gap with a 2% escalation
  tolerance, exact Price-Volume-Mix and FX bridges, revenue at risk from supply constraints, and anti-plugging checks.
  Load when the user asks to convert the demand plan to revenue, compare forecast to budget or AOP, explain the gap to
  the financial plan, build a PVM or revenue bridge, prepare the S&OP financial view, or check whether volume was
  plugged. Inputs: consensus and constrained plans, financial plan, gross-to-net prices, FX rates, fiscal calendar,
  finance hierarchy mapping. Triggers: "volume to value", "forecast vs budget", "PVM", "revenue bridge", "gap to
  plan", "AOP".
metadata:
  author: "Next Quarter"
  organization: "Next Quarter (nextq)"
  suite: "Next Quarter Supply Chain Intelligence"
  version: "1.0.0"
---

# Demand–Financial Reconciliation
*Part of the **Next Quarter (nextq)** Supply Chain Skill Suite — Governed by SUPPLY_CHAIN_GOLD_STANDARD.md.*

## 1. Role & Objective
**Persona:** S&OP / IBP Finance Partner (demand-to-revenue reconciliation lead).
**Decision supported:** Does the agreed unconstrained demand plan, valued at net price, support the financial plan? If not, how big is the gap, what drives it (volume, mix, price, FX, supply constraint), and which decisions go to the management business review?

**Objective:**
- An exact, reproducible bridge from the financial plan to the demand plan value at the agreed reconciliation level (default brand × region × fiscal month).
- An escalation list for gaps beyond tolerance.
- A guarantee that the **volume plan is never silently plugged** to hit the financial number.

## 2. Gold Standard Rules Enforced
| Rule | Inlined standard |
|---|---|
| **GS-G-06** Volume-to-value | $\mathit{V}=\sum_i Q_i\times P_i^{\text{net}}$, where net price = list − trade spend / discounts (here: list − on-invoice discounts − off-invoice trade − rebates, per base unit, on the same gross-to-net basis finance uses). Same currency and same fiscal calendar on both sides. **Gap** $=V_{\text{demand}}-V_{\text{financial}}$ is reported, decomposed by GS-G-07, and resolved in the management business review. **Tolerance [RoT]:** $\lvert\text{gap}\rvert/V_{\text{fin}} > 2\%$ at the reconciliation level → **ESCALATE**. **Never plug** the volume plan. |
| **GS-G-07** PVM [STD accounting practice] | Plan (0) vs. forecast (1), with $\mathit{Q}$ = total volume and $\mathit{s}_i=Q_i/Q$ the mix share. **Volume** $=(Q_1-Q_0)\bar P_0$ with $\bar P_0=\sum_i s_{i,0}P_{i,0}$. **Mix** $=Q_1\sum_i(s_{i,1}-s_{i,0})P_{i,0}$. **Price** $=\sum_i Q_{i,1}(P_{i,1}-P_{i,0})$. Check: Volume + Mix + Price = $\mathit{V}_1-V_0$ **exactly** (residual asserted ≤ $10^{-9}$ relative). |
| **GS-G-05** Plans kept separate | Three labeled plans: **unconstrained consensus demand** (from `nextq-consensus-demand-review`), **constrained supply / sales plan** (from `nextq-supply-demand-balancing`), and **financial plan**. Gaps between them are S&OP decisions, never forecast edits. Revenue at risk from supply $=\sum_i (Q^{\text{unc}}_i-Q^{\text{con}}_i)^+P^{\text{net}}_i$. |
| **GS-D-09** Time buckets | One fiscal calendar table (`date → fiscal_week, fiscal_period, fiscal_quarter, fiscal_year, days_in_period`). Weekly → **calendar** months: split each week by days in each month (optionally day-of-week weighted), never assigned wholesale. Weekly → **4-4-5 fiscal** periods map whole weeks by the calendar table. Compare unequal periods on per-day rates. |
| **GS-D-10** 4-4-5 / 53-week | Quarters are 13 weeks; a 53rd week is inserted periodically. If finance and demand calendars disagree on week 53, align explicitly (drop, merge, or dummy) and report the effect as a separate `CALENDAR` bridge line. |
| **GS-G-11** Verification | PVM sums exact; hierarchy roll-ups $\lvert\text{parent}-\sum\text{children}\rvert\le10^{-6}$ relative; calendar apportionment conserves totals. |
| **GS-G-10 / GS-D-13** | Standard output; every price, FX, or mapping substitution logged (before, after, rule, timestamp). |

**Additional bridge conventions [RoT / practice, documented]:**
- **New items** (in the forecast, not in the plan) take $\mathit{P}_0:=P_1$, so their price effect is 0 and their value flows through volume and mix. **Discontinued** items (in the plan, not in the forecast) take $\mathit{P}_1:=P_0$. Both are listed in a NEW/DISC sub-table.
- **FX:** PVM is computed in **constant currency** (plan FX). FX effect $=\sum_i Q_{i,1}P^{\text{local}}_{i,1}(\text{FX}_1-\text{FX}_0)$ is a separate bridge line.
- **Bridge order:** Plan → Volume → Mix → Price → FX → Calendar → Forecast. The lines must sum exactly.

## 3. Required Inputs & Data Schema
| Table | Key columns | Notes |
|---|---|---|
| `consensus_plan` | `item_id, location_id/region, period (week or month), qty_unc, version_id, lag` | Unconstrained, base UoM |
| `constrained_plan` | `item_id, region, period, qty_con` | From supply review |
| `financial_plan` | `rec_level_key (brand × region × fiscal_month), value_fin, currency, qty_fin (optional), price_fin (optional)` | If finance plans by item, item level is allowed |
| `price_list` | `item_id, customer_group/region, valid_from, valid_to, list_price, on_invoice_disc, off_invoice_trade, rebates, currency` | Gross-to-net per base unit |
| `fx_rates` | `currency, period, fx_plan, fx_actual_or_forecast` | Reporting currency per local unit |
| `fiscal_calendar` | `date, fiscal_week, fiscal_month, fiscal_quarter, fiscal_year, is_week53` | Single source (GS-D-09) |
| `finance_mapping` | `item_id → brand, region → finance_region, valid_from` | Unmapped SKUs go to `UNMAPPED` |
| `override_log` (optional) | `item_id, period, delta_qty, reason_code, cycle` | For the plugging check |

**Grain:** reconciliation level brand × region × fiscal month (default); the bridge is drillable to item × region × month. **Minimum:** one financial plan version and the current consensus version for the same horizon.

## 4. Execution Workflow
1. **Align calendars:** convert consensus weeks to the finance period with the calendar table (day-split for calendar months; whole-week mapping for 4-4-5). Assert totals are conserved. Treat week 53 per policy and log it.
2. **Map the hierarchy:** item → brand, region → finance region. Unmapped volume goes to `UNMAPPED`; if its share of value is > 0.5% [RoT], flag `MAPPING_GAP`.
3. **Net prices:**
   - Pick the price valid at each period (validity dates). $\mathit{P}^{\text{net}}=\text{list}-\text{on-invoice}-\text{off-invoice}-\text{rebates}$.
   - Missing price: use the last valid price, flagged `PRICE_CARRIED`. If none, flag `PRICE_MISSING` and exclude from value (show the units).
   - $\mathit{P}^{\text{net}}\le 0$: flag `NONPOSITIVE_NET_PRICE`.
4. **Value the plans** in local currency, then convert at plan FX (constant currency) and at forecast FX:
   - unconstrained $\mathit{V}_{\text{unc}}=\sum Q^{\text{unc}}P^{\text{net}}$
   - constrained $\mathit{V}_{\text{con}}$
   - financial $\mathit{V}_{\text{fin}}$
5. **Gap and status** per reconciliation cell and total:
   - gap $=V_{\text{unc}}-V_{\text{fin}}$; gap% = gap / $\mathit{V}_{\text{fin}}$
   - $\lvert\text{gap\%}\rvert>2\%$ → ESCALATE
   - $\mathit{V}_{\text{fin}}=0$ with $\mathit{V}_{\text{unc}}>0$ → ESCALATE (`NO_FIN_PLAN`)
6. **Bridge:**
   - If finance supplies item-level qty and price, run full PVM (GS-G-07) at item level within each cell, then FX and calendar lines.
   - If finance supplies value only, the bridge is Plan → Demand value with a single `UNEXPLAINED_NO_FIN_DETAIL` line, flagged.
   - Assert the exact sum.
7. **Supply view:** revenue at risk $=\sum (Q^{\text{unc}}-Q^{\text{con}})^+P^{\text{net}}$ per cell. Report $\mathit{V}_{\text{con}}$ vs. $\mathit{V}_{\text{fin}}$ separately. This gap is a supply decision (`nextq-supply-demand-balancing`), not a demand change.
8. **Anti-plugging check:**
   - Compare this cycle's consensus to last cycle's.
   - Volume change without approved reason codes (override log) above 1% of cell value [RoT], **and** a gap that moved from outside to inside tolerance → `PLUGGING_SUSPECTED`. The cell is escalated with the before/after.
   - Never adjust volumes in this skill.
9. **Resolution options** for each escalated cell, listed for the management business review:
   - (i) demand upside/risk scenarios with probability
   - (ii) price or trade-spend actions (owner: RGM)
   - (iii) mix actions
   - (iv) supply recovery
   - (v) financial plan re-forecast

   The chosen option is recorded as a decision, not as a forecast edit.
10. **Output** (Section 6).

## 5. Verification Snippets
```python
import numpy as np, pandas as pd

def net_price(list_price, on_invoice_disc=0.0, off_invoice_trade=0.0, rebates=0.0):
    """GS-G-06 net price per base unit = list - on-invoice discounts - off-invoice trade spend - rebates (finance basis)."""
    return list_price - on_invoice_disc - off_invoice_trade - rebates

def value(df, q="qty", p="net_price", fx="fx_rate"):
    """V = sum Q*P_net (converted to reporting currency at the stated rate)."""
    return float((df[q] * df[p] * df.get(fx, 1.0)).sum())

def pvm(plan, fcst, key="item_id"):
    """GS-G-07 Price-Volume-Mix, plan(0) vs forecast(1). New items (Q0=0) take P0:=P1 (price effect 0);
    discontinued items (Q1=0) contribute through volume+mix only. Exact: V+M+P = V1-V0."""
    m = plan[[key, "qty", "net_price"]].merge(fcst[[key, "qty", "net_price"]], on=key, how="outer",
                                              suffixes=("0", "1")).fillna({"qty0": 0.0, "qty1": 0.0})
    m["net_price0"] = m["net_price0"].fillna(m["net_price1"])    # NEW item
    m["net_price1"] = m["net_price1"].fillna(m["net_price0"])    # DISCONTINUED item
    Q0, Q1 = m.qty0.sum(), m.qty1.sum()
    V0, V1 = (m.qty0 * m.net_price0).sum(), (m.qty1 * m.net_price1).sum()
    if Q0 == 0 or Q1 == 0:                                        # NEW or DISC cell: mix undefined
        return dict(V0=V0, V1=V1, volume=V1 - V0, mix=0.0, price=0.0, residual=0.0)
    s0, s1 = m.qty0 / Q0, m.qty1 / Q1
    P0bar = (s0 * m.net_price0).sum()
    vol = (Q1 - Q0) * P0bar
    mix = Q1 * ((s1 - s0) * m.net_price0).sum()
    price = (m.qty1 * (m.net_price1 - m.net_price0)).sum()
    resid = (V1 - V0) - (vol + mix + price)
    assert abs(resid) <= 1e-9 * max(abs(V1), 1.0), f"PVM residual {resid}"
    return dict(V0=V0, V1=V1, volume=vol, mix=mix, price=price, residual=resid)

def fx_effect(qty1, p1_local, fx_plan, fx_actual):
    """FX effect isolated after constant-currency PVM: sum Q1*P1_local*(FX_actual - FX_plan)."""
    return float(np.sum(np.asarray(qty1) * np.asarray(p1_local) * (np.asarray(fx_actual) - np.asarray(fx_plan))))

def gap_status(v_demand, v_fin, tol=0.02):
    """Gap = V_demand - V_fin; |gap|/V_fin > 2% escalates [RoT]."""
    gap = v_demand - v_fin
    pct = gap / v_fin if v_fin else np.nan
    return gap, pct, ("ESCALATE" if (not np.isfinite(pct)) or abs(pct) > tol else "WITHIN_TOLERANCE")

def weeks_to_fiscal_months(weekly, calendar):
    """GS-D-09: apportion weekly qty to the month of each day (sum of 7 day-shares). calendar: date -> fiscal_month."""
    rows = []
    for _, r in weekly.iterrows():
        days = pd.date_range(r.week_start, periods=7, freq="D")
        months = calendar.reindex(days)
        assert months.notna().all(), "CALENDAR_GAP"
        for mth, n in months.value_counts().items():
            rows.append((r.item_id, mth, r.qty * n / 7.0))
    out = pd.DataFrame(rows, columns=["item_id", "fiscal_month", "qty"]).groupby(["item_id", "fiscal_month"], as_index=False).qty.sum()
    assert abs(out.qty.sum() - weekly.qty.sum()) <= 1e-9 * max(weekly.qty.sum(), 1)
    return out

def plugging_suspected(delta_volume_unexplained_pct, gap_before_pct, gap_after_pct, tol=0.02, min_delta=0.01):
    """Volume moved without reason codes and the gap closed to within tolerance -> PLUGGING_SUSPECTED."""
    return delta_volume_unexplained_pct > min_delta and abs(gap_before_pct) > tol and abs(gap_after_pct) <= tol
```
**Test vector** (verified):
- **Net price:** 10 − 1.0 − 0.5 − 0.3 = **8.2**.
- **PVM**, plan {A 100 @ 10, B 50 @ 20, C 20 @ 5} vs. forecast {A 110 @ 10.5, B 40 @ 20, D (new) 30 @ 8}:
  - V0 = 2,100, V1 = 2,195
  - Volume +123.53, Mix −83.53, Price +55.00; residual 0
- **FX:** Q {10, 20} × P_local 5 with FX 1.10 → {1.08, 1.12} → effect **+1.00**.
- **Gap** 2,195 vs. 2,100 = +95 (+4.52%) → **ESCALATE**. 2,050 vs. 2,048 = +0.10% → WITHIN_TOLERANCE.
- **Calendar split:** weeks starting 21 Jan and 28 Jan (70 each) give Jan 110 / Feb 30 (the 28 Jan week has 4 January days and 3 February days). Total conserved.
- **Plugging:** 3% unexplained volume moved the gap from −5% to −1% → `PLUGGING_SUSPECTED` = True.

```sql
-- Gap per reconciliation cell (constant currency) with escalation status
WITH dem AS (
  SELECT m.brand, m.finance_region, c.fiscal_month,
         SUM(p.qty_unc * (pl.list_price - pl.on_invoice_disc - pl.off_invoice_trade - pl.rebates) * fx.fx_plan) AS v_unc
  FROM consensus_plan_monthly p
  JOIN finance_mapping m ON m.item_id = p.item_id
  JOIN fiscal_month_dim c ON c.fiscal_month = p.fiscal_month
  JOIN price_list pl ON pl.item_id = p.item_id AND pl.region = p.region
                     AND c.month_start BETWEEN pl.valid_from AND pl.valid_to
  JOIN fx_rates fx ON fx.currency = pl.currency AND fx.period = p.fiscal_month
  GROUP BY 1,2,3)
SELECT d.*, f.value_fin, d.v_unc - f.value_fin AS gap,
       (d.v_unc - f.value_fin) / NULLIF(f.value_fin,0) AS gap_pct,
       CASE WHEN f.value_fin IS NULL OR f.value_fin = 0 OR ABS((d.v_unc - f.value_fin)/f.value_fin) > 0.02
            THEN 'ESCALATE' ELSE 'WITHIN_TOLERANCE' END AS status
FROM dem d FULL JOIN financial_plan f
  ON f.brand = d.brand AND f.finance_region = d.finance_region AND f.fiscal_month = d.fiscal_month;
```

## 6. Standardized Output Format
```markdown
# Demand–Financial Reconciliation — <cycle YYYY-MM> — run FIN_<run_id>

## Executive Summary
- Unconstrained demand value <V_unc> vs financial plan <V_fin>: gap <gap> (<gap%>); <n_esc> of <n_cells> cells escalated (>2%).
- Bridge: volume <v>, mix <m>, price <p>, FX <fx>, calendar <c> (sums exactly).
- Revenue at risk from supply constraints: <R> (<r%> of V_unc) — routed to supply review.
- Plugging check: <n_plug> cells PLUGGING_SUSPECTED. Confidence: <price coverage %>, <mapping coverage %>.

## Metadata (mandatory)
| Level | Grain | Time window | Calendar | Currency / FX basis | Price basis | Plan versions | Lag | Weighting |
|---|---|---|---|---|---|---|---|---|
| brand × region (drill: item) | fiscal month | <horizon> | 4-4-5 / calendar, week-53 policy | <ccy>, constant (plan FX) + FX line | net (list − on − off − rebates) | consensus <v>, fin <v>, constrained <v> | <ℓ> | value |

## KPI Table
| Cell | V_unc | V_con | V_fin | Gap | Gap % | Status | Revenue at risk | Price coverage % |
|---|---|---|---|---|---|---|---|---|

## Bridge (Plan → Demand value)
| Line | Value | % of plan | Driver detail |
|---|---|---|---|
| Financial plan | | | |
| Volume | | | (Q1−Q0)·P̄0 |
| Mix | | | Q1·Σ(s1−s0)P0 |
| Price | | | ΣQ1(P1−P0) |
| FX | | | ΣQ1·P1_local·ΔFX |
| Calendar | | | week-53 / period alignment |
| **Demand value** | | | residual = 0 ✔ |

## NEW / DISCONTINUED Items
| item | status | Q0 | Q1 | P used | value effect |
|---|---|---|---|---|---|

## Exceptions
| ID | Trigger (rule) | cell / item | metric | value | action | owner |
|---|---|---|---|---|---|---|
| | Gap >2% (GS-G-06) | | gap % | | MBR decision | S&OP leader |
| | PLUGGING_SUSPECTED (GS-G-06) | | unexplained Δ | | revert / justify | Demand manager |
| | PRICE_MISSING / NONPOSITIVE_NET_PRICE | | | | fix price master | Pricing |
| | MAPPING_GAP >0.5% | | | | fix mapping | Master data |
| | Revenue at risk (GS-G-05) | | | | supply review | Supply planner |

## Recommendations
1. …

## Data-Quality Log
| item / cell | issue (price carried, FX missing, unmapped, calendar conflict) | treatment |
|---|---|---|

## Reproducibility
Plan version IDs, price list snapshot, FX table, calendar version, mapping version, tolerance (2%), code §5.
```

## 7. Edge Cases
- **Finance plans in a different UoM or at gross price:** convert to base UoM and the same gross-to-net basis before comparing, or the gap is meaningless (`BASIS_MISMATCH`).
- **Returns and credits:** if finance plans net of returns, apply the planned return rate to $\mathit{Q}$ as a separate line (`RETURNS`). Never net it into the demand plan.
- **Retroactive rebates or price increases** effective mid-period: split the period by validity dates (day-weighted) and log it.
- **Negative gap concentrated in one item:** drill the PVM to item level and show the top 10 contributors.
- **Volume with zero total** in plan or forecast at a cell ($\mathit{Q}_0=0$ or $\mathit{Q}_1=0$): mix shares are undefined. Report the whole value change as volume (NEW or DISC cell) and skip the mix line.
- **Multiple financial plan versions** (AOP vs. latest estimate): reconcile against the version declared in metadata; show the other as a reference column.
- **Intercompany or transfer pricing:** use external customer net price only. Intercompany volume is excluded and flagged.
