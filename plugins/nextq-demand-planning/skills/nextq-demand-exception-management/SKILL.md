---
name: nextq-demand-exception-management
description: >-
  [Next Quarter Supply Chain Suite] Runs management-by-exception for demand planners: evaluates the standard triggers
  (tracking signal, persistent bias, MASE > 1, negative FVA, frozen-zone swings, censored history, forward
  days-of-supply shortage or excess, interval miscalibration, unreconciled hierarchy), converts hits into units and
  value at risk with ABC weights, consolidates them into one case per item-location, applies hysteresis, aging and
  snooze rules, and publishes a capped, ranked queue per planner with root cause, action and owner. Load when the user
  asks which forecasts or items need attention, to build or prioritize an exception list or planner worklist, triage
  forecast alerts, reduce alert noise, or explain why an item was flagged. Inputs: accuracy and FVA metrics by lag,
  forecast versions, inventory projections, margins, segments, planner assignments. Triggers: "exceptions", "alerts",
  "planner worklist", "which items to review", "value at risk".
metadata:
  author: "Next Quarter"
  organization: "Next Quarter (nextq)"
  suite: "Next Quarter Supply Chain Intelligence"
  version: "1.0.0"
---

# Demand Exception Management
*Part of the **Next Quarter (nextq)** Supply Chain Skill Suite — Governed by SUPPLY_CHAIN_GOLD_STANDARD.md.*

## 1. Role & Objective
**Persona:** Demand Planning Manager / Planning Excellence Lead.
**Decision supported:** Which item-locations should each planner work on this cycle, in what order, and what should they do about each?

**Objective:** A deterministic, de-duplicated, value-ranked exception queue that:
- uses the exact GS trigger definitions,
- never floods planners (capped top-N),
- carries cause, impact (units and value), recommended action and owner for every case,
- tracks the case lifecycle (open, aging, snoozed, closed) with hysteresis so alerts do not flap.

## 2. Gold Standard Rules Enforced
| Rule | Inlined standard |
|---|---|
| **GS-G-03** Trigger set [RoT, configurable] | (1) $\lvert TS\rvert>4$. (2) $\lvert\text{Bias\%}\rvert>10\%$ at decision level for the last 3 cycles (15% for intermittent / lumpy). (3) MASE > 1.0 on the rolling window. (4) FVA of final vs. naïve < 0. (5) Forecast change vs. last cycle > ±25% inside the frozen horizon. (6) Censored-history share > 30%. (7) Forward DOS < safety-stock days (shortage) or > max days / shelf life (excess). (8) Prediction-interval coverage off nominal by > 5 pp. (9) Unreconciled hierarchy delta > 0.5% at any node. Each exception lists **cause, impact (units and value), recommended action**. **Prioritize by value at risk** = \|units at risk\| × unit margin (or cost) × ABC/XYZ weight; show the top N only. |
| **GS-K-06** Tracking signal | $TS=\sum(A_i-F_i)/\text{MAD}$ over a rolling window (13 weeks / 6 months [RoT]). **Positive = under-forecast.** Same function as `nextq-forecast-accuracy-diagnostics`. |
| **GS-K-05** Bias | $\text{Bias\%}=\sum(F-A)/\sum A\times100$, **positive = over-forecast** (opposite sign to TS: state the direction in words). |
| **GS-K-04** MASE | $q_j=e_j/\big(\frac{1}{T-m}\sum_{t=m+1}^{T}\lvert y_t-y_{t-m}\rvert\big)$, $\text{MASE}=\text{mean}\lvert q_j\rvert$, scaled by the training in-sample (seasonal) naïve MAE. > 1 = worse than in-sample naïve. Scale ≈ 0 → `SCALE_ZERO`, fall back to WAPE. |
| **GS-K-07** FVA | $\text{FVA}=\text{Err}(\text{naïve})-\text{Err}(\text{final})$ in WAPE points, same items and lag. < 0 = the process destroys value. |
| **GS-G-09** Segmentation | ABC (value) × XYZ (forecastability) sets review frequency (A weekly, B biweekly, C monthly [RoT]) and weights. **ABC weights A = 3, B = 2, C = 1 [RoT]** (identical to the accuracy skill). |
| **GS-G-10** Output contract | Executive summary, KPI table with lag/level/window/N, exceptions table (ID, trigger with GS rule, item/location, units, value, action, owner), recommendations, data-quality log, reproducibility block. |

