---
name: nextq-service-level-performance-analysis
description: >-
  [Next Quarter Supply Chain Suite] Measures and root-causes delivered service: computes cycle service level, unit,
  line and order fill rate and OTIF with Wilson confidence intervals and explicit date basis and window, uses
  reconstructed unconstrained demand as the fill-rate denominator, tests actual stockouts against the planned CSL with
  a binomial test, attributes every stockout cycle to execution, supply, forecast or planned risk, and ranks
  persistent service gaps by ABC-weighted value at risk. Load when the user asks why service or fill rate dropped, to
  report OTIF, CSL or fill rate, to compare planned vs actual service, to attribute stockouts to forecast vs supply vs
  execution, or to find items persistently below target. Inputs: order lines with requested and delivered qty and
  dates, replenishment cycles, targets, margins. Triggers: "OTIF", "fill rate", "service level", "stockout root
  cause", "CSL".
metadata:
  author: "Next Quarter"
  organization: "Next Quarter (nextq)"
  suite: "Next Quarter Supply Chain Intelligence"
  version: "1.0.0"
---

# Service-Level Performance Analysis
*Part of the **Next Quarter (nextq)** Supply Chain Skill Suite — Governed by SUPPLY_CHAIN_GOLD_STANDARD.md.*

## 1. Role & Objective
**Persona:** Customer Service Performance Analyst / Inventory Planning Manager.
**Decision supported:** Did we deliver the service we planned, and if not, was it the forecast, supply, execution, or the risk we chose to accept? Where should corrective action go first?

**Objective:**
- Measure, with confidence intervals and explicit bases:
  - cycle service level (CSL)
  - unit, line and order fill rate
  - OTIF
- Compare against planned targets (statistical test, not eyeballing).
- Attribute every stockout cycle to a root cause with a fixed precedence.
- Rank persistent gaps by value at risk.
- Feed corrections to `nextq-safety-stock-optimization`, `nextq-replenishment-policy-design`, `nextq-forecast-accuracy-diagnostics` and supply or execution owners.

