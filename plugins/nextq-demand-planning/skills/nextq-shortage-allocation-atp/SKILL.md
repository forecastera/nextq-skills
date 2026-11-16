---
name: nextq-shortage-allocation-atp
description: >-
  [Next Quarter Supply Chain Suite] Allocates constrained supply when demand exceeds it and computes
  available-to-promise: runs fair-share (with floors and weights), priority-tier and margin-optimal LP allocation
  under a declared rule, integerizes by largest remainder, computes cumulative ATP with look-ahead and allocated ATP
  by channel, quotes promise dates, and reports each rule's projected fill rate, OTIF and margin while leaving the
  unconstrained forecast unchanged. Load when the user asks how to allocate scarce inventory or capacity, ration
  product across customers or channels, compare fair-share vs priority vs margin allocation, compute ATP or CTP-style
  promise dates, or quantify OTIF impact of a shortage. Inputs: open orders with requested dates, constrained supply,
  commitments, customer tiers and floors, margins. Triggers: "allocation", "shortage", "fair share", "ATP", "order
  promising", "rationing".
metadata:
  author: "Next Quarter"
  organization: "Next Quarter (nextq)"
  suite: "Next Quarter Supply Chain Intelligence"
  version: "1.0.0"
---

# Shortage Allocation & Available-to-Promise
*Part of the **Next Quarter (nextq)** Supply Chain Skill Suite — Governed by SUPPLY_CHAIN_GOLD_STANDARD.md.*

## 1. Role & Objective
**Persona:** Supply Allocation Manager / Customer Service & Order Promising Lead.
**Decision supported:** When constrained supply is below demand, who gets how much and when? What can we promise a new order without breaking existing commitments?

**Objective:**
- Allocate scarce supply by a **declared, auditable rule**: fair share, priority tiers or margin-optimal LP.
- Compute cumulative and allocated ATP with look-ahead.
- Quote promise dates.
- Report each rule's projected impact in **fill-rate (GS-K-09)** and **OTIF (GS-K-10)** terms, plus margin.
- The unconstrained demand forecast is left untouched.

## 2. Gold Standard Rules Enforced
| Rule | Inlined standard |
|---|---|
| **GS-G-08** Allocation under shortage | When constrained supply < demand, allocate by a declared rule: **fair share** (proportional to demand or forecast), **priority tiers** (customer or segment SLA), or **margin-optimal** (an LP). Log the rule and its outcome in OTIF and fill-rate terms. **Allocation never changes the unconstrained forecast.** |
| **GS-G-05** Constrained vs. unconstrained | Allocations and promises are the **constrained** plan. Demand history keeps the requested quantity and date (for later reconstruction by `nextq-unconstrained-demand-reconstruction`). Accuracy is measured on unconstrained demand only. |
| **GS-K-09** Fill rate | Unit fill rate = units shipped from stock on first pass / units **requested** (unconstrained). State the basis (units, lines, orders). Backfilled shipments do not count as filled. |
| **GS-K-10** OTIF | OTIF = # orders (or lines) on time **AND** in full / # orders (or lines). **On time** against the customer-requested or committed date (state which) within a window, default −1/+0 days [RoT]. **In full** means delivered ≥ ordered. Both must hold for the same order. State the measurement point (ship vs. receipt). Note: measuring against a *re-committed* date after a shortage inflates OTIF, so report both bases when promises were re-dated. |
| **GS-G-11** Verification | Σ allocations = min(supply, Σ demand) within 1e-6. $0\le x_i\le d_i$. Integer units preserve the total (largest-remainder rounding). ATP never promises beyond cumulative supply minus commitments in any future period. |
| **GS-G-03** Exceptions | Customers below their SLA floor or contractual minimum are ranked by value at risk × ABC weight (3/2/1). Top N shown. |

