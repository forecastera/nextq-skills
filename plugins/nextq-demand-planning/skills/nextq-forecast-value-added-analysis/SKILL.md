---
name: nextq-forecast-value-added-analysis
description: >-
  [Next Quarter Supply Chain Suite] Runs Forecast Value Added (FVA) analysis (Gilliland): stair-step reports comparing
  naive, statistical, ML, analyst override, sales/marketing input, consensus and final forecasts on the same items,
  lag and window; FVA by participant, reason code, override direction and size; significance testing; and
  retain/suspend decisions for each forecasting touchpoint. Load when the user asks whether overrides or the consensus
  process add value, wants an FVA or stair-step report, asks if sales input or planner adjustments improve accuracy,
  wants to evaluate a forecasting process or a planner, or wants to cut non-value-adding steps. Inputs: versioned
  forecast snapshots per process step (with override logs: owner, reason code) and unconstrained actuals at
  SKU-Location-Week/Month. Triggers: "FVA", "forecast value added", "do our overrides help", "stair-step report",
  "naive vs statistical vs consensus", "judgmental adjustments".
metadata:
  author: "Next Quarter"
  organization: "Next Quarter (nextq)"
  suite: "Next Quarter Supply Chain Intelligence"
  version: "1.0.0"
---

# Forecast Value Added Analysis
*Part of the **Next Quarter (nextq)** Supply Chain Skill Suite — Governed by SUPPLY_CHAIN_GOLD_STANDARD.md.*

## 1. Role & Objective
**Persona:** Forecasting Process Auditor (FVA practitioner, after Gilliland / SAS / IBF practice).
**Decision supported:** Which forecasting process steps and participants improve accuracy versus the step before them and versus a naïve forecast? Which should be retained, retrained, restricted, or suspended?

**Objective:** Produce a statistically defensible FVA stair-step with breakdowns by step, participant, reason code, override direction, and override size. It applies the GS-G-01/02 governance rules and returns a deterministic retention decision per touchpoint and segment.

## 2. Gold Standard Rules Enforced
| Rule | Inlined standard |
|---|---|
| **GS-K-07** FVA [STD] | $\text{FVA}_s=\text{Err}(\text{prior step})-\text{Err}(s)$, using the **same metric** (WAPE by default; MASE per series as secondary), **same items, same periods, same lag**. **Positive = value added.** Standard stair-step: **Naïve → Statistical (→ ML/Reconciled) → Analyst override → Sales/Marketing input → Consensus → Final.** Each step is reported vs. the **prior step** and vs. **naïve**. The naïve model is the ultimate comparator: NF1 random walk $\hat y_{T+h}=y_T$, or NF2 seasonal $\hat y_{T+h}=y_{T+h-m(k+1)}$ for seasonal items (Gilliland; SAS white paper). Every version must be stored as a snapshot. |
| **GS-K-01** WAPE | $\sum\lvert A-F\rvert/\sum\lvert A\rvert\times100$ on unconstrained actuals. |
| **GS-K-04** MASE | Per series, scale $=\frac{1}{T-m}\sum\lvert y_t-y_{t-m}\rvert$ on pre-origin history. Per-series FVA in MASE units handles scale differences. |
| **GS-K-05** Bias | $\text{Bias\%}=\sum(F-A)/\sum A\times100$ (+ = over). FVA reports include bias change per step, because overrides often add optimism bias. |
| **GS-M-01 / GS-M-02** | The decision lag is stated. Naïve forecasts are generated at the same lag (NF1 at lag ℓ uses $\mathit{y}_{t-\ell}$). |
| **GS-G-01** Override evidence & policy [EVID + RoT] | Fildes, Goodwin, Lawrence & Nikolopoulos (2009), > 60,000 forecasts: **small adjustments** often damage accuracy, **larger adjustments** tend to improve it, and **positive (upward) adjustments** are less likely to help and show optimism bias. Policy: every override needs a **reason code** + rationale + owner + timestamp. **Materiality:** overrides with $\lvert\Delta\rvert<\max(0.10\cdot F_{\text{stat}},\ 1\cdot\text{MAD}_{\text{stat}})$ are blocked unless they correct a data error. **Upward overrides > 20%** need secondary approval [RoT]. This skill **audits** compliance. |
| **GS-G-02** Retention rule [RoT] | Track FVA by participant × reason code × direction × size bucket every cycle. A touchpoint with **negative FVA over a rolling ≥ 6 cycles**, with either a significant result (p < 0.05, or a bootstrap 95% CI entirely < 0) **or** the same sign in ≥ 5 of 6 cycles, is **suspended** for that segment; the prior step's forecast is used instead. If the statistical forecast does not beat naïve, the segment goes to model review, and naïve may be used until it is fixed. |
| **GS-K-13** | Report the level, N items, N periods, % reconstructed actuals. Aggregate FVA and decision-level FVA are reported separately. |
| **GS-G-03 / 10 / 11** | Exceptions, the standard output contract, and reproducible seeded tests. |