**Units at risk per trigger** (documented convention; $F_{LT}$ = forecast units over the next lead time):

| Trigger | Units at risk | Valued at |
|---|---|---|
| TS_OUT_OF_CONTROL | $\lvert\sum_{\text{window}}(F-A)\rvert$ | margin |
| BIAS_PERSISTENT | $\lvert\text{Bias\%}\rvert/100\cdot F_{LT}$ | margin |
| MASE_GT_1 | $(\text{MAE}_{\text{model}}-\text{MAE}_{\text{naïve}})^+\cdot$ LT periods | margin |
| FVA_NEGATIVE | $\lvert\text{FVA}\rvert/100\cdot F_{LT}$ | margin |
| FROZEN_SWING | $\lvert\Delta\%\rvert/100\cdot F_{\text{last, frozen}}$ | margin |
| CENSORED_HISTORY | recent lost units ($\hat D-S$) from reconstruction | margin |
| SHORTAGE_DOS | $(\text{SS days}-\text{DOS})\cdot$ daily forecast | margin |
| EXCESS_DOS | $(\text{DOS}-\text{max days})\cdot$ daily forecast | **cost** |
| PI_MISCALIBRATED | $\lvert\text{coverage gap pp}\rvert/100\cdot F_{LT}$ | margin |
| UNRECONCILED | $\lvert\text{parent}-\sum\text{children}\rvert$ | margin |

**Lifecycle rules [RoT]:**
- **One case per item-location.** The primary trigger is the one with the highest VaR; the others are listed as secondary.
- **Ranking uses primary VaR**, not the sum. Triggers overlap (TS and bias describe the same error), so summing would double count.
- **Queue cap:** top 25 per planner per cycle.
- **Hysteresis:** open at |TS| > 4; close only after |TS| < 3 for 2 consecutive cycles. Other triggers close after 2 clear cycles.
- **Snooze:** allowed for at most 3 cycles, with a reason code; it auto-reopens if VaR doubles.
- **Aging:** cases open for more than 4 cycles escalate to the planning manager.

## 3. Required Inputs & Data Schema
**`exception_metrics`** (one row per item × location at the decision level and decision lag; produced by `nextq-forecast-accuracy-diagnostics`, `nextq-forecast-value-added-analysis`, `nextq-unconstrained-demand-reconstruction`, `nextq-hierarchical-forecast-reconciliation` and the inventory projection):

| Column | Type | Meaning |
|---|---|---|
| `item_id, location_id, planner` | string | keys and owner |
| `abc, xyz, demand_class` | enum | segments (GS-G-09, GS-M-03) |
| `ts, sum_f_minus_a_window` | float | TS on the rolling window; Σ(F−A) over the same window |
| `bias_pct, bias_cycles_outside` | float, int | Bias% (+ = over) and consecutive cycles outside the limit |
| `mase, mae_model, mae_naive, lt_periods` | float, int | MASE and its parts; lead time in periods |
| `fva_final_vs_naive` | float | WAPE points |
| `in_frozen, change_vs_last_pct, fcst_last_frozen` | bool, float | frozen-zone swing inputs |
| `pct_reconstructed, lost_units_recent` | float | from reconstruction |
| `dos_fwd, ss_days, max_days, daily_fcst` | float | forward DOS (GS-K-11) and limits |
| `coverage_gap_pp` | float | empirical − nominal coverage |
| `unreconciled_pct, unreconciled_units` | float | from reconciliation |
| `fcst_next_lt, unit_margin, unit_cost` | float | value conversion |
| `metric_lag, window, snapshot_id` | meta | provenance |

**`case_history`**: `case_id, item_id, location_id, primary, opened_cycle, status ∈ {OPEN, SNOOZED, CLOSED}, snooze_until, snooze_reason, clear_cycles, last_var`.

## 4. Execution Workflow
1. **Validate inputs.**
   - All metrics must be at the same decision lag and snapshot. Rows with a different lag are rejected (`LAG_MISMATCH`).
   - A NaN metric skips only its trigger and is logged.