**Allocation formulas:**
- **Fair share:** $x_i=\min\big(d_i,\ \lambda w_i d_i\big)$, with λ set so that $\sum x_i=\min(S,\sum d_i)$ (cap-and-redistribute). Optional floors $f_i$ (contract minimums) are served first. Weights $w_i$ default to 1 (proportional to demand). Integerized by largest remainder (ties go to the lower index, a documented deterministic rule).
- **Priority tiers:** tier 1 fully, then tier 2, …; fair share within the tier that runs short.
- **Margin LP:** $\max\sum_i m_i x_i$ s.t. $\sum_i x_i\le S$, $\ \phi d_i\le x_i\le d_i$. Protection floor φ = 0.5 [RoT]; if $\phi\sum d_i>S$, the LP is infeasible and φ must be lowered and logged.
- **Cumulative ATP (look-ahead):** $CATP_t=\max\big(0,\ \min_{u\ge t}[\text{CumSupply}_u-\text{CumCommitted}_u]\big)$, where supply includes on-hand in period 1.
  - Discrete ATP: $ATP_t=CATP_t-CATP_{t-1}$.
  - A new order of q units for period t is promisable iff $q\le CATP_t$. Otherwise offer the earliest t′ with $CATP_{t'}\ge q$, or a partial quantity now.
- **Allocated ATP:** $CATP_t$ split by channel or customer shares from the declared rule (integer largest remainder). Unused allocations are released back to the pool at a declared time, e.g. 1 period before the frozen fence [RoT].

**Rule choice guidance** (declare in the S&OP or MBR, not per order):

| Rule | When to use |
|---|---|
| Fair share | Default when contracts are silent. Spreads the pain; maximizes the number of partially served customers. |
| Priority | Contractual SLAs or strategic accounts. Maximizes OTIF for top tiers. |
| Margin LP | Short-term profit protection with floors, so that no customer drops below φ. |

Always show all three side by side before the decision.

## 3. Required Inputs & Data Schema
| Table | Columns | Notes |
|---|---|---|
| `demand_orders` | `order_id, line_id, customer_id, item_id, location_id, requested_qty, requested_date, committed_date, order_date` | unconstrained requested quantity and date |
| `forecast_unc` | `customer_or_channel, item_id, period, qty` | unconstrained, used for allocating future periods not yet ordered |
| `supply` | `item_id, location_id, period, on_hand, scheduled_receipts` | constrained supply (from `nextq-supply-demand-balancing`) |
| `commitments` | `item_id, location_id, period, committed_qty` | already-promised orders |
| `customers` | `customer_id, tier, sla_fill_target, contract_min_qty, abc_class` | tiers and floors |
| `economics` | `customer_id, item_id, net_price, unit_cost` | margin m = net price − cost (GS-G-06 basis) |
| `rule_config` | `rule (fair/priority/lp), weights, phi, release_time, otif_window, date_basis, measurement_point` | declared and logged |

**Grain:** item × location × period (allocation); order line (promising and OTIF).

## 4. Execution Workflow
1. **Detect shortage:** for each item × location × period, compare supply available ($CATP$) with open demand + forecast. Shortage when Σ demand > supply. If none, promise normally (step 6).
2. **Declare the rule:** read `rule_config` (approved in S&OP/MBR). If missing, compute all three and stop for a decision (`RULE_NOT_DECLARED`). Never allocate silently.
3. **Allocate:** run fair share, priority and margin LP. Verify each with `verify_allocation` (GS-G-11).
4. **Score:**
   - Per rule: allocated units, margin, projected unit fill rate (Σx/Σd), projected OTIF (an order is in full only if $x_i\ge d_i$, and on time per window and date basis), and customers below their SLA or contract floor.
   - Present a side-by-side table and recommend per the declared rule.
5. **Publish allocations** as allocated ATP by customer or channel and period; set the release time for unused allocation.
6. **Promise:**
   - For each new order: if $q\le$ the customer's allocated ATP (or $CATP_t$ when unallocated), promise the requested date.
   - Otherwise offer (a) the earliest full date, (b) a partial now plus the balance at the earliest date, or (c) substitution, recorded as a decision.
   - Keep the requested date for OTIF and the reconstruction skill.
7. **Record demand integrity:** requested quantities and dates stay in history, flagged with `ALLOCATED` (input to `nextq-unconstrained-demand-reconstruction`, GS-M-11). The unconstrained forecast is unchanged (assert the version hash equals the input).
8. **Exceptions** (GS-G-03): customers below floor, projected OTIF below the tier target, LP infeasible at φ, frozen-zone re-promises. Ranked by value at risk × ABC weight 3/2/1.
9. **Post-period:** measured fill rate and OTIF go to `nextq-service-level-performance-analysis` (requested vs. committed date basis both shown).
10. **Output** (Section 6).

## 5. Verification Snippets
```python
import numpy as np
from scipy.optimize import linprog

def _largest_remainder(raw, cap, total):
    """Integerize: floor, then give remaining units to largest fractional parts (ties -> lower index), never above cap."""
    base = np.minimum(np.floor(raw + 1e-9), cap); left = int(round(total - base.sum()))
    order = np.lexsort((np.arange(len(raw)), -(raw - np.floor(raw + 1e-9))))
    for i in order:
        if left <= 0: break
        if base[i] < cap[i]: base[i] += 1; left -= 1
    return base

def fair_share(supply, demand, floors=None, weights=None):
    """GS-G-08 fair share: x_i proportional to w_i * d_i, capped at d_i (cap-and-redistribute), after optional floors
    (contractual minimums, served first, pro rata if the floors themselves exceed supply). Integer units by largest
    remainder. Sum x = min(supply, sum d)."""
    d = np.asarray(demand, float); w = np.ones_like(d) if weights is None else np.asarray(weights, float)
    total = min(float(supply), d.sum())
    f = np.zeros_like(d) if floors is None else np.minimum(np.asarray(floors, float), d)
    if f.sum() >= total:
        return _largest_remainder(f * total / f.sum(), f, total) if f.sum() > 0 else np.zeros_like(d)
    x = f.copy(); rem = total - f.sum(); open_ = d - x > 1e-12
    while rem > 1e-9 and open_.any():                       # cap-and-redistribute on continuous shares
        sh = np.where(open_, w * d, 0.0); add = rem * sh / sh.sum()
        add = np.minimum(add, d - x); x += add; rem = total - x.sum(); open_ = d - x > 1e-9
    return _largest_remainder(x, d, total)

def priority_allocation(supply, demand, tier):
    """GS-G-08 priority tiers: tier 1 fully before tier 2 ...; fair share within the tier that runs short."""
    d = np.asarray(demand, float); tier = np.asarray(tier); x = np.zeros_like(d); rem = float(supply)
    for t in sorted(set(tier.tolist())):
        m = tier == t
        if rem <= 0: break
        if d[m].sum() <= rem: x[m] = d[m]; rem -= d[m].sum()
        else: x[m] = fair_share(rem, d[m]); rem = 0
    return x

def margin_lp(supply, demand, unit_margin, min_fill=0.0):
    """GS-G-08 margin-optimal LP: max sum m_i x_i  s.t. sum x_i <= supply, min_fill*d_i <= x_i <= d_i.
    min_fill (e.g. 0.5 [RoT]) protects every customer; infeasible if sum(min_fill*d) > supply -> lower min_fill."""
    d = np.asarray(demand, float); m = np.asarray(unit_margin, float)
    res = linprog(-m, A_ub=[np.ones_like(d)], b_ub=[supply], bounds=list(zip(min_fill * d, d)), method="highs")
    if res.status != 0: raise ValueError("LP infeasible: floors exceed supply; lower min_fill")
    return np.round(res.x, 6)

def cumulative_atp(on_hand, receipts, committed):
    """Cumulative ATP with look-ahead: CATP_t = min_{u>=t} (cum supply_u - cum committed_u), floored at 0.
    A new order for period t is promisable iff qty <= CATP_t (it consumes supply from t onward without breaking any
    future commitment). Discrete ATP_t = CATP_t - CATP_{t-1}."""
    s = np.asarray(receipts, float).copy(); s[0] += on_hand
    net = np.cumsum(s) - np.cumsum(np.asarray(committed, float))
    catp = np.maximum(np.minimum.accumulate(net[::-1])[::-1], 0.0)
    return catp, np.diff(np.r_[0.0, catp])

def promise(qty, req_period, catp):
    """Earliest period >= req_period with CATP >= qty (full promise); also the quantity available on the request date."""
    ok = np.nonzero(catp[req_period:] >= qty)[0]
    return (int(req_period + ok[0]) if ok.size else None), float(min(qty, catp[req_period]))

def allocated_atp(catp_t, shares):
    """Allocated ATP: split ATP by channel shares (from the declared allocation rule) with integer largest remainder."""
    sh = np.asarray(shares, float) / np.sum(shares)
    return _largest_remainder(sh * catp_t, np.full(len(sh), np.inf), np.floor(catp_t))

def projected_service(alloc, demand, promised_period, requested_period, window=(-1, 0)):
    """K-09 fill rate = sum alloc / sum requested (unconstrained) qty; K-10 OTIF per order: in full (alloc >= ordered)
    AND on time (promised - requested within window, -1/+0 periods [RoT]) for the SAME order."""
    a, d = np.asarray(alloc, float), np.asarray(demand, float)
    delta = np.asarray(promised_period) - np.asarray(requested_period)
    on_time = (delta >= window[0]) & (delta <= window[1]); in_full = a >= d - 1e-9
    return dict(fill_rate=a.sum() / d.sum(), otif=float(np.mean(on_time & in_full)), in_full=float(in_full.mean()), on_time=float(on_time.mean()))

def verify_allocation(x, demand, supply):
    """GS-G-11: sum x = min(supply, sum d) (within 1e-6), 0 <= x <= d."""
    d = np.asarray(demand, float)
    assert abs(x.sum() - min(supply, d.sum())) <= 1e-6 and (x >= -1e-9).all() and (x <= d + 1e-9).all()
    return True

if __name__ == "__main__":                                  # self-test: reproduces the documented test vector
    d = [500, 300, 150, 50]
    x = fair_share(700, d); assert x.tolist() == [350, 210, 105, 35] and verify_allocation(x, d, 700)
    assert fair_share(700, d, floors=[0, 0, 150, 50]).tolist() == [313, 187, 150, 50]
    assert fair_share(700, d, weights=[1, 1, 1, 4]).tolist() == [342, 205, 103, 50]
    assert fair_share(2000, d).tolist() == d and fair_share(10, [1, 1, 1]).tolist() == [1, 1, 1]
    xt = priority_allocation(700, d, [2, 1, 1, 2]); assert xt.tolist() == [227, 300, 150, 23] and verify_allocation(xt, d, 700)
    xl = margin_lp(700, d, [4, 6, 5, 3], min_fill=0.5); assert xl.tolist() == [250, 300, 125, 25] and verify_allocation(xl, d, 700)
    assert margin_lp(700, d, [4, 6, 5, 3]).tolist() == [250, 300, 150, 0]
    for alloc in (x, xt, xl):                                 # margin and service trade-off of each declared rule
        print([int(v) for v in alloc], "margin", float(np.dot(alloc, [4, 6, 5, 3])), "otif", projected_service(alloc, d, [0] * 4, [0] * 4)["otif"])
    catp, atp = cumulative_atp(100, [0, 200, 0, 150], [80, 60, 120, 100])
    assert catp.tolist() == [20, 40, 40, 90] and atp.tolist() == [20, 20, 0, 50]
    assert promise(50, 0, catp) == (3, 20.0) and promise(120, 0, catp) == (None, 20.0)
    assert allocated_atp(90, [0.5, 0.3, 0.2]).tolist() == [45, 27, 18]
    ps = projected_service(x, d, [0, 0, 1, 0], [0, 0, 0, 0]); assert abs(ps["fill_rate"] - 0.7) < 1e-12 and ps["otif"] == 0.0
    print("self-test passed")
```
**Test vector** (executed by the `__main__` self-test above): supply 700 against demand [500, 300, 150, 50] (Σ 1,000).

**Allocation rules:**

| Rule | Allocation | Margin @ [4, 6, 5, 3] | Projected OTIF | Fill rate |
|---|---|---|---|---|
| Fair share | [350, 210, 105, 35] | 3,290 | 0.00 | 0.70 |
| Priority (tiers [2, 1, 1, 2]) | [227, 300, 150, 23] | 3,527 | 0.50 | 0.70 |
| Margin LP (φ = 0.5) | [250, 300, 125, 25] | 3,500 | 0.25 | 0.70 |

- Total fill rate is identical (0.70) under every rule; OTIF and margin differ. This is why GS-G-08 requires logging outcomes in both terms.
- **Fair-share variants:**
  - Floors [0, 0, 150, 50] → [313, 187, 150, 50] (the 0.5/0.5 tie goes to the lower index).
  - Weights [1, 1, 1, 4] → [342, 205, 103, 50] (customer 4 capped at demand, remainder redistributed).
- **LP without floor:** [250, 300, 150, 0]. Customer 4 gets nothing, which is why the φ floor exists.

**ATP** (on hand 100; receipts [0, 200, 0, 150]; committed [80, 60, 120, 100]):
- CATP = [20, 40, 40, 90]; ATP = [20, 20, 0, 50].
- An order of 50 in period 1 → full promise in period 4, or 20 now.
- An order of 120 → no full promise within the horizon.
- Allocated ATP of 90 at shares 50/30/20 → [45, 27, 18].

```sql
-- GS-G-11 allocation audit: each query must return zero rows
SELECT item_id, location_id, period
FROM allocation a JOIN demand_period d USING (item_id, location_id, period) JOIN supply_period s USING (item_id, location_id, period)
GROUP BY item_id, location_id, period
HAVING ABS(SUM(a.alloc_qty) - LEAST(MAX(s.supply_qty), SUM(d.demand_qty))) > 1e-6;
SELECT a.* FROM allocation a JOIN demand_period d USING (item_id, location_id, period, customer_id)
WHERE a.alloc_qty < 0 OR a.alloc_qty > d.demand_qty;
-- Requested quantity and date must survive allocation (GS-G-05)
SELECT order_id, line_id FROM demand_orders_history WHERE requested_qty IS NULL OR requested_date IS NULL;
```

## 6. Standardized Output Format
```markdown
# Shortage Allocation & ATP — <item family / location> — period <P> — run ALLOC_<run_id>

## Executive Summary
- Shortage: demand <D> vs supply <S> (<gap> units, <gap_value> at net price) across <n_items> items, <n_cust> customers.
- Declared rule: <rule> (approved <MBR date>). Projected fill rate <fr>, OTIF <otif> (<basis>), margin <m>.
- Alternatives: fair-share OTIF <..>/margin <..>; priority <..>/<..>; LP <..>/<..>.
- <k> customers below contract floor/SLA; unconstrained forecast unchanged (hash verified).

## Metadata (mandatory)
| Level | Grain | Time window | Rule | Weighting | OTIF window & date basis | Measurement point | Lag | Release time |
|---|---|---|---|---|---|---|---|---|
| item × location × customer | period / order line | <periods> | fair / priority / LP (φ) | demand / weights | −1/+0 vs requested (and committed) | ship / receipt | n/a | <..> |

## KPI Table — Rule Comparison
| Rule | Allocated units | Fill rate (units) | OTIF (orders) | Margin | Customers below floor | Top-tier OTIF |
|---|---|---|---|---|---|---|

## Allocation Detail
| Customer | Tier | ABC | Requested | Allocated | Fill % | In full? | Promise date | Value at risk |
|---|---|---|---|---|---|---|---|---|

## ATP
| Period | Supply | Committed | CATP | ATP | Allocated ATP by channel |
|---|---|---|---|---|---|

## Exceptions (ranked by value at risk × ABC weight 3/2/1, top N)
| ID | Trigger (rule) | customer / item | metric | value | action | owner |
|---|---|---|---|---|---|---|
| | Rule not declared (GS-G-08) | | | | MBR decision | S&OP leader |
| | Below contract floor / SLA (GS-G-03) | | fill % | | escalate / substitute | Account manager |
| | LP infeasible at φ (GS-G-08) | | Σφd vs S | | lower φ (logged) | Supply planner |
| | Forecast changed by allocation (GS-G-05) | | hash | | revert | Demand manager |

## Recommendations
1. …

## Data-Quality Log
| object | issue (missing requested date, tier unknown → lowest tier, margin missing) | treatment |
|---|---|---|

## Reproducibility
rule_config, supply snapshot, commitments snapshot, forecast version hash (unchanged), code §5.
```

## 7. Edge Cases
- **Demand ≤ supply:** no allocation. Fair share returns demand (tested), and promising runs on CATP only.
- **Floors exceed supply:** floors are scaled pro rata (integer largest remainder) and `FLOORS_EXCEED_SUPPLY` is raised for MBR.
- **Gaming (inflated orders under fair share):** allocate on forecast or history-based shares, not just open orders, for customers whose orders exceed their forecast by > 50% [RoT]. Log it.
- **Multi-period shortages:** allocate period by period on CATP (not per-period supply) so later commitments are protected. Backlog carry-over is an explicit decision per customer contract.
- **Substitution or alternate location:** treated as separate supply with its own allocation, and logged as `SUBSTITUTED` (in full only if the customer accepted it per contract).
- **Small integers** (1–3 units across many customers): the largest-remainder tie rule is deterministic (lower index, i.e. sort customers by priority then ID beforehand) and documented.
- **Re-committed dates:** OTIF is reported against both requested and committed dates. Only the requested basis reflects customer experience.