## 2. Gold Standard Rules Enforced
| Rule | Inlined standard |
|---|---|
| **GS-K-08** CSL | CSL = # replenishment cycles with no stockout / # cycles. Planned CSL = Φ(k). It ignores shortage size and is **not a customer metric**: always say "cycle service level", never just "service level". **CSL is never equated with fill rate.** |
| **GS-K-09** Fill rate | Unit fill rate = units shipped from stock on first pass / units **requested** (unconstrained). Line and order fill are analogous. State the basis. **Backfills do not count as filled.** |
| **GS-K-10** OTIF | OTIF = # orders (or lines) on time **AND** in full / # orders (or lines). On time is measured against the requested or committed date (state which) within a window, default −1/+0 days [RoT]. Early deliveries outside the window are not on time. In full means delivered ≥ ordered. Both must hold on the same order. State the measurement point (ship vs. receipt). |
| **GS-M-11** Censoring | Where requested quantity is not recorded (retail and stores), the fill-rate denominator is **reconstructed unconstrained demand**: (a) orders, (b) availability scaling $\hat D=S/a$ (uplift capped ×2), (c) model-based, or (d) imputed. Taken from `nextq-unconstrained-demand-reconstruction`. Report % of the denominator reconstructed. > 30% → `LOW_CONFIDENCE` [RoT]. |
| **GS-K-13** Reporting | Report the decision level (SKU-location-cycle) and aggregates **separately**. Aggregate fill rate is volume-weighted (Σ filled / Σ requested), not the mean of item rates. Report N items, N cycles or orders, % reconstructed. |
| **GS-G-03** Exceptions | Trigger [RoT]: fill rate below target − 2 pp for 3 consecutive cycles, or a binomial test showing CSL significantly below plan. Value at risk = units short × unit margin × ABC weight (A = 3, B = 2, C = 1). Top N, max 25 per planner [RoT]. |
| **GS-G-09** Segments | Targets by ABC/XYZ segment. Defaults (from the suite's safety-stock policy) A/B/C = 98/95/90% CSL [RoT]. Fill-rate targets are set separately and never derived by equating them with CSL. |

**Statistics:**
- **Wilson interval** for each proportion: $\frac{\hat p+\frac{z^2}{2n}\pm z\sqrt{\frac{\hat p(1-\hat p)}{n}+\frac{z^2}{4n^2}}}{1+\frac{z^2}{n}}$, z = 1.96.
- **Planned vs. actual:** one-sided binomial test $P(X\ge x\mid n,\ 1-\text{CSL}_{target})$. p < 0.05 [RoT] means `BELOW_PLAN`. With n < 20 cycles, report the observation only (`INSUFFICIENT_CYCLES`).

**Root-cause precedence** (a stockout gets exactly one cause):

| Cause | Test | Owner / route |
|---|---|---|
| EXECUTION | stock was available at the location or network when demand arrived (pick, allocation, master data, system) | warehouse / customer service |
| SUPPLY | actual lead time > planned, or received < ordered | procurement / supplier; lead-time σ to `nextq-safety-stock-optimization` |
| FORECAST | actual lead-time demand − forecast > kσ_DL | `nextq-forecast-accuracy-diagnostics` / `nextq-demand-exception-management` |
| PLANNED_RISK | none of the above: within the accepted 1 − CSL risk | no action, unless the frequency exceeds plan (binomial test) |

## 3. Required Inputs & Data Schema
| Table | Columns | Notes |
|---|---|---|
| `order_lines` | `order_id, line_id, customer_id, item_id, location_id, ordered_qty, requested_qty, first_pass_qty, delivered_qty, requested_date, committed_date, delivered_date, ship_date, reconstructed (bool)` | requested_qty = unconstrained |
| `cycles` | `item_id, location_id, cycle_id, start_date, end_date, stockout (bool), stock_available (bool), lt_planned, lt_actual, ordered_qty, received_qty, ltd_forecast, ltd_actual, sigma_dl, k` | replenishment cycles (receipt to receipt) |
| `targets` | `item_id, location_id, csl_target, fill_target, otif_target, segment (ABC/XYZ)` | from safety-stock or policy skills |
| `economics` | `item_id, unit_margin, unit_cost, abc_class` | value at risk |
| `config` | `otif_window, date_basis, measurement_point, gap_pp, runs, alpha` | declared |

**Grain:** SKU-location-cycle (CSL), order line (fill rate, OTIF). Aggregated by segment, customer and region.

## 4. Execution Workflow
1. **Validate:**
   - Requested qty and dates are present; reconstructed share is computed.
   - Delivered ≤ shipped.
   - Cycles are contiguous.
   - Backfill lines are excluded from first pass.
2. **Compute KPIs** at the decision level: CSL per SKU-location (K-08), unit / line / order fill rate (K-09), OTIF on the requested **and** committed bases (K-10). All with Wilson CIs and N.
3. **Aggregate** (K-13): volume-weighted by segment, customer, region and total. Reported separately from the decision level.
4. **Planned vs. actual:** binomial test per SKU-location (n ≥ 20 cycles), then segment rollups.
5. **Root-cause** each stockout cycle with the precedence above. Summarize causes in counts and units short, by segment.
6. **Persistent gaps** (G-03): fill rate below target − 2 pp for 3 consecutive cycles, or `BELOW_PLAN`. Rank by value at risk × ABC weight; keep the top N.
7. **Diagnose the policy:**
   - PLANNED_RISK frequency above plan → SS or σ input wrong → `nextq-safety-stock-optimization` (check σ uses forecast error, not demand σ).
   - High CSL but low fill rate → order quantity or review period → `nextq-replenishment-policy-design`.
   - FORECAST-dominant → accuracy and bias review.
8. **Recommend** by owner, with expected service gain where quantifiable.
9. **Output** (Section 6).

## 5. Verification Snippets
```python
import numpy as np, pandas as pd
from scipy import stats

ABC_W = {"A": 3, "B": 2, "C": 1}

def wilson(k, n, z=1.96):
    """Wilson score interval for a proportion (CSL, fill rate by lines/orders, OTIF): robust for small n and p near 1."""
    if n == 0: return (np.nan, np.nan)
    p = k / n; den = 1 + z * z / n; c = (p + z * z / (2 * n)) / den
    h = z * np.sqrt(p * (1 - p) / n + z * z / (4 * n * n)) / den
    return (max(0.0, c - h), min(1.0, c + h))

def csl(cycles):
    """GS-K-08: CSL = cycles with no stockout / cycles. cycles: DataFrame with bool column `stockout`."""
    n = len(cycles); k = int((~cycles["stockout"]).sum())
    return dict(csl=k / n if n else np.nan, n_cycles=n, ci=wilson(k, n))

def fill_rates(lines):
    """GS-K-09 on order lines with requested_qty (unconstrained; reconstructed if censored, GS-M-11) and
    first_pass_qty (shipped from stock on first pass; backfills excluded). Unit, line, order bases."""
    l = lines.assign(full=lines["first_pass_qty"] >= lines["requested_qty"])
    unit = l["first_pass_qty"].clip(upper=l["requested_qty"]).sum() / l["requested_qty"].sum()
    line = l["full"].mean(); order = l.groupby("order_id")["full"].all().mean()
    return dict(unit_fill=float(unit), line_fill=float(line), order_fill=float(order),
                pct_reconstructed=float(l.get("reconstructed", pd.Series(False, index=l.index)).mean()))

def otif(lines, window=(-1, 0), basis="requested_date", level="order"):
    """GS-K-10: on time = delivered_date - <basis> in [window] days [RoT -1/+0]; in full = delivered_qty >= ordered_qty;
    both on the SAME order (or line). Basis must be stated (requested vs committed)."""
    d = (pd.to_datetime(lines["delivered_date"]) - pd.to_datetime(lines[basis])).dt.days
    l = lines.assign(on_time=(d >= window[0]) & (d <= window[1]), in_full=lines["delivered_qty"] >= lines["ordered_qty"])
    g = l.groupby("order_id")[["on_time", "in_full"]].all() if level == "order" else l[["on_time", "in_full"]]
    k = int((g["on_time"] & g["in_full"]).sum()); n = len(g)
    return dict(otif=k / n, on_time=float(g["on_time"].mean()), in_full=float(g["in_full"].mean()), n=n, ci=wilson(k, n), basis=basis)

def planned_vs_actual(n_cycles, n_stockouts, target_csl):
    """Is the observed stockout count consistent with the planned CSL? One-sided binomial test
    P(X >= observed | p = 1 - target). p < 0.05 -> service below plan beyond chance [RoT alpha]."""
    p = stats.binom.sf(n_stockouts - 1, n_cycles, 1 - target_csl)
    return dict(expected_stockouts=n_cycles * (1 - target_csl), observed=n_stockouts, p_value=float(p), below_plan=bool(p < 0.05))

def root_cause(cyc, k):
    """Attribute each stockout cycle (precedence: EXECUTION > SUPPLY > FORECAST > PLANNED_RISK):
    EXECUTION   - stock was available in the network/location at the time (pick, allocation, data or ship error)
    SUPPLY      - receipt late (actual lead time > planned) or short (received < ordered)
    FORECAST    - lead-time demand error exceeded k*sigma_DL (actual LTD - forecast LTD > k sigma_DL)
    PLANNED_RISK- none of the above: the stockout is within the risk the CSL target accepts (1 - CSL)."""
    out = []
    for r in cyc.itertuples():
        if not r.stockout: out.append(None); continue
        if r.stock_available: out.append("EXECUTION")
        elif r.lt_actual > r.lt_planned or r.received_qty < r.ordered_qty: out.append("SUPPLY")
        elif r.ltd_actual - r.ltd_forecast > k * r.sigma_dl: out.append("FORECAST")
        else: out.append("PLANNED_RISK")
    return cyc.assign(cause=out)

def persistent_gap(series, target, gap_pp=2.0, runs=3):
    """GS-G-03 trigger: fill rate below target - 2 pp for 3 consecutive cycles [RoT]. Returns True at each breach index."""
    below = np.asarray(series) < target - gap_pp / 100; run = 0; flags = []
    for b in below: run = run + 1 if b else 0; flags.append(run >= runs)
    return np.array(flags)

def value_at_risk(short_units, unit_margin, abc):
    """GS-G-03 ranking value: |units short| x unit margin x ABC weight (3/2/1)."""
    return abs(short_units) * unit_margin * ABC_W[abc]

if __name__ == "__main__":                                  # self-test: reproduces the documented test vector
    lo, hi = wilson(48, 50); assert abs(lo - 0.8654) < 1e-4 and abs(hi - 0.9890) < 1e-4
    r = csl(pd.DataFrame(dict(stockout=[False] * 45 + [True] * 5))); assert r["csl"] == 0.9 and abs(r["ci"][0] - 0.7864) < 1e-4
    lines = pd.DataFrame(dict(order_id=[1, 1, 2, 3, 3, 4], requested_qty=[10, 5, 8, 4, 6, 10],
                              first_pass_qty=[10, 3, 8, 4, 6, 0], reconstructed=[0, 0, 0, 0, 0, 1]))
    f = fill_rates(lines); assert abs(f["unit_fill"] - 31 / 43) < 1e-12 and abs(f["line_fill"] - 4 / 6) < 1e-12 and f["order_fill"] == 0.5
    ol = pd.DataFrame(dict(order_id=[1, 1, 2, 3, 4], requested_date=["2026-01-10"] * 5,
                           committed_date=["2026-01-10", "2026-01-10", "2026-01-12", "2026-01-10", "2026-01-10"],
                           delivered_date=["2026-01-10", "2026-01-09", "2026-01-12", "2026-01-08", "2026-01-10"],
                           ordered_qty=[5] * 5, delivered_qty=[5, 5, 5, 5, 4]))
    assert otif(ol)["otif"] == 0.25 and otif(ol, basis="committed_date")["otif"] == 0.5 and otif(ol, level="line")["otif"] == 0.4
    pa = planned_vs_actual(52, 6, 0.95); assert abs(pa["p_value"] - 0.0445) < 1e-4 and pa["below_plan"]
    assert not planned_vs_actual(52, 4, 0.95)["below_plan"]
    c = pd.DataFrame(dict(stockout=[True] * 4 + [False], stock_available=[True] + [False] * 4, lt_actual=[2, 4, 2, 2, 2],
                          lt_planned=[2] * 5, received_qty=[100] * 5, ordered_qty=[100] * 5,
                          ltd_actual=[210, 220, 300, 230, 200], ltd_forecast=[200] * 5, sigma_dl=[28.28] * 5))
    causes = root_cause(c, 1.645)["cause"].tolist()
    assert causes[:4] == ["EXECUTION", "SUPPLY", "FORECAST", "PLANNED_RISK"] and pd.isna(causes[4])
    assert persistent_gap([0.97, 0.92, 0.92, 0.92, 0.96, 0.92], 0.95).tolist() == [False, False, False, True, False, False]
    assert value_at_risk(-40, 2.5, "A") == 300.0
    print("self-test passed")
```
**Test vector** (executed by the `__main__` self-test above):

| Test | Input | Result |
|---|---|---|
| Wilson interval | 48/50 | [0.8654, 0.9890] |
| CSL | 45 of 50 cycles without stockout | **0.90**, CI [0.786, 0.957] |
| Fill rate | 6 lines, 4 orders; requested Σ 43, first pass Σ 31 | unit **0.721**, line 0.667, order 0.50 (all different bases) |
| OTIF, requested-date basis | 4 orders (an early −2 day delivery fails the −1/+0 window; one order short) | **0.25** |
| OTIF, committed-date basis | same orders | **0.50** |
| OTIF, line level | same orders | 0.40 |
| Planned vs. actual | 52 cycles at 95% target (expected 2.6 stockouts) | 6 observed → p = 0.0445, `BELOW_PLAN`; 4 observed → not significant |
| Root cause | one stockout cycle per cause | EXECUTION, SUPPLY (LT 4 vs. 2), FORECAST (error 100 > 1.645 × 28.28 = 46.5), PLANNED_RISK (error 30) |
| Persistent gap | fill [0.97, 0.92, 0.92, 0.92, 0.96, 0.92] vs. 0.95 target | trigger in cycle 4 only |
| Value at risk | 40 units × 2.5 margin × A weight 3 | 300 |

```sql
-- Volume-weighted unit fill rate and OTIF by segment (K-09, K-10, K-13); backfills excluded upstream
SELECT t.segment,
       SUM(LEAST(l.first_pass_qty, l.requested_qty)) * 1.0 / SUM(l.requested_qty)        AS unit_fill_rate,
       AVG(CASE WHEN l.reconstructed THEN 1.0 ELSE 0 END)                                AS pct_reconstructed,
       COUNT(DISTINCT l.order_id)                                                        AS n_orders
FROM order_lines l JOIN targets t USING (item_id, location_id)
GROUP BY t.segment;

SELECT order_id,
       BOOL_AND(delivered_date - requested_date BETWEEN -1 AND 0) AS on_time_requested,
       BOOL_AND(delivered_qty >= ordered_qty)                      AS in_full
FROM order_lines GROUP BY order_id;
```

## 6. Standardized Output Format
```markdown
# Service-Level Performance — <scope> — <period> — run SVC_<run_id>

## Executive Summary
- Unit fill rate <fr> (target <t>, CI <..>); OTIF <otif> vs requested date (<otif_c> vs committed); CSL <csl> (planned <p>).
- <n> SKU-locations BELOW_PLAN; stockout causes: execution <e>%, supply <s>%, forecast <f>%, planned risk <r>%.
- Top value at risk: <items> (<value>); key action: <..>.
- <x>% of fill-rate denominator reconstructed (GS-M-11).

## Metadata (mandatory)
| Level | Grain | Time window | OTIF window & basis | Measurement point | Weighting | Lag | % reconstructed | N items / cycles / orders |
|---|---|---|---|---|---|---|---|---|
| SKU-location; segment; total | cycle / order line | <dates> | −1/+0 days; requested & committed | ship / receipt | volume (Σ/Σ) | forecast lag L for root cause | <..> | <..> |

## KPI Table — Decision Level vs Aggregate (reported separately)
| Level | Segment | CSL (CI) | CSL target | Unit fill (CI) | Line fill | Order fill | OTIF req (CI) | OTIF commit | N |
|---|---|---|---|---|---|---|---|---|---|

## Planned vs Actual
| SKU-location | Cycles | Expected stockouts | Observed | p-value | Status |
|---|---|---|---|---|---|

## Root-Cause Attribution
| Cause | Stockout cycles | % | Units short | Value | Route to |
|---|---|---|---|---|---|

## Exceptions (ranked by value at risk × ABC weight 3/2/1, top N ≤ 25 per planner)
| ID | Trigger (rule) | item / location | metric | value | cause | action | owner |
|---|---|---|---|---|---|---|---|
| | Fill < target − 2pp × 3 cycles (GS-G-03) | | fill | | | | |
| | CSL below plan, p<0.05 (GS-K-08) | | CSL | | | SS review | Inventory planner |
| | >30% reconstructed denominator (GS-M-11) | | % | | | low confidence | Analyst |

## Recommendations
1. …

## Data-Quality Log
| object | issue (missing requested date → committed basis only, backfills not flagged, cycles <20) | treatment |
|---|---|---|

## Reproducibility
config (window, basis, gap_pp, runs, α), target snapshot, reconstruction version, code §5.
```

## 7. Edge Cases
- **Only committed dates available:** report OTIF on the committed basis, flagged `COMMITTED_BASIS_ONLY`. This overstates customer experience when promises were re-dated after shortages.
- **Lost sales (retail):** fill rate uses the reconstructed denominator. If availability is < 50% of the period, use method (c) or (d) and flag. Never compute fill rate on recorded sales (it would be about 100% by construction).
- **Partial deliveries accepted by contract** (tolerance, e.g. ≥ 98% counts as in full): apply only if declared in `config`, and report the tolerance.
- **Very high targets with few cycles** (e.g. 99% CSL with 12 cycles): the CI spans tens of points. Use `INSUFFICIENT_CYCLES` and evaluate on pooled segment cycles.
- **Multiple stockout causes in one cycle:** the precedence assigns one cause. Secondary causes are listed in a notes column (no double counting of units).
- **Substitutions:** counted as filled only if the customer accepted them (per contract); otherwise not in full.
- **Intermittent items:** CSL per cycle is the right lens. Fill rate can swing heavily on single orders, so show N.