**Size buckets** (relative override size $\mathit{r}=\lvert F_{\text{adj}}-F_{\text{prior}}\rvert/F_{\text{prior}}$):
- Q1–Q4 quartiles of $\mathit{r}$ within the run, and fixed bands {< 5%, 5–10%, 10–20%, 20–50%, > 50%} [RoT].
- Direction: UP if $\mathit{F}_{\text{adj}}>F_{\text{prior}}$, DOWN if lower, NONE if equal.

**Directional hit rate:** an override is "right direction" if $\operatorname{sign}(F_{\text{adj}}-F_{\text{prior}})=\operatorname{sign}(A-F_{\text{prior}})$. Report it by direction.

## 3. Required Inputs & Data Schema
**`forecast_snapshots`** (same schema as `nextq-forecast-accuracy-diagnostics`):
`item_id, location_id, target_period, created_period, lag, forecast_version ∈ {NF1, NF2, STAT, ML, REC, ANALYST, SALES, MARKETING, CONSENSUS, FINAL}, fcst_qty (base UoM)`.

**`override_log`**: one row per override event.

| Column | Type | Req. | Notes |
|---|---|---|---|
| `override_id` | string | Y | |
| `item_id`, `location_id` (or `node_id` + `level`) | string | Y | Aggregate-level overrides are disaggregated per `nextq-consensus-demand-review` before scoring |
| `target_period`, `created_period` | date | Y | |
| `step` | enum | Y | ANALYST / SALES / MARKETING / CONSENSUS / EXECUTIVE |
| `owner_id` | string | Y | Participant |
| `reason_code` | enum | Y | PROMO, CUSTOMER_INTEL, NEW_LISTING, LOST_LISTING, SUPPLY_EVENT, PRICE_CHANGE, DATA_ERROR, MARKET_TREND, OTHER |
| `rationale` | text | Y | |
| `qty_before`, `qty_after` | decimal ≥ 0 | Y | |
| `approved_by` | string | Cond. | Required if UP > 20% |

**`actuals`**: unconstrained actuals (`nextq-demand-history-cleansing`), with structural periods excluded.

**`attributes`**: ABC, XYZ, demand_class, unit_margin.

**Minimum:**
- ≥ 6 completed cycles at the decision lag for the retention decision (fewer: report only, decision = `INSUFFICIENT_CYCLES`).
- ≥ 30 item-periods per breakdown cell, or the cell is merged into `OTHER` [RoT].

