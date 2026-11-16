---
name: nextq-consensus-demand-review
description: >-
  [Next Quarter Supply Chain Suite] Runs the monthly S&OP / IBP demand review: builds the unconstrained consensus
  demand plan from the statistical baseline plus governed sales, marketing and planner overrides; enforces reason
  codes, materiality thresholds, approvals, time fences (frozen / slushy / liquid) and FVA-based suspensions;
  disaggregates aggregate overrides coherently; and produces the change waterfall, assumptions log and decision pack.
  Load when the user asks to run or prepare a demand review or consensus meeting, consolidate sales and marketing
  input, apply or approve overrides, build the consensus or unconstrained demand plan, explain why the plan changed vs
  last cycle, or prepare the S&OP demand handoff. Inputs: baseline / reconciled forecasts, override submissions, FVA
  decisions, last cycle's plan, time fences, prices, at SKU-Location-Week/Month and family-region-month. Triggers:
  "demand review", "consensus forecast", "S&OP demand plan", "sales input", "override approval".
metadata:
  author: "Next Quarter"
  organization: "Next Quarter (nextq)"
  suite: "Next Quarter Supply Chain Intelligence"
  version: "1.0.0"
---

# Consensus Demand Review
*Part of the **Next Quarter (nextq)** Supply Chain Skill Suite — Governed by SUPPLY_CHAIN_GOLD_STANDARD.md.*

## 1. Role & Objective
**Persona:** Demand Review Process Owner (S&OP / IBP demand manager).
**Decision supported:** What is the agreed, **unconstrained** demand plan for the horizon? Which overrides are accepted, rejected, or escalated? What changed vs. last cycle, and why? Which risks and opportunities go to the supply review and the management business review?

**Objective:** A governed consensus plan that is coherent across the hierarchy, compliant with override policy, auditable to the reason-code level, and handed off with a value view and open decisions. **Only unconstrained demand** is decided here. Supply constraints are an S&OP gap decision (`nextq-supply-demand-balancing`), not a forecast edit.

## 2. Gold Standard Rules Enforced
| Rule | Inlined standard |
|---|---|
| **GS-G-01** Override policy [EVID + RoT] | Evidence (Fildes et al. 2009, > 60,000 forecasts): small adjustments often hurt, large ones tend to help, and upward ones are less likely to help (optimism bias). **Rules:** (1) Every override has a **reason code** (PROMO, CUSTOMER_INTEL, NEW_LISTING, LOST_LISTING, SUPPLY_EVENT [demand-side only], PRICE_CHANGE, DATA_ERROR, MARKET_TREND, OTHER) + rationale + owner + timestamp. No reason, no override. (2) **Materiality:** reject if $\lvert\Delta\rvert<\max(0.10\cdot F_{\text{base}},\ 1\cdot\text{MAD}_{\text{base}})$ at the decision level, unless DATA_ERROR. (3) **UP > 20%** vs. base requires secondary approval (demand manager) + documented evidence, and is tracked separately. (4) Overrides are applied at a **declared level** and disaggregated coherently (GS-D-07). |
| **GS-G-02** FVA retention | Apply the current `nextq-forecast-value-added-analysis` decisions. A touchpoint × segment marked **SUSPEND** cannot override in that segment: the submission is auto-rejected with code `FVA_SUSPENDED`. **WATCH** submissions need demand-manager approval. |
| **GS-G-04** Time fences [RoT, configurable] | **Frozen** (≤ cumulative lead time, e.g. 0–4 weeks): changes only by exception approval (S&OP leader); default reject. **Slushy** (e.g. 5–12 weeks): reason code + materiality + approval rules. **Liquid** (> 12 weeks): open, subject to the rules. Every consensus version is snapshotted at each lag for FVA. |
| **GS-G-05** Constrained vs. unconstrained | Keep three plans separate and labeled: **unconstrained consensus demand** (this skill), the constrained supply / sales plan, and the financial plan. Accuracy is measured on unconstrained demand. Gaps are S&OP decisions. A SUPPLY_EVENT reason may model **demand** effects (for example, a competitor stockout); it may never reduce demand for **our** constraint. |
| **GS-D-07** Coherence | Disaggregate a parent override $O_P$ to its children with the **base-forecast proportions** $c_{\text{new}}=c_{\text{base}}\cdot O_P/\sum c_{\text{base}}$. If $\sum c_{\text{base}}=0$, use average historical proportions $p_j=\frac1T\sum y_{j,t}/y_t$; if those are also 0, use an equal split. Locked child overrides are honored first: the parent remainder is spread over unlocked children. Final coherence: $\lvert\text{parent}-\sum\text{children}\rvert\le10^{-6}$ relative. |
| **GS-K-05** Bias | Show each participant's trailing Bias% $=\sum(F-A)/\sum A$ (+ = over) next to their submission, so the meeting sees their track record. |
| **GS-K-07** FVA hook | Store versions STAT → ANALYST → SALES/MARKETING → CONSENSUS → FINAL with creation dates. |
| **GS-G-06** Value view (handoff) | Value $V=\sum_i Q_i\times P_i^{\text{net}}$, where net price = list − trade/discounts. Gap to the financial plan $V_{\text{demand}}-V_{\text{fin}}$ is reported, and > 2% at the reconciliation level escalates [RoT]. The volume plan is **never plugged** to hit the financial number (full reconciliation: `nextq-demand-financial-reconciliation`). |
| **GS-G-03** Exceptions | Consensus change vs. last cycle > ±25% within the frozen/slushy zone [RoT]; consensus deviates from base by > 30% with no approved evidence; participant Bias% > +10% submitting UP overrides; incoherence after edits. |
| **GS-G-10 / 11** | Standard output; reproducible override application. |