2. **Evaluate triggers** per item-location with the thresholds above. The bias limit is 15% for INTERMITTENT/LUMPY and 10% otherwise.
3. **Quantify** units at risk and VaR per hit. VaR = units × (margin, or cost for excess) × ABC weight.
4. **Consolidate** into one case per item-location: primary = max VaR, secondary = the rest.
5. **Root-cause hint** from the trigger pattern (deterministic map):

   | Pattern | Root-cause hint |
   |---|---|
   | TS + BIAS, same direction | `SYSTEMATIC_BIAS` (model level or override optimism) |
   | MASE_GT_1 without bias | `MODEL_MISFIT` / noise |
   | FVA_NEGATIVE | `PROCESS_TOUCHPOINT` (see FVA retention decisions) |
   | CENSORED_HISTORY | `AVAILABILITY` |
   | FROZEN_SWING | `LATE_CHANGE` |
   | SHORTAGE/EXCESS_DOS with no forecast trigger | `SUPPLY_OR_POLICY` |
   | PI_MISCALIBRATED | `UNCERTAINTY_MODEL` |
   | UNRECONCILED | `HIERARCHY_PROCESS` |

6. **Recommended action and owner** (deterministic map):

   | Root cause | Recommended action | Owner |
   |---|---|---|
   | SYSTEMATIC_BIAS | re-level the model / review overrides | Demand planner |
   | MODEL_MISFIT | re-select the model via `nextq-statistical-baseline-forecasting` or `nextq-intermittent-demand-forecasting` | Demand planner |
   | PROCESS_TOUCHPOINT | apply WATCH/SUSPEND | Demand manager |
   | AVAILABILITY | run `nextq-unconstrained-demand-reconstruction` and review allocation | Demand planner + supply |
   | LATE_CHANGE | approve or revert per time fence | S&OP leader |
   | SUPPLY_OR_POLICY | route to `nextq-safety-stock-optimization` / `nextq-supply-demand-balancing` | Supply planner |
   | UNCERTAINTY_MODEL | recalibrate intervals | Demand planner |
   | HIERARCHY_PROCESS | re-run reconciliation | Demand planner |

7. **Lifecycle update** against `case_history`:
   - Apply hysteresis, snooze expiry, auto-reopen (VaR ≥ 2× the snoozed VaR), and aging escalation (> 4 cycles).
   - New case IDs are deterministic: `EXC-<cycle>-<item>-<loc>`.
8. **Rank and cap:**
   - Sort by planner, then primary VaR (descending), then item_id (tie-break).
   - Keep the top 25 per planner. Report the count and VaR of cases below the cap as `BELOW_CAP` (not hidden; summarized).
9. **KPIs of the exception process itself:**
   - cases opened, closed, aged
   - share of total VaR covered by the queue
   - median days to close
   - repeat rate (reopened within 4 cycles)
10. **Output** (Section 6).

