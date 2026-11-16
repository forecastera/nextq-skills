---
name: nextq-inventory-health-optimization
description: >-
  [Next Quarter Supply Chain Suite] Diagnoses and optimizes inventory health per SKU-location: computes forward days
  of supply against the forecast, inventory turns and days of inventory at cost, shortage, excess and dead-stock
  status (TSB-based), FEFO expiry at risk and aging buckets, reconciles inventory value to the ledger, recommends
  rebalancing transfers between locations and dispositions in a fixed order (cancel or push out POs, transfer,
  promote, return to vendor, liquidate), and ranks exceptions by ABC-weighted value at risk. Load when the user asks
  about excess, obsolete, slow-moving, aging or expiring inventory, days of supply, inventory turns, working-capital
  reduction, stock rebalancing between DCs or stores, or what to do with dead stock. Inputs: on-hand and on-order at
  cost, lots with expiry, daily forecast, safety-stock and max levels, demand history, transfer lanes, ledger value.
  Triggers: "excess inventory", "DOS", "turns", "dead stock", "rebalance".
metadata:
  author: "Next Quarter"
  organization: "Next Quarter (nextq)"
  suite: "Next Quarter Supply Chain Intelligence"
  version: "1.0.0"
---

# Inventory Health & Optimization
*Part of the **Next Quarter (nextq)** Supply Chain Skill Suite — Governed by SUPPLY_CHAIN_GOLD_STANDARD.md.*

## 1. Role & Objective
**Persona:** Inventory Optimization Manager / Supply Chain Finance Partner.
**Decision supported:** Where is inventory unhealthy (short, excess, aging, expiring, dead), how much value is at stake, and what should we do (cancel or push out, transfer, promote, return, liquidate) in which order?

**Objective:**
- Per SKU-location: forward days of supply, turns at cost, health status, excess and dead-stock units and value, expiry at risk, aging profile.
- A ranked action list (rebalancing transfers and dispositions) with value at risk and owners.
- A reconciled inventory value consistent with finance.