## 3. Required Inputs & Data Schema
**`base_plan`**: the reconciled statistical/ML baseline (`nextq-hierarchical-forecast-reconciliation` output).
`node_id, level, item_id, location_id (bottom), period, fcst_qty (base UoM), fcst_p10, fcst_p90, mad_base (per bottom series, at decision lag), version='REC'`.

**`override_submissions`**:

| Column | Type | Req. | Notes |
|---|---|---|---|
| `submission_id` | string | Y | |
| `owner_id`, `step` | string, enum (ANALYST/SALES/MARKETING/EXEC) | Y | |
| `node_id`, `level` | string | Y | Declared override level |
| `period` (or `period_from`/`period_to`) | date | Y | |
| `override_type` | enum | Y | ABSOLUTE (new qty), DELTA_QTY, DELTA_PCT |
| `value` | decimal | Y | |
| `reason_code`, `rationale`, `evidence_link` | enum, text, url | Y, Y, Cond. | Evidence required if UP > 20% |
| `lock_children` | bool | N | |
| `submitted_at` | timestamp | Y | Submissions after the cut-off are rejected (`LATE`) |

**`fva_decisions`** (from `nextq-forecast-value-added-analysis`): `step, owner_id, segment_key, decision ∈ {RETAIN, WATCH, SUSPEND, INSUFFICIENT_CYCLES}`.

**`last_cycle_plan`**: `node_id, period, consensus_qty`.

**`time_fences`**: `frozen_periods, slushy_periods` (per item group, in plan buckets).

**`prices`**: `item_id, location_id, period, net_price`.

**`participant_bias`**: trailing Bias% per owner (from `nextq-forecast-accuracy-diagnostics`).

**`hierarchy`**: as in `nextq-hierarchical-forecast-reconciliation`.

**Granularity:** decisions at the declared override level. Storage at the bottom grain (SKU-Location-Week). Meeting views at family-region-month (day-apportioned, GS-D-09).

## 4. Execution Workflow
1. **Freeze inputs.** Snapshot `base_plan` as the `STAT/REC` version, and record the cut-off timestamp.
2. **Normalize submissions** to an absolute target $O$ at the declared node and period: ABSOLUTE → value; DELTA_QTY → base + value; DELTA_PCT → base × (1 + value). Compute $\Delta=O-F_{\text{base}}$ and $r=\Delta/F_{\text{base}}$.
3. **Validate each submission** in this order (first failure wins) and record the status:
   1. `LATE` (after cut-off) → reject.
   2. Missing reason code or rationale → reject `NO_REASON`.
   3. FVA decision `SUSPEND` for (step, owner, segment) → reject `FVA_SUSPENDED`.
   4. **Time fence:** frozen → `ESCALATE_FROZEN` (S&OP leader); slushy and liquid continue.
   5. **Materiality:** $\lvert\Delta\rvert<\max(0.10F_{\text{base}},\ \text{MAD}_{\text{node}})$ and reason ≠ DATA_ERROR → reject `IMMATERIAL`. $\text{MAD}_{\text{node}}$ for an aggregate = the MAD of the node's base forecast errors (or Σ child MAD as an upper bound).
   6. **Constraint leakage:** reason SUPPLY_EVENT with $\Delta<0$ and rationale referencing own supply → reject `CONSTRAINT_NOT_DEMAND` (route to the supply review).
   7. **UP > 20%**: requires `evidence_link` and approval → `PENDING_APPROVAL` until approved.
   8. FVA `WATCH` → `PENDING_APPROVAL`.
   9. Otherwise → `ACCEPTED`.