## 5. Verification Snippets
```python
import numpy as np, pandas as pd

ABC_W = {"A": 3, "B": 2, "C": 1}          # [RoT], same weights as nextq-forecast-accuracy-diagnostics

def triggers(r):
    """r: one item-location row of metrics (decision level, decision lag). Returns list of (code, rule, units_at_risk)."""
    out = []
    lt_f = r.fcst_next_lt                                   # forecast units over the next lead time (frozen horizon)
    bias_lim = 15.0 if r.demand_class in ("INTERMITTENT", "LUMPY") else 10.0
    if np.isfinite(r.ts) and abs(r.ts) > 4:
        out.append(("TS_OUT_OF_CONTROL", "GS-K-06", abs(r.sum_f_minus_a_window)))
    if r.bias_cycles_outside >= 3 and abs(r.bias_pct) > bias_lim:
        out.append(("BIAS_PERSISTENT", "GS-K-05", abs(r.bias_pct) / 100 * lt_f))
    if np.isfinite(r.mase) and r.mase > 1.0:
        out.append(("MASE_GT_1", "GS-K-04", max(r.mae_model - r.mae_naive, 0) * r.lt_periods))
    if np.isfinite(r.fva_final_vs_naive) and r.fva_final_vs_naive < 0:
        out.append(("FVA_NEGATIVE", "GS-K-07", abs(r.fva_final_vs_naive) / 100 * lt_f))
    if r.in_frozen and abs(r.change_vs_last_pct) > 25:
        out.append(("FROZEN_SWING", "GS-G-04", abs(r.change_vs_last_pct) / 100 * r.fcst_last_frozen))
    if r.pct_reconstructed > 0.30:
        out.append(("CENSORED_HISTORY", "GS-M-11", r.lost_units_recent))
    if r.dos_fwd < r.ss_days:
        out.append(("SHORTAGE_DOS", "GS-K-11", (r.ss_days - r.dos_fwd) * r.daily_fcst))
    if r.dos_fwd > r.max_days:
        out.append(("EXCESS_DOS", "GS-K-11", (r.dos_fwd - r.max_days) * r.daily_fcst))
    if np.isfinite(r.coverage_gap_pp) and abs(r.coverage_gap_pp) > 5:
        out.append(("PI_MISCALIBRATED", "GS-K-14", abs(r.coverage_gap_pp) / 100 * lt_f))
    if r.unreconciled_pct > 0.5:
        out.append(("UNRECONCILED", "GS-D-07", r.unreconciled_units))
    return out

def value_at_risk(units, r, code):
    """VaR = |units at risk| x unit value x ABC weight. Excess valued at cost; others at unit margin."""
    unit_val = r.unit_cost if code == "EXCESS_DOS" else r.unit_margin
    return abs(units) * unit_val * ABC_W.get(r.abc, 1)

def build_cases(metrics, top_n=25):
    """One case per item-location; primary trigger = highest VaR. Rank by PRIMARY VaR (triggers overlap, e.g. TS and
    bias describe the same error, so summing would double count); top-N per planner."""
    rows = []
    for _, r in metrics.iterrows():
        trig = [(c, rule, u, value_at_risk(u, r, c)) for c, rule, u in triggers(r)]
        if not trig:
            continue
        trig.sort(key=lambda t: -t[3])
        rows.append(dict(item_id=r.item_id, location_id=r.location_id, planner=r.planner,
                         primary=trig[0][0], rule=trig[0][1], units_at_risk=trig[0][2],
                         var_primary=trig[0][3], secondary=",".join(t[0] for t in trig[1:])))
    cases = pd.DataFrame(rows)
    if cases.empty:
        return cases
    cases = cases.sort_values(["planner", "var_primary", "item_id"], ascending=[True, False, True])
    cases["rank_in_queue"] = cases.groupby("planner").cumcount() + 1
    return cases[cases.rank_in_queue <= top_n].reset_index(drop=True)

def hysteresis(ts_series, open_at=4.0, close_at=3.0, clear_cycles=2):
    """Open when |TS|>open_at; close only after |TS|<close_at for clear_cycles consecutive cycles [RoT]."""
    state, clear, out = "CLOSED", 0, []
    for v in ts_series:
        if state == "CLOSED" and abs(v) > open_at:
            state, clear = "OPEN", 0
        elif state == "OPEN":
            clear = clear + 1 if abs(v) < close_at else 0
            if clear >= clear_cycles:
                state, clear = "CLOSED", 0
        out.append(state)
    return out
```
**Test vector** (executed). The base row is all clear: margin 2, cost 5, $F_{LT}$ = 400, LT = 4 periods, daily forecast 10, SS days 10, max days 60.
- **I1** (A item): TS = −5.2 with Σ(F−A) = 120, and Bias +14% for 3 cycles.
  - TS VaR = 120 × 2 × 3 = 720.
  - Bias VaR = 0.14 × 400 × 2 × 3 = 336.
  - Primary = TS_OUT_OF_CONTROL (720); secondary = BIAS_PERSISTENT.
- **I2** (C item): DOS 90 > 60, and MASE 1.3 (MAE 9 vs. 6).
  - Excess VaR = 30 × 10 × 5 (cost) × 1 = **1,500**.
  - MASE VaR = 3 × 4 × 2 × 1 = 24.
  - Primary = EXCESS_DOS.
- **I3** (intermittent): Bias 12% for 3 cycles stays under the 15% intermittent limit, so no case.
- **Queue order:** I2 (1,500) > I1 (720).
- **Hysteresis** on TS [3, 4.5, 3.5, 2.9, 3.2, 2.0, 1.0, 5] gives CLOSED, OPEN, OPEN, OPEN, OPEN, OPEN, CLOSED, OPEN. The 3.2 resets the clear counter.

