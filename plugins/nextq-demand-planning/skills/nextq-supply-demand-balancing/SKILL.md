---
name: nextq-supply-demand-balancing
description: >-
  [Next Quarter Supply Chain Suite] Runs the S&OP supply review: compares the unconstrained consensus demand plan with
  capacity and inventory, computes rough-cut capacity load and projected inventory against targets, builds a
  constrained supply plan with shelf-life and max-inventory aware pre-build, evaluates base, upside, downside and
  supply-risk scenarios in units and revenue at risk, and frames decision options (pre-build, overtime, alternate
  source, allocation, demand shaping) by time fence. Load when the user asks to balance supply and demand, run a
  supply review or rough-cut capacity check, find capacity constraints, build a constrained plan, quantify the
  demand-supply gap or compare S&OP scenarios. Inputs: unconstrained consensus demand, capacity and rates, inventory
  and targets, net prices and costs, time fences, fiscal calendar. Triggers: "supply review", "constrained plan",
  "RCCP", "demand-supply gap".
metadata:
  author: "Next Quarter"
  organization: "Next Quarter (nextq)"
  suite: "Next Quarter Supply Chain Intelligence"
  version: "1.0.0"
---

# Supply–Demand Balancing
*Part of the **Next Quarter (nextq)** Supply Chain Skill Suite — Governed by SUPPLY_CHAIN_GOLD_STANDARD.md.*

## 1. Role & Objective
**Persona:** S&OP Supply Review Lead / Master Planner.
**Decision supported:** Can we supply the unconstrained demand plan? If not, where (resource, period, product family) and by how much, and which options (with cost and revenue impact) should the management business review decide on?

**Objective:**
- A **constrained supply plan** kept separate from the unconstrained demand plan.
- Resource load and projected inventory per fiscal period, scenario results in units and value.
- A decision table with options valid for each time-fence zone.
- Allocation needs handed to `nextq-shortage-allocation-atp`, and value gaps to `nextq-demand-financial-reconciliation`.

## 2. Gold Standard Rules Enforced
| Rule | Inlined standard |
|---|---|
| **GS-G-05** Constrained vs. unconstrained | Keep separate and labeled: **unconstrained demand forecast** (what customers want; from `nextq-consensus-demand-review`), **constrained supply / sales plan** (what we will ship; produced here), and the **financial plan**. Accuracy is measured on unconstrained demand. Gaps are explicit S&OP **decisions**, never forecast adjustments. Demand shaping (price, promo, substitution) is recorded as a decision with its own expected demand effect, not as an edit to the unconstrained plan. |
| **GS-G-04** Time fences | **Frozen** (within lead time): changes only by exception approval; options limited to allocation and expedite. **Slushy:** changes with reason codes (overtime, alternate source, re-sequencing). **Liquid:** open (capacity investments, new sources, pre-build plans). |
| **GS-G-06** Value | Revenue at risk $=\sum_i \text{unmet}_i\times P^{\text{net}}_i$ (net price = list − trade / discounts), in the same currency and fiscal calendar as finance. Inventory valued at cost. The volume plan is never plugged to close a gap. |
| **GS-D-09** Buckets | Capacity (hours or units) and demand on the same fiscal calendar. Compare 4- and 5-week periods on **per-day rates**. Weekly execution plans come from monthly S&OP via `nextq-time-bucket-calendar-alignment`. |
| **GS-G-08** Allocation hand-off | When the constrained plan cannot meet demand in a period, the shortage is allocated by a **declared rule** (fair-share, priority, margin LP) in `nextq-shortage-allocation-atp`. Allocation never changes the unconstrained forecast. |
| **GS-G-10** Output contract | Executive summary, KPI table, exceptions with value, recommendations, assumptions log, reproducibility block. |
| **GS-G-11** Verification | Balance identity $I_t=I_{t-1}+\text{supply}_t-\text{served}_t$ and served + unmet = demand are asserted. Inventory is never negative in the constrained plan (unmet is reported separately). |