4. **Resolve conflicts.** For multiple accepted submissions on the same node-period, apply them in step order ANALYST → SALES/MARKETING → EXEC. A later step overrides an earlier one, with both logged. Submissions on parent and child for the same period: the child is locked first, then the parent remainder is applied to unlocked children.
5. **Disaggregate (GS-D-07)** each accepted parent override with the proportional rule and its fallbacks, honoring locks. If the locked children sum exceeds $O_P$, reject the parent override as `LOCK_CONFLICT` and escalate.
6. **Build the consensus** at the bottom grain, then re-aggregate bottom-up. **Verify coherence** (tolerance $10^{-6}$).
7. **Waterfall vs. last cycle** per node-period: last consensus → base change (model/actuals) → accepted overrides by reason code → consensus. The bridge must sum exactly.
8. **Exceptions and risk register:** apply the GS-G-03 triggers. List the upside/downside scenarios submitted (not applied) with probability and impact.
9. **Value view:** $V=\sum Q\times P^{\text{net}}$ by family-region-month, with the gap to the financial plan. Flag > 2% for the management business review (no plugging).
10. **Publish** `CONSENSUS_<cycle>` snapshot (+ `FINAL` after approvals close), the decisions log, and the pack (Section 6). Hand off to `nextq-supply-demand-balancing` and `nextq-demand-financial-reconciliation`.

## 5. Verification Snippets
```python
import numpy as np, pandas as pd

REASONS = {"PROMO","CUSTOMER_INTEL","NEW_LISTING","LOST_LISTING","SUPPLY_EVENT","PRICE_CHANGE","DATA_ERROR","MARKET_TREND","OTHER"}

def validate(sub, base, mad, fence, fva_decision, cutoff, approved=False):
    """Returns status per GS-G-01/02/04/05 (first failure wins)."""
    O = {"ABSOLUTE": sub["value"], "DELTA_QTY": base + sub["value"], "DELTA_PCT": base * (1 + sub["value"])}[sub["override_type"]]
    d = O - base; r = d / base if base > 0 else np.inf
    if sub["submitted_at"] > cutoff: return "LATE", O
    if sub.get("reason_code") not in REASONS or not sub.get("rationale"): return "NO_REASON", O
    if fva_decision == "SUSPEND": return "FVA_SUSPENDED", O
    if fence == "FROZEN": return "ESCALATE_FROZEN", O
    if abs(d) < max(0.10 * base, mad) and sub["reason_code"] != "DATA_ERROR": return "IMMATERIAL", O
    if sub["reason_code"] == "SUPPLY_EVENT" and d < 0 and sub.get("own_supply", False): return "CONSTRAINT_NOT_DEMAND", O
    if r > 0.20 and not (sub.get("evidence_link") and approved): return "PENDING_APPROVAL", O
    if fva_decision == "WATCH" and not approved: return "PENDING_APPROVAL", O
    return "ACCEPTED", O

def disaggregate(O_parent, child_base: pd.Series, hist_prop: pd.Series | None = None, locked: pd.Series | None = None):
    """GS-D-07 proportional split honoring locked children; fallbacks: historical proportions -> equal."""
    locked = locked if locked is not None else pd.Series(dtype=float)
    rem = O_parent - locked.sum()
    if rem < -1e-9: raise ValueError("LOCK_CONFLICT")
    free = child_base.drop(locked.index, errors="ignore")
    if free.sum() > 0:                       w = free / free.sum()
    elif hist_prop is not None and hist_prop.reindex(free.index).sum() > 0:
        hp = hist_prop.reindex(free.index).fillna(0); w = hp / hp.sum()
    else:                                    w = pd.Series(1 / len(free), index=free.index)
    out = pd.concat([locked, rem * w])
    assert abs(out.sum() - O_parent) <= 1e-6 * max(abs(O_parent), 1)
    return out

def waterfall(last, base_now, consensus, overrides_by_reason: dict):
    """Bridge must close exactly: last -> base change -> overrides by reason -> consensus."""
    steps = {"last_consensus": last, "base_change": base_now - last, **overrides_by_reason}
    assert abs(last + (base_now - last) + sum(overrides_by_reason.values()) - consensus) < 1e-6, "bridge does not close"
    return steps
```
**Test vectors** (executed):
- base = 100, MAD = 8: DELTA_PCT +0.05 → `IMMATERIAL`; +0.15 → `ACCEPTED`; +0.30 without evidence → `PENDING_APPROVAL`.
- `disaggregate(120, children=[60,30,10])` → [72, 36, 12].
- With child A locked at 50 → A = 50, and the remaining 70 split 52.5/17.5.