## 4. Execution Workflow
1. **Assemble the stair-step panel.** For each (item, location, target_period) at the decision lag, pivot the versions into columns. Generate NF1/NF2 if absent (NF2 for SEASONAL items, else NF1). Missing steps carry forward the prior step's value (no touch means FVA = 0) and are marked `UNTOUCHED`.
2. **Align scope.** Keep only rows that have actuals and every step being compared (`ITEMSET_ALIGNED`). Exclude structural periods.
3. **Step-level FVA.** For each step $\mathit{s}$, compute $\text{WAPE}_s$ and $\text{Bias}_s$ on the aligned panel. Then compute FVA vs. prior $=\text{WAPE}_{s-1}-\text{WAPE}_s$ and FVA vs. naïve $=\text{WAPE}_{\text{naïve}}-\text{WAPE}_s$. Also compute per-series MASE FVA (median).
4. **Touched-only FVA.** Repeat step 3 on **touched** item-periods only (where step $\mathit{s}$ changed the value). Untouched rows dilute the effect. Report both.
5. **Breakdowns** (touched rows): participant, reason code, direction, size bucket (quartile and fixed band), ABC, promo vs. non-promo, and lag. Per cell, report N, WAPE before/after, FVA, bias change, and directional hit rate.
6. **Significance.**
   - Compute the paired per-row absolute-error differential $\mathit{d}=\lvert A-F_{s-1}\rvert-\lvert A-F_s\rvert$.
   - **Bootstrap over items** (seeded, 2,000 resamples) for the CI of Σd / ΣA × 100 (WAPE FVA). Also run a Wilcoxon signed-rank test on per-item mean $\mathit{d}$ when N ≥ 20.
   - Report the CI and p-value. When > 3 breakdown cells are tested, apply Benjamini-Hochberg across cells.
7. **Compliance audit (GS-G-01):**
   - % of overrides with a valid reason code and rationale.
   - % **sub-materiality** overrides ($\lvert\Delta\rvert<\max(10\%\,F,\ \text{MAD})$, excluding DATA_ERROR).
   - % of UP > 20% overrides without `approved_by`.
   - FVA of sub-materiality overrides, which tests the Fildes et al. finding on your data.
8. **Rolling retention decision (GS-G-02)** per touchpoint × segment over the last ≥ 6 cycles:
   - Compute the per-cycle FVA sign and pooled significance.
   - Decision over the last 6 cycles: `RETAIN` (mean FVA ≥ 0), `WATCH` (mean FVA < 0 but neither significant nor ≥ 5/6 cycles negative), or `SUSPEND` (mean FVA < 0 and either significant or ≥ 5/6 cycles negative). Fewer than 6 cycles → `INSUFFICIENT_CYCLES`.
   - Also flag `STAT_BELOW_NAIVE` segments → model review.
9. **Cost of process (optional):** touches × minutes per touch [user input] vs. FVA, giving "value per hour".
10. **Output** (Section 6). Persist the decision table for `nextq-consensus-demand-review` (which enforces suspensions) and `nextq-demand-exception-management`.