**Balancing parameters [RoT, configurable]:**
- **Utilization status:** > 100% `OVERLOADED`; 85–100% `WATCH` (little buffer for variability); ≤ 85% `OK`.
- **Pre-build rule:** cover each shortage from the **latest** earlier period with slack (shortest holding time), never more than `shelf_periods` ahead, and never above `max_inv`.
- **Scenarios:** base = consensus; upside / downside = the consensus P90 / P10 path (from `nextq-ml-causal-demand-forecasting` or the baseline intervals); supply risk = capacity × (1 − risk haircut).
- **Escalation:** revenue at risk above 2% of the period's demand value goes to the management business review [RoT, aligned with the GS-G-06 2% tolerance].

## 3. Required Inputs & Data Schema
| Table | Columns | Notes |
|---|---|---|
| `demand_unc` | `family, region, fiscal_period, qty_unc, qty_p10, qty_p90, version_id` | unconstrained, from consensus |
| `resources` | `resource_id, fiscal_period, available_hours, overtime_hours_max, overtime_cost_per_hour` | from the calendar (shifts × trading days) |
| `routings` | `family, resource_id, hours_per_unit` | rough-cut rates |
| `supply_sources` | `family, source_id, capacity_units, cost_per_unit, lead_time` | alternate sources |
| `inventory` | `family, region, opening_on_hand, target_inventory (SS + cycle), max_inventory, shelf_life_periods` | targets from `nextq-safety-stock-optimization` / `nextq-replenishment-policy-design` |
| `prices_costs` | `family, region, net_price, unit_cost, holding_rate` | GS-G-06 |
| `time_fences` | `family, frozen_periods, slushy_periods` | GS-G-04 |
| `fiscal_calendar` | `fiscal_period, days, trading_days` | GS-D-09 |

**Grain:** product family × region (or plant) × fiscal month (S&OP); resource × fiscal month for capacity. **Horizon:** 18–24 months [RoT].

## 4. Execution Workflow
1. **Align:** demand and capacity on fiscal periods. Convert capacity to hours from trading days × shifts. Use per-day rates when comparing periods of unequal length.
2. **Rough-cut capacity:** load$_{r,t}=\sum_f \text{qty}_{f,t}\times\text{hours/unit}_{f,r}$, utilization = load / available hours, then status.
3. **Unconstrained projection:**
   - $I_t=I_{t-1}+\text{planned supply}_t-\text{demand}_t$.
   - gap$_t=I_t-\text{target}_t$.
   - Periods with gap < 0 are shortage periods (`BELOW_TARGET`); negative $I$ is `STOCKOUT_PROJECTED`.
4. **Constrained plan (base):**
   - Convert capacity to units for each family (or run per resource for single-family resources).
   - Build to demand up to capacity, then consume opening stock, then pre-build per the rule.
   - Unmet demand remains; served + unmet = demand is asserted.
5. **Options** per shortage period, filtered by time-fence zone:

   | Option | Allowed zones |
   |---|---|
   | (a) overtime (cost/hour, up to the max) | slushy, liquid |
   | (b) alternate source (cost/unit, lead time must fit) | slushy, liquid |
   | (c) pre-build (holding cost) | liquid; slushy if capacity exists |
   | (d) allocation → `nextq-shortage-allocation-atp` | all zones, **the only option in frozen** besides expedite |
   | (e) demand shaping (decision with expected effect; owner RGM / sales) | slushy, liquid |
   | (f) accept the gap (revenue at risk) | all |

   Cost each option: incremental cost vs. revenue protected (units × net price − variable cost).
6. **Scenarios:** re-run steps 2–5 for upside, downside and supply-risk. Report unmet units, revenue at risk, average inventory value and utilization per scenario.
7. **Recommend:**
   - Rank options by net value (protected margin − cost) per period, within zone rules.
   - Show the constrained plan after recommended options. The unconstrained plan is unchanged.