```sql
-- Trigger flags at decision level (one row per item-location); VaR ranking is done in §5 Python
SELECT item_id, location_id, planner, abc,
       ABS(ts) > 4                                                               AS trg_ts,
       bias_cycles_outside >= 3 AND ABS(bias_pct) >
         CASE WHEN demand_class IN ('INTERMITTENT','LUMPY') THEN 15 ELSE 10 END  AS trg_bias,
       mase > 1.0                                                                AS trg_mase,
       fva_final_vs_naive < 0                                                    AS trg_fva,
       in_frozen AND ABS(change_vs_last_pct) > 25                                AS trg_swing,
       pct_reconstructed > 0.30                                                  AS trg_censored,
       dos_fwd < ss_days                                                         AS trg_short,
       dos_fwd > max_days                                                        AS trg_excess,
       ABS(coverage_gap_pp) > 5                                                  AS trg_pi,
       unreconciled_pct > 0.5                                                    AS trg_unrec
FROM exception_metrics
WHERE metric_lag = :decision_lag AND snapshot_id = :snapshot;
```

## 6. Standardized Output Format
```markdown
# Demand Exception Queue — cycle <YYYY-MM-WW> — run EXC_<run_id>

## Executive Summary
- <N_cases> cases (<new> new, <aged> aged >4 cycles); queue covers <x>% of total VaR <V>.
- Top drivers: <SYSTEMATIC_BIAS n, VaR>, <AVAILABILITY n, VaR>, <SUPPLY_OR_POLICY n, VaR>.
- <n_below> cases below cap (VaR <v>) summarized, not hidden.
- Process health: median days to close <d>; repeat rate <r>%.

## Metadata (mandatory)
| Level | Grain | Lag | Time window (TS / bias / MASE) | Weighting | Thresholds version | Snapshot | Queue cap |
|---|---|---|---|---|---|---|---|
| SKU-Location (decision level) | WEEK | <ℓ> | 13 wks / 3 cycles / 13 wks | VaR = units × margin(cost) × ABC 3/2/1 | GS-G-03 default | <id> | 25 per planner |

## KPI Table
| Metric | Value | Rule |
|---|---|---|
| Cases opened / closed / aged | | GS-G-03 |
| % of total VaR in queue | | GS-G-03 |
| Median days to close | | n/a |
| Repeat rate (4 cycles) | | n/a |

## Exceptions (ranked per planner)
| ID | Planner | Rank | Item | Loc | Primary trigger (rule) | Secondary | Units at risk | VaR | Direction (words) | Root cause | Action | Owner | Age (cycles) | Status |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|

## Below-Cap Summary
| Planner | N cases | VaR | Top trigger |
|---|---|---|---|

## Recommendations
1. …

## Data-Quality Log
| item | issue (lag mismatch, NaN metric, SCALE_ZERO, missing margin) | treatment |
|---|---|---|

## Reproducibility
Thresholds, weights, cap, hysteresis parameters, input snapshot IDs, code §5.
```

## 7. Edge Cases
- **Missing margin:** value at cost × 0.3 [RoT] and flag `MARGIN_ASSUMED`. Never drop the case.
- **Tracking signal undefined** (MAD = 0, perfect or all-zero window): skip the TS trigger and log it.
- **New items** (segment NEW): only FROZEN_SWING, DOS and CENSORED triggers apply. Accuracy triggers wait for 13 clean weeks; launch accuracy belongs to `nextq-new-product-lifecycle-forecasting`.
- **Phase-out items:** suppress MASE/TS; keep EXCESS_DOS (run-out risk).
- **Shortage and excess on the same item at different locations:** separate cases; add a `REBALANCE` hint when both exist inside the same network.
- **Planner with no assignment:** route to the `UNASSIGNED` queue owned by the demand planning manager.
- **Threshold changes mid-cycle:** cases keep the threshold version they opened under until closed (logged).
- **Massive event** (e.g. more than 20% of items triggering at once [RoT]): raise one `SYSTEMIC_EVENT` case at the aggregate level (calendar shift, data feed failure) before planner queues are published.