## 5. Verification Snippets
```python
import numpy as np, pandas as pd

def wape(A, F): A, F = np.asarray(A, float), np.asarray(F, float); return 100 * np.abs(A - F).sum() / np.abs(A).sum()

def stairstep(panel, steps, actual="A"):
    """panel: one row per item-loc-period at decision lag; columns = steps (ordered) + actual."""
    rows, prev = [], None
    for s in steps:
        w = wape(panel[actual], panel[s]); b = 100 * (panel[s] - panel[actual]).sum() / panel[actual].sum()
        rows.append(dict(step=s, WAPE=w, Bias_pct=b,
                         FVA_vs_prior=(None if prev is None else rows[-1]["WAPE"] - w),
                         FVA_vs_naive=rows[0]["WAPE"] - w if rows else 0.0,
                         touched_pct=(None if prev is None else 100 * (panel[s] != panel[prev]).mean())))
        prev = s
    return pd.DataFrame(rows)

def fva_bootstrap(panel, prior, step, item_col="item", actual="A", n=2000, seed=42):
    """CI of WAPE FVA (pts) = 100*sum(|A-Fprior|-|A-Fstep|)/sum(A), resampling items."""
    g = panel.assign(d=(panel[actual] - panel[prior]).abs() - (panel[actual] - panel[step]).abs()) \
             .groupby(item_col).agg(d=("d", "sum"), A=(actual, "sum"))
    rng = np.random.default_rng(seed); idx = np.arange(len(g)); est = []
    for _ in range(n):
        s = g.iloc[rng.choice(idx, len(idx), replace=True)]
        est.append(100 * s.d.sum() / s.A.sum())
    point = 100 * g.d.sum() / g.A.sum()
    lo, hi = np.percentile(est, [2.5, 97.5])
    return point, lo, hi

def size_band(before, after):
    r = np.abs(after - before) / np.where(before == 0, np.nan, before)
    return pd.cut(r, [-np.inf, .05, .10, .20, .50, np.inf], labels=["<5%", "5-10%", "10-20%", "20-50%", ">50%"])

def materiality_violation(before, after, mad, reason):
    return (np.abs(after - before) < np.maximum(0.10 * before, mad)) & (reason != "DATA_ERROR")

def retention_decision(cycle_fva: list[float], ci_hi: float, p_value: float | None):
    """GS-G-02: >=6 cycles; negative & (significant or >=5/6 negative) -> SUSPEND."""
    if len(cycle_fva) < 6: return "INSUFFICIENT_CYCLES"
    last = cycle_fva[-6:]; neg = sum(f < 0 for f in last); pooled_neg = np.mean(last) < 0
    signif = (ci_hi < 0) or (p_value is not None and p_value < 0.05)
    if pooled_neg and (signif or neg >= 5): return "SUSPEND"
    if pooled_neg: return "WATCH"
    return "RETAIN"
```
**Test vector** (verified):
- `A=[100,100,100,100]`, NF1 = `[80,120,90,110]`, STAT = `[95,105,98,102]`, ANALYST = `[100,115,98,110]`.
- WAPE: NF1 15.0, STAT 3.5, ANALYST 6.75.
- FVA: STAT vs. naïve +11.5; ANALYST vs. prior −3.25 (value destroyed, Bias +5.75% over); ANALYST vs. naïve +8.25.

```sql
-- Stair-step at decision lag on the aligned item set (GS-K-07)
WITH p AS (
  SELECT item_id, location_id, target_period, a.actual_unconstrained AS A,
         MAX(CASE WHEN forecast_version='NF1'       THEN fcst_qty END) AS nf1,
         MAX(CASE WHEN forecast_version='STAT'      THEN fcst_qty END) AS stat,
         MAX(CASE WHEN forecast_version='CONSENSUS' THEN fcst_qty END) AS cons,
         MAX(CASE WHEN forecast_version='FINAL'     THEN fcst_qty END) AS fin
  FROM forecast_snapshots f JOIN actuals a USING (item_id, location_id)
  WHERE f.lag = :lag AND a.period = f.target_period AND a.period_state <> 'STRUCTURAL_ZERO'
  GROUP BY 1,2,3,4)
SELECT 100*SUM(ABS(A-nf1))/SUM(A) AS wape_naive, 100*SUM(ABS(A-stat))/SUM(A) AS wape_stat,
       100*SUM(ABS(A-cons))/SUM(A) AS wape_cons,  100*SUM(ABS(A-fin))/SUM(A)  AS wape_final
FROM p WHERE nf1 IS NOT NULL AND stat IS NOT NULL AND cons IS NOT NULL AND fin IS NOT NULL;
```