8. **Escalate:** revenue at risk > 2% of period demand value, any `OVERLOADED` resource in the slushy zone, or a frozen-zone shortage.
9. **Hand-offs:** shortages to allocation; value gaps to financial reconciliation; constrained plan to execution (weekly via the calendar skill).
10. **Output** (Section 6).

## 5. Verification Snippets
```python
import numpy as np, pandas as pd

def rccp_load(demand_units, rate_hours_per_unit, capacity_hours):
    """Rough-cut capacity: load % = sum(units * hours/unit) / available hours per period (fiscal-calendar hours)."""
    load = np.asarray(demand_units, float) * rate_hours_per_unit
    util = load / np.asarray(capacity_hours, float)
    status = np.where(util > 1.0, "OVERLOADED", np.where(util > 0.85, "WATCH", "OK"))     # 85% [RoT]
    return util, status

def projected_inventory(i0, supply, demand, target):
    """I_t = I_{t-1} + supply_t - demand_t (unconstrained demand, backlog allowed as negative);
    gap_t = I_t - target_t (target = SS + cycle stock from the policy)."""
    inv = i0 + np.cumsum(np.asarray(supply, float) - np.asarray(demand, float))
    return inv, inv - np.asarray(target, float)

def prebuild(demand, cap_units, i0=0.0, max_inv=np.inf, shelf_periods=None):
    """Constrained supply plan: build to demand up to capacity; remaining shortages are covered by building EARLIER in
    the latest period with slack (minimizes holding time), respecting max inventory and shelf life (build at most
    shelf_periods ahead). Opening stock is consumed first. Returns supply plan x, ending inventory, unmet demand
    (unmet = not served in its period; whether it is lost or backlogged is an S&OP decision, never a forecast edit)."""
    d = np.asarray(demand, float); cap = np.asarray(cap_units, float); T = len(d)
    x = np.minimum(d, cap)
    need = d - x
    stock = i0
    for t in range(T):
        use = min(stock, need[t]); need[t] -= use; stock -= use
    for t in range(T):
        lo = 0 if shelf_periods is None else max(0, t - shelf_periods)
        for u in range(t - 1, lo - 1, -1):                   # latest earlier period first
            if need[t] <= 1e-12: break
            slack = cap[u] - x[u]
            if slack <= 0: continue
            inv_path = i0 + np.cumsum(x - (d - need))        # served-based inventory (earlier periods resolved)
            head = max_inv - inv_path[u:t].max()
            q = min(slack, need[t], max(head, 0.0))
            if q > 0: x[u] += q; need[t] -= q
    inv = i0 + np.cumsum(x - (d - need))
    assert (inv >= -1e-9).all()
    return x, inv, need

def scenario_eval(name, demand, cap_units, net_price, unit_cost, i0=0.0, **kw):
    """Evaluate a demand/supply scenario: unmet units, revenue at risk (GS-G-06 net price), avg inventory value, utilisation."""
    x, inv, unmet = prebuild(demand, cap_units, i0=i0, **kw)
    return dict(scenario=name, unmet_units=float(unmet.sum()), revenue_at_risk=float(unmet.sum() * net_price),
                avg_inv_value=float(np.maximum(inv, 0).mean() * unit_cost), utilisation=float(x.sum() / np.sum(cap_units)))

def per_day_rate(qty, days):
    """GS-D-09: compare 4- vs 5-week fiscal months on per-day rates."""
    return np.asarray(qty, float) / np.asarray(days, float)

def in_frozen(period_index, frozen_periods):
    """GS-G-04: options inside the frozen zone are limited to allocation / expedite (exception approval)."""
    return period_index < frozen_periods

if __name__ == "__main__":                                  # self-test: reproduces the documented test vector
    u, st = rccp_load([1000, 1200, 900], 0.5, [560, 560, 700])
    assert np.allclose(u, [0.892857, 1.071429, 0.642857], atol=1e-5) and list(st) == ["WATCH", "OVERLOADED", "OK"]
    inv, gap = projected_inventory(100, [500] * 3, [400, 700, 450], [150] * 3)
    assert inv.tolist() == [200, 0, 50] and gap.tolist() == [50, -150, -100]
    d, cap = [400, 500, 900, 700], [600] * 4
    x, inv, un = prebuild(d, cap);                    assert x.tolist() == [600] * 4 and inv.tolist() == [200, 300, 0, 0] and un.tolist() == [0, 0, 0, 100]
    x, inv, un = prebuild(d, cap, shelf_periods=1);   assert x.tolist() == [400, 600, 600, 600] and un.tolist() == [0, 0, 200, 100]
    x, inv, un = prebuild(d, cap, max_inv=250);       assert x.tolist() == [550, 600, 600, 600] and inv.max() <= 250 and un.tolist() == [0, 0, 50, 100]
    assert prebuild(d, cap, i0=150)[2].sum() == 0
    for dd in (d, [450, 600, 1000, 800]):                   # GS-G-11 balance identity: I_t - I_{t-1} = x_t - served_t
        x, inv, un = prebuild(dd, cap)
        assert np.allclose(np.diff(np.r_[0.0, inv]), x - (np.array(dd) - un)) and (inv >= 0).all()
    b = scenario_eval("base", d, cap, 12.0, 7.0); up = scenario_eval("upside", [450, 600, 1000, 800], cap, 12.0, 7.0)
    assert (b["unmet_units"], b["revenue_at_risk"], b["avg_inv_value"]) == (100, 1200, 875) and up["revenue_at_risk"] == 5400
    assert np.allclose(per_day_rate([2800, 3500], [28, 35]), [100, 100]) and in_frozen(1, 2) and not in_frozen(3, 2)
    print("self-test passed")
```
**Test vector** (executed by the `__main__` self-test above):
- **RCCP:** units [1000, 1200, 900] at 0.5 h/unit against [560, 560, 700] hours → utilization [0.893, 1.071, 0.643] = WATCH / **OVERLOADED** / OK.
- **Projection:** I₀ = 100, supply 500/period, demand [400, 700, 450], target 150 → I = [200, 0, 50], gap [+50, −150, −100].
- **Pre-build:** demand [400, 500, 900, 700], capacity 600/period.

  | Constraint | Supply plan x | Inventory | Unmet |
  |---|---|---|---|
  | None | [600, 600, 600, 600] | [200, 300, 0, 0] | 100 in period 4 |
  | Shelf life 1 period | [400, 600, 600, 600] | — | [0, 0, 200, 100] |
  | Max inventory 250 | [550, 600, 600, 600] | — | [0, 0, 50, 100] |
  | Opening stock 150 | — | — | 0 |