## 2. Gold Standard Rules Enforced
| Rule | Inlined standard |
|---|---|
| **GS-K-11** Forward DOS | Days until cumulative **forecast** demand exhausts on-hand (state whether on-order and in-transit are included). The simple form, on-hand / average daily forecast over the next N days, is allowed only for flat forecasts. Backward (history) DOS hides seasonality: never used for status. Same bucket calendar (GS-D-09). |
| **GS-K-12** Turns | Turns = annualized COGS / **average inventory at cost** (≥ 2 points, preferably the 13 monthly points). Days of inventory = 365 / turns. Never mix COGS with retail-valued inventory. |
| **GS-M-06** TSB for dead stock | TSB: occurrence probability π updated **every** period (π ← π + β(1[y>0] − π)), size z only on demand (z ← z + α(y − z)); forecast π·z. It decays toward zero during long zero runs, so it is **required** for obsolescence and dead-stock decisions (Croston/SBA do not decay). α, β ∈ [0.05, 0.3] [RoT]. |
| **GS-G-03** Exceptions | DOS triggers: forward DOS < SS days (**shortage**) or > max days / shelf life (**excess or obsolescence**). Rank by value at risk = units × unit cost (excess) or units × margin (shortage) × ABC weight (A = 3, B = 2, C = 1). Top N, max 25 per planner [RoT]. |
| **GS-G-06** Value | Inventory valued **at cost**, in finance's currency and fiscal calendar. The skill's total inventory value must reconcile to the ledger; a gap > 2% escalates (the GS-G-06 tolerance). Revenue effects of dispositions (promotion, liquidation) use net price. Volumes are never plugged to match the ledger. |
| **GS-G-09** Segments | ABC (value, A/B/C = 80/15/5% [RoT]) × XYZ × demand class set review frequency and targets. Health thresholds can differ by segment (e.g. C-items' max days lower). |
| **GS-G-11** Verification | Σ status units = Σ position. Transfers never exceed source excess or destination need. Inventory value reconciles to the ledger within 2%. |

**Health parameters [RoT, configurable]:**
- **Shortage:** forward DOS < SS days, where SS days = SS units / average daily forecast (SS from `nextq-safety-stock-optimization`).
- **Excess:** position (on-hand + on-order) > policy max (s + Q or S from `nextq-replenishment-policy-design`), or forward DOS > shelf life. Excess units = position − max.
- **Dead risk:** TSB annual demand < 1% of on-hand (more than about 100 years of cover), or TSB ≈ 0. Status precedence: DEAD_RISK > SHORTAGE > EXCESS > HEALTHY.
- **Aging buckets:** 0–90 / 91–180 / 181–365 / > 365 days since receipt.
- **Expiry at risk:** FEFO consumption of lots against the daily forecast; units unsold before their expiry date.
- **Rebalancing:** transfer only where the unit transfer cost is below the value protected per unit (avoided shortage margin or avoided write-off). Cheapest lane first (greedy).
- **Disposition order:** (1) cancel or push out open POs → (2) transfer → (3) promote (a decision with expected demand effect, routed to `nextq-promotion-lift-cannibalization-planning`; never a forecast edit) → (4) return to vendor → (5) liquidate → (6) write off.

## 3. Required Inputs & Data Schema
| Table | Columns | Notes |
|---|---|---|
| `inventory` | `item_id, location_id, on_hand, on_order, in_transit, unit_cost, snapshot_date` | at cost |
| `lots` | `item_id, location_id, lot_id, qty, receipt_date, expiry_date` | FEFO and aging |
| `forecast_daily` | `item_id, location_id, date, qty` | unconstrained forecast (daily or disaggregated, GS-D-09) |
| `policy` | `item_id, location_id, ss_units, max_units, shelf_life_days` | from safety-stock and replenishment-policy skills |
| `history` | `item_id, location_id, period, demand` | for TSB |
| `cogs_inventory` | `period, cogs, inventory_at_cost` | turns (13 points) |
| `lanes` | `from_loc, to_loc, unit_transfer_cost, lead_time` | rebalancing |
| `item_master` | `item_id, abc, xyz, unit_margin, net_price, returnable, eol_flag` | segments and value |
| `ledger` | `period, inventory_value_gl` | GS-G-06 reconciliation |

**Grain:** SKU-location (status), lot (expiry and aging), segment and total (turns, value).

## 4. Execution Workflow
1. **Validate:**
   - Inventory is at cost.
   - Forecast covers ≥ max(shelf life, 180 days) [RoT], else `HORIZON_EXCEEDED`.
   - Lots sum to on-hand.
   - Reconcile total value to the ledger (> 2% gap escalates).
2. **Forward DOS** for on-hand and for position (on-hand + on-order + in-transit). Report both and label them.
3. **TSB** per SKU-location on history (per-day rate), then dead-risk test.
4. **Classify** status and units (precedence DEAD_RISK > SHORTAGE > EXCESS > HEALTHY). Value: excess and dead at cost; shortage at margin.
5. **Expiry and aging:** FEFO expiry at risk per lot; aging buckets in units and value.
6. **Turns:** by segment and total (13-point average at cost), with days of inventory.
7. **Rebalance:** match excess and dead-risk locations to shortage locations across lanes (greedy, cost < value protected). Lead time must be shorter than the destination's DOS-to-stockout.
8. **Dispositions** for the residual excess, dead and expiring stock, in the declared order:
   - Cancel or push out open POs first (cheapest).
   - Promotion only for items with a positive TSB rate.
   - Return to vendor if `returnable`.
   - Liquidation recovery at net price.
9. **Exceptions:** rank by value at risk × ABC weight; top N.
10. **Output** (Section 6). Policy feedback to the replenishment and safety-stock skills (e.g. repeated excess means max too high or MOQ-driven).

## 5. Verification Snippets
```python
import numpy as np, pandas as pd

ABC_W = {"A": 3, "B": 2, "C": 1}

def forward_dos(stock, daily_fcst):
    """GS-K-11 forward DOS: days until cumulative FORECAST demand exhausts `stock` (state whether stock includes
    on-order/in-transit). Fractional within the exhausting day; returns inf-flag if the horizon does not exhaust it."""
    f = np.asarray(daily_fcst, float); cum = np.cumsum(f)
    idx = np.searchsorted(cum, stock, side="left")
    if idx >= len(f): return float(len(f)), True            # >= horizon (flag HORIZON_EXCEEDED)
    prev = cum[idx - 1] if idx > 0 else 0.0
    return float(idx + (stock - prev) / f[idx]), False

def turns(cogs_annual, inv_points_at_cost):
    """GS-K-12: turns = annualized COGS / average inventory AT COST (>= 2 points, prefer 13 monthly points);
    days of inventory = 365 / turns."""
    pts = np.asarray(inv_points_at_cost, float); assert len(pts) >= 2, "need >= 2 inventory points"
    t = cogs_annual / pts.mean(); return dict(turns=t, days_of_inventory=365 / t, n_points=len(pts))

def tsb(y, alpha=0.1, beta=0.1):
    """GS-M-06 TSB (Teunter-Syntetos-Babai 2011): occurrence prob pi updated every period, size z on demand only;
    forecast = pi * z; decays toward zero in long zero runs -> required for dead-stock decisions."""
    y = np.asarray(y, float); nz = y[y > 0]
    pi = (y > 0).mean() if len(y) else 0.0; z = nz.mean() if len(nz) else 0.0
    for v in y:
        if v > 0: pi += beta * (1 - pi); z += alpha * (v - z)
        else: pi += beta * (0 - pi)
    return pi * z

def classify(on_hand, on_order, daily_fcst_avg, ss_units, max_units, tsb_rate, shelf_days=None, dead_rate=0.01):
    """Health status per SKU-location (GS-G-03 DOS triggers):
    SHORTAGE  : forward DOS < SS days (SS days = SS / daily forecast)
    EXCESS    : on_hand + on_order > max stock (policy max = s+Q or S from nextq-replenishment-policy-design) or DOS > shelf life
    DEAD_RISK : TSB rate per day < dead_rate * (on_hand) / 365 [RoT] i.e. > ~100 years cover, or TSB ~ 0
    HEALTHY   : otherwise. Excess units = position - max."""
    pos = on_hand + on_order
    dos = pos / daily_fcst_avg if daily_fcst_avg > 0 else np.inf
    ss_days = ss_units / daily_fcst_avg if daily_fcst_avg > 0 else 0
    if tsb_rate <= 1e-9 or (on_hand > 0 and tsb_rate * 365 < dead_rate * on_hand): return "DEAD_RISK", pos, dos
    if dos < ss_days: return "SHORTAGE", 0.0, dos
    if pos > max_units or (shelf_days is not None and dos > shelf_days): return "EXCESS", max(pos - max_units, 0.0), dos
    return "HEALTHY", 0.0, dos

def fefo_expiry_at_risk(lots, daily_fcst):
    """FEFO consumption: lots = [(qty, days_to_expiry)], earliest expiry consumed first; units not sold before their
    expiry are at risk (write-off unless actioned)."""
    f = np.asarray(daily_fcst, float); cum = np.r_[0, np.cumsum(f)]; used = 0.0; risk = np.zeros(len(lots))
    for i in sorted(range(len(lots)), key=lambda j: lots[j][1]):
        qty, exp = lots[i]
        avail = cum[min(int(exp), len(f))] - used          # demand available before this lot expires
        sold = max(min(qty, avail), 0.0); used += sold; risk[i] = qty - sold
    return risk                                             # in input lot order

def aging_buckets(receipt_ages_days, qty):
    """Age buckets 0-90 / 91-180 / 181-365 / >365 days [RoT]."""
    b = pd.cut(np.asarray(receipt_ages_days), [-1, 90, 180, 365, np.inf], labels=["0-90", "91-180", "181-365", ">365"])
    return pd.Series(qty, dtype=float).groupby(b, observed=False).sum()

def rebalance(excess, need, unit_cost_transfer, unit_value):
    """Greedy transfers: move excess units to locations with need, cheapest lane first, only where transfer cost per
    unit < value protected per unit (avoided shortage or avoided write-off). Returns list of (from, to, qty)."""
    ex = dict(excess); nd = dict(need); moves = []
    lanes = sorted(((c, f, t) for (f, t), c in unit_cost_transfer.items()), key=lambda x: x[0])
    for c, f, t in lanes:
        if c >= unit_value or ex.get(f, 0) <= 0 or nd.get(t, 0) <= 0: continue
        q = min(ex[f], nd[t]); ex[f] -= q; nd[t] -= q; moves.append((f, t, q))
    return moves

DISPOSITION_ORDER = ["CANCEL_OR_PUSH_OUT_PO", "TRANSFER", "PROMOTE", "RETURN_TO_VENDOR", "LIQUIDATE", "WRITE_OFF"]

def value_at_risk(units, unit_cost, abc):
    """GS-G-03 ranking value for excess/obsolescence: units x unit cost x ABC weight (3/2/1)."""
    return abs(units) * unit_cost * ABC_W[abc]

if __name__ == "__main__":                                  # self-test: reproduces the documented test vector
    assert forward_dos(250, [10] * 10 + [30] * 30) == (15.0, False) and forward_dos(5000, [10] * 90) == (90.0, True)
    t = turns(1_200_000, [300000, 280000, 320000, 310000, 290000, 300000, 305000, 295000, 300000, 310000, 290000, 300000, 300000])
    assert abs(t["turns"] - 4.0) < 1e-9 and abs(t["days_of_inventory"] - 91.25) < 1e-9 and t["n_points"] == 13
    assert abs(tsb([0] * 30 + [5] + [0] * 20) - 0.0612) < 1e-4 and abs(tsb([3, 0, 2, 0, 4, 0, 3, 0, 2, 0]) - 1.3472) < 1e-4
    assert classify(100, 0, 10, 150, 600, 10)[0] == "SHORTAGE"
    assert classify(900, 200, 10, 150, 600, 10)[:2] == ("EXCESS", 500)
    assert classify(300, 0, 10, 150, 600, 10, shelf_days=20)[0] == "EXCESS"
    assert classify(500, 0, 0.1, 5, 100, 0.0005)[:2] == ("DEAD_RISK", 500)
    assert classify(300, 0, 10, 150, 600, 10)[0] == "HEALTHY"
    assert fefo_expiry_at_risk([(100, 5), (200, 30), (150, 10)], [20] * 40).tolist() == [0, 0, 50]
    assert aging_buckets([10, 100, 200, 400, 50], [5, 6, 7, 8, 9]).tolist() == [14, 6, 7, 8]
    mv = rebalance({"DC1": 300, "DC2": 50}, {"DC3": 200, "DC4": 120},
                   {("DC1", "DC3"): 1.0, ("DC1", "DC4"): 2.5, ("DC2", "DC4"): 0.8, ("DC2", "DC3"): 3.0}, 2.0)
    assert mv == [("DC2", "DC4", 50), ("DC1", "DC3", 200)]
    assert value_at_risk(400, 7.5, "B") == 6000.0 and DISPOSITION_ORDER[0] == "CANCEL_OR_PUSH_OUT_PO"
    print("self-test passed")
```
**Test vector** (executed by the `__main__` self-test above):
- **Forward DOS:** stock 250 against forecast 10/day × 10 days then 30/day → **15.0** days. A flat average over 40 days (25/day) would give 10, which shows why the forward form matters. Stock 5,000 → horizon exceeded (flag).
- **Turns:** COGS 1.2M against a 13-point average inventory of 300k at cost → **4.0** turns, **91.25** days of inventory.
- **TSB (α = β = 0.1):** 30 zeros, one 5, 20 zeros → 0.061/period (decayed). An active alternating series → 1.347.

| Case | Inputs | Status |
|---|---|---|
| 1 | on-hand 100, SS 150, forecast 10/day | **SHORTAGE** (DOS 10 < SS days 15) |
| 2 | position 1,100 vs. max 600 | **EXCESS** 500 |
| 3 | DOS 30 > shelf life 20 | **EXCESS** |
| 4 | on-hand 500, TSB 0.0005/day | **DEAD_RISK** 500 |
| 5 | DOS 30 | **HEALTHY** |

- **FEFO:** lots (100 units, expiring day 5), (200, day 30), (150, day 10) at 20/day → at risk [0, 0, **50**]. The day-10 lot can only sell 100 after the first lot.
- **Aging:** units [5, 6, 7, 8, 9] at ages [10, 100, 200, 400, 50] → 0–90: 14, 91–180: 6, 181–365: 7, > 365: 8.
- **Rebalance** (value protected 2.0/unit):
  - DC2→DC4 moves 50 (cost 0.8), DC1→DC3 moves 200 (cost 1.0).
  - DC1→DC4 is rejected (cost 2.5 ≥ 2.0).
- **Value at risk:** 400 excess units × 7.5 cost × B weight 2 = 6,000.

```sql
-- Turns at cost on 13 monthly points (K-12) and ledger reconciliation (G-06)
WITH pts AS (SELECT period, inventory_at_cost FROM cogs_inventory ORDER BY period DESC LIMIT 13)
SELECT (SELECT SUM(cogs) FROM cogs_inventory WHERE period > CURRENT_DATE - INTERVAL '12 months')
       / AVG(inventory_at_cost) AS turns,
       365.0 / ((SELECT SUM(cogs) FROM cogs_inventory WHERE period > CURRENT_DATE - INTERVAL '12 months')
       / AVG(inventory_at_cost)) AS days_of_inventory, COUNT(*) AS n_points
FROM pts;

SELECT l.period, SUM(i.on_hand * i.unit_cost) AS skill_value, MAX(l.inventory_value_gl) AS gl_value,
       ABS(SUM(i.on_hand * i.unit_cost) / MAX(l.inventory_value_gl) - 1) > 0.02 AS escalate
FROM inventory i JOIN ledger l ON l.period = DATE_TRUNC('month', i.snapshot_date)
GROUP BY l.period;
```

## 6. Standardized Output Format
```markdown
# Inventory Health — <scope> — snapshot <date> — run INV_<run_id>

## Executive Summary
- Inventory <value> at cost (ledger gap <g>%); turns <t> (<doi> days); <x>% value healthy.
- Excess <units>/<value>, dead risk <value>, expiring within horizon <value>, shortage at <n> SKU-locations (<margin at risk>).
- Actions: <k> transfers (<value protected>), PO cancels/push-outs <value>, dispositions <value>; top item <..>.

## Metadata (mandatory)
| Level | Grain | Time window / horizon | DOS basis | Valuation | Weighting | Lag | Turns points | Calendar |
|---|---|---|---|---|---|---|---|---|
| SKU-location; segment; total | day / lot | forecast <N> days | forward; on-hand & position (incl. on-order, in-transit) | cost (FIFO/std) | value | forecast as of <date> | 13 | fiscal (GS-D-09) |

## KPI Table
| Segment (ABC/XYZ) | Inventory at cost | Turns | Days of inventory | Median fwd DOS | % SHORTAGE | % EXCESS value | % DEAD_RISK value | Expiry at risk |
|---|---|---|---|---|---|---|---|---|

## Health Detail
| item / location | Status | On-hand | Position | Fwd DOS | SS days | Max | Excess units | TSB rate | Value at risk |
|---|---|---|---|---|---|---|---|---|---|

## Aging & Expiry
| Bucket / lot | Units | Value | Expiry at risk |
|---|---|---|---|

## Actions (in disposition order)
| ID | Type | item / from → to | Units | Cost | Value protected | Owner | Due |
|---|---|---|---|---|---|---|---|

## Exceptions (ranked by value at risk × ABC weight 3/2/1, top N ≤ 25 per planner)
| ID | Trigger (rule) | item / location | metric | value | action | owner |
|---|---|---|---|---|---|---|
| | Fwd DOS < SS days (GS-G-03) | | DOS | | expedite / transfer | Supply planner |
| | Fwd DOS > max / shelf life (GS-G-03) | | DOS | | cancel PO / transfer / promote | Inventory planner |
| | TSB dead risk (GS-M-06) | | rate | | RTV / liquidate | Category / finance |
| | Ledger gap >2% (GS-G-06) | | % | | reconcile | Finance |

## Recommendations
1. …

## Data-Quality Log
| object | issue (lots ≠ on-hand, missing cost, forecast horizon short, expiry missing) | treatment |
|---|---|---|

## Reproducibility
Snapshot ids, policy version, forecast version, thresholds (RoT list §2), code §5.
```

## 7. Edge Cases
- **Seasonal pre-build:** high DOS before the season is not excess if forward DOS uses the seasonal forecast. Classify only on the forward basis; pre-builds approved in `nextq-supply-demand-balancing` carry a `PLANNED_PREBUILD` tag.
- **New products:** fewer than 8 periods of history means TSB is unreliable. Use the lifecycle forecast (`nextq-new-product-lifecycle-forecasting`) and suppress DEAD_RISK for the launch window.
- **End-of-life (`eol_flag`):** set max to 0. All position is excess and goes to disposition, starting with cancelling POs.
- **Negative or zero on-hand with open backorders:** status SHORTAGE. DOS = 0. Not included in turns averages.
- **Consignment / VMI stock:** excluded from turns and value unless owned; reported separately.
- **Transfers that create a shortage at the source:** the source keeps at least max(SS, the lead-time demand of the next replenishment); only the true excess (position − max) is transferable.
- **Substitutable items:** dead risk of an item with a live successor goes to a substitution or promotion decision before liquidation.