## 6. Standardized Output Format
```markdown
# Forecast Value Added — <scope> — run FVA_<run_id>

## Executive Summary
- Final forecast WAPE <x>% vs naïve <y>% → total FVA <y−x> pts; statistical step contributes <a> pts, human steps <b> pts.
- Touched-only: analyst overrides FVA <c> pts (CI <lo, hi>), sales input <d> pts, consensus <e> pts.
- Overrides: <n> events; <u>% UP (hit rate <h_up>%), <v>% DOWN (hit rate <h_dn>%); sub-materiality <s>% with FVA <f> pts.
- Decisions: SUSPEND <k> touchpoint-segments, WATCH <w>, RETAIN <r>; STAT_BELOW_NAIVE <z> segments.

## Metadata (mandatory)
| Level | Grain | Lag | Time window | Cycles | Weighting | Metric | Actuals basis | N item-periods | % touched |
|---|---|---|---|---|---|---|---|---|---|
| SKU-Location | WEEK | <ℓ> | <start–end> | <c> | volume (WAPE) | WAPE (MASE secondary) | unconstrained; <r>% reconstructed | | |

## FVA Stair-Step (aligned items)
| Process step | WAPE % | Bias % (+=over) | FVA vs prior (pts) | FVA vs naïve (pts) | 95% CI vs prior | % touched |
|---|---|---|---|---|---|---|
| Naïve (NF1/NF2) | | | — | — | — | — |
| Statistical | | | | | | — |
| Analyst override | | | | | | |
| Sales / Marketing | | | | | | |
| Consensus | | | | | | |
| Final | | | | | | |

## FVA by Participant / Reason / Direction / Size (touched only)
| Dimension | Cell | N | WAPE before | WAPE after | FVA pts | 95% CI | Bias Δ pts | Dir. hit rate % | Decision |
|---|---|---|---|---|---|---|---|---|---|
| Participant | <owner> | | | | | | | | |
| Reason code | PROMO | | | | | | | | |
| Direction | UP / DOWN | | | | | | | | |
| Size band | <5% / 5–10% / 10–20% / 20–50% / >50% | | | | | | | | |

## Governance Compliance (GS-G-01)
| Check | Value | Threshold | Status |
|---|---|---|---|
| Overrides with reason code + rationale | % | 100% | |
| Sub-materiality overrides (excl. DATA_ERROR) | % | 0% | |
| UP >20% without secondary approval | % | 0% | |

## Exceptions
| ID | Trigger (rule) | touchpoint / owner | segment | FVA pts | cycles negative | action | owner |
|---|---|---|---|---|---|---|---|
| | SUSPEND (GS-G-02) | | | | | revert to prior step for segment | |
| | STAT_BELOW_NAIVE (GS-G-02) | | | | | model review; naïve interim | |
| | Optimism bias: UP overrides FVA<0 | | | | | require evidence + approval | |
| | Non-compliant overrides | | | | | training / block in tool | |

## Recommendations
1. …

## Data-Quality Log
| issue | count | treatment |
|---|---|---|
| steps missing (carried forward as UNTOUCHED) | | |
| overrides without actuals yet | | excluded |
| aggregate overrides disaggregated | | per nextq-consensus-demand-review rule |

## Reproducibility
Snapshot ids, lag, step order, bootstrap n/seed, BH correction, size bands, code §5.
```

## 7. Edge Cases
- **Steps that touch almost nothing** (< 2% of rows): the overall FVA ≈ 0 by dilution. Lead with touched-only FVA.
- **Overrides made after actuals are partly known** (inside the period): exclude them, or score them at their true (shorter) lag. Never score them at the decision lag.
- **Promo overrides:** compare against the statistical forecast **plus** the modeled promo lift (from `nextq-promotion-lift-cannibalization-planning`) when available, so a planner is not credited for lift the model already had.
- **Censored actuals:** an UP override that looks wrong may be right if sales were censored. Use unconstrained actuals. If the item has > 30% reconstructed actuals, the decision is capped at `WATCH`.
- **Mixed levels:** overrides entered at brand level are disaggregated with the documented rule before item-level scoring. Also report the aggregate-level FVA.
- **Small cells:** merge cells with < 30 item-periods into `OTHER`. Never suspend on a cell below that size.