- **Scenarios** (net price 12, cost 7):
  - Base: unmet 100, revenue at risk **1,200**, average inventory value 875.
  - Upside: unmet 450, revenue at risk **5,400**.
- **Per-day rates:** 2,800 over 28 days vs. 3,500 over 35 days → both 100/day (no real increase between a 4- and a 5-week month).

```sql
-- Rough-cut load per resource and fiscal period with status
SELECT r.resource_id, d.fiscal_period,
       SUM(d.qty_unc * rt.hours_per_unit)                              AS load_hours,
       MAX(r.available_hours)                                          AS available_hours,
       SUM(d.qty_unc * rt.hours_per_unit) / NULLIF(MAX(r.available_hours),0) AS utilisation,
       CASE WHEN SUM(d.qty_unc * rt.hours_per_unit) > MAX(r.available_hours)        THEN 'OVERLOADED'
            WHEN SUM(d.qty_unc * rt.hours_per_unit) > 0.85 * MAX(r.available_hours) THEN 'WATCH'
            ELSE 'OK' END                                              AS status
FROM demand_unc d
JOIN routings rt ON rt.family = d.family
JOIN resources r ON r.resource_id = rt.resource_id AND r.fiscal_period = d.fiscal_period
GROUP BY 1,2;
```

## 6. Standardized Output Format
```markdown
# Supply Review & Balancing — cycle <YYYY-MM> — run BAL_<run_id>

## Executive Summary
- Unconstrained demand <U> vs constrained supply <C> over <H> periods: unmet <u> units (<v> revenue at risk, <p>% of demand value).
- Constraints: <n_over> resource-periods OVERLOADED, <n_watch> WATCH; key bottleneck <resource> in <periods>.
- Recommended: <options> protecting <units>/<value> at cost <cost>; remaining gap to MBR: <value>.
- Scenarios: upside unmet <u+>, downside excess inventory <i->, supply-risk unmet <ur>.

## Metadata (mandatory)
| Level | Grain | Horizon / time window | Calendar | Demand version | Capacity basis | Value basis | Weighting | Lag |
|---|---|---|---|---|---|---|---|---|
| family × region; resource | fiscal month | <start–end> | 4-4-5 (per-day rates) | consensus <id> (unconstrained) | hours (trading days × shifts) | net price; cost | units / value | n/a (plan) |

## KPI Table
| Period | Zone | Demand unc | Constrained supply | Unmet | Revenue at risk | Proj. inventory | Target | Gap | Max utilisation | Status |
|---|---|---|---|---|---|---|---|---|---|---|

## Resource Load
| Resource | Period | Load h | Available h | Utilisation | Status | Overtime available |
|---|---|---|---|---|---|---|

## Scenarios
| Scenario | Unmet units | Revenue at risk | Avg inventory value | Utilisation | Key option |
|---|---|---|---|---|---|

## Decision Options (for MBR)
| ID | Period | Zone | Option | Units protected | Revenue protected | Cost | Net value | Owner | Decision |
|---|---|---|---|---|---|---|---|---|---|

## Exceptions
| ID | Trigger (rule) | family / resource | metric | value | action | owner |
|---|---|---|---|---|---|---|
| | Revenue at risk >2% (GS-G-06) | | % | | MBR decision | S&OP leader |
| | OVERLOADED in slushy zone (GS-G-04) | | util | | overtime / alternate source | Supply planner |
| | Frozen-zone shortage (GS-G-04) | | units | | allocation (nextq-shortage-allocation-atp) | Supply / sales |
| | Plan edited to close gap (GS-G-05) | | | | revert; record decision | Demand manager |

## Recommendations
1. …

## Data-Quality Log
| object | issue (missing routings, capacity not calendarised, P10/P90 missing → ±% assumption) | treatment |
|---|---|---|

## Reproducibility
Demand version, capacity snapshot, rules (85%, pre-build), scenario definitions, code §5.
```

## 7. Edge Cases
- **Shared resources across families:** pre-build and options are evaluated at the resource level first, with capacity split by the declared priority. Otherwise use the margin-per-hour order [RoT], logged.
- **Pre-build beyond shelf life or into a lower-price period:** blocked by `shelf_periods`. Report the holding and obsolescence cost of any pre-build longer than 2 periods [RoT].
- **Downside scenario with excess:** report excess inventory value and route it to `nextq-inventory-health-optimization` (push-out or cancel supply first).
- **Supplier capacity unknown:** use the contractual volume with a `CAPACITY_ASSUMED` flag. Never assume infinite capacity.
- **Demand in different UoM from capacity** (cases vs. pallets vs. hours): convert through routings and pack master data; missing conversions are excluded and reported.
- **Backlog vs. lost demand:** B2B shortfalls may backlog into the next period (add to next-period demand as an explicit decision). Retail shortfalls are lost and reported as revenue at risk. Never silently carry over.
- **New products** (no history-based capacity rates): use engineering rates, flagged, until 3 periods of actuals.