```sql
-- Post-build coherence and compliance checks (must return zero rows each)
SELECT node_id, period FROM consensus_parent p
WHERE ABS(p.qty - (SELECT SUM(c.qty) FROM consensus_bottom c JOIN hierarchy_map h
                   ON h.bottom_id=c.bottom_id WHERE h.node_id=p.node_id AND c.period=p.period))
      > 1e-6*GREATEST(ABS(p.qty),1);
SELECT submission_id FROM override_decisions
WHERE status='ACCEPTED' AND (reason_code IS NULL OR rationale IS NULL OR (pct_change>0.20 AND approved_by IS NULL));
```

## 6. Standardized Output Format
```markdown
# Demand Review Pack — cycle <YYYY-MM> — CONSENSUS_<cycle>

## Executive Summary
- Unconstrained consensus for <horizon>: <units> (<+/-x>% vs last cycle; <+/-y>% vs base); value <currency> (gap to financial plan <g>% → <escalate/ok>).
- Overrides: <n> submitted, <a> accepted, <r> rejected (IMMATERIAL <i>, NO_REASON <nr>, FVA_SUSPENDED <s>, FROZEN <f>), <p> pending approval.
- Top drivers of change: <reason codes with units>; key risks/opportunities: <list>.
- Decisions needed at MBR: <list>.

## Metadata (mandatory)
| Level (decision / storage / view) | Grain | Horizon | Time fences | Lag snapshot | Weighting | Unit | Price basis | Base version |
|---|---|---|---|---|---|---|---|---|
| declared / SKU-Loc-Week / Family-Region-Month | WEEK→MONTH (day-apportioned) | <periods> | frozen 0–4w, slushy 5–12w, liquid >12w | CONSENSUS at lags 1..H | volume (units); value Σ Q×P_net | base UoM | net price | REC_<id> |

## Consensus vs Base vs Last Cycle (family-region-month)
| Family | Region | Month | Last consensus | Base (REC) | Accepted overrides | Consensus | Δ vs last % | Δ vs base % | Value | Fence |
|---|---|---|---|---|---|---|---|---|---|---|

## Change Waterfall (total horizon)
| Step | Units | Value |
|---|---|---|
| Last cycle consensus | | |
| Base change (new actuals / model) | | |
| Overrides: PROMO | | |
| Overrides: CUSTOMER_INTEL | | |
| Overrides: other reasons … | | |
| **This cycle consensus** | | |

## Override Decisions Log
| submission | owner (trailing Bias %) | step | node | period | base | requested | Δ % | reason | fence | FVA status | decision | approver |
|---|---|---|---|---|---|---|---|---|---|---|---|---|

## Exceptions
| ID | Trigger (rule, threshold) | node | period | value | action | owner |
|---|---|---|---|---|---|---|
| | Δ vs last >±25% in frozen/slushy (GS-G-03) | | | | confirm with sales/supply | |
| | Consensus vs base >30% without approved evidence | | | | demand manager review | |
| | UP override from owner with Bias >+10% | | | | require evidence | |
| | LOCK_CONFLICT / incoherence | | | | resolve before publish | |
| | Value gap to financial plan >2% (GS-G-06) | | | | MBR decision (no plugging) | |

## Risks & Opportunities (not in consensus)
| ID | Description | Node | Periods | Units ± | Probability | Value ± | Owner |
|---|---|---|---|---|---|---|---|

## Recommendations / Decisions Requested
1. …

## Data-Quality / Assumptions Log
| assumption / issue | source | treatment |
|---|---|---|

## Reproducibility
Input snapshot ids (base, submissions, FVA decisions, fences, prices), cut-off timestamp, disaggregation rule and fallbacks, code §5.
```

## 7. Edge Cases
- **Base = 0 with an override > 0** (new listing): materiality is undefined. Accept only with reason NEW_LISTING and route the shape and quantity to `nextq-new-product-lifecycle-forecasting`. Flag `ZERO_BASE`.
- **Override ranges spanning fences:** split the submission by period. Each part is validated against its own fence.
- **Sales quota disguised as forecast** (override equals target or budget): if the consensus equals the financial plan within 0.5% at many nodes, flag `PLUGGING_SUSPECTED` (GS-G-06).
- **Conflicting submissions from the same step:** the latest timestamp wins, and both are logged. Different steps follow the step order.
- **Customer-level overrides in a location hierarchy:** convert via the customer → ship-to location mapping before disaggregation, and log the mapping version.
- **Cancelled promotions after submission:** PROMO overrides carry `promo_id`. If the promo status becomes cancelled, the override is auto-withdrawn and the plan re-published.
