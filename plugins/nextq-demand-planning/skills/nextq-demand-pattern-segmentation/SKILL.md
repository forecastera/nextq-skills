---
name: nextq-demand-pattern-segmentation
description: >-
  [Next Quarter Supply Chain Suite] Classifies every item-location demand series (smooth / erratic / intermittent /
  lumpy via ADI and CV², ABC value, XYZ forecastability, count-distribution fit, lifecycle) and routes each to a
  forecasting model family, inventory distribution, service-level tier and review cadence. Load when the user asks to
  segment or classify SKUs, run ABC/XYZ or ADI/CV² (Syntetos-Boylan) analysis, decide which forecasting method suits
  which items, find intermittent or lumpy items, spot slow movers or obsolescence risk, or set differentiated service
  targets. Inputs: cleaned demand history (output of nextq-demand-history-cleansing) at SKU-Location-Week/Month, unit
  cost or price, optional in-sample naive errors. Triggers: "which items are intermittent", "ABC XYZ", "demand
  classification", "Croston or ETS", "forecastability", "segment my portfolio".
metadata:
  author: "Next Quarter"
  organization: "Next Quarter (nextq)"
  suite: "Next Quarter Supply Chain Intelligence"
  version: "1.0.0"
---

# Demand Pattern Segmentation
*Part of the **Next Quarter (nextq)** Supply Chain Skill Suite — Governed by SUPPLY_CHAIN_GOLD_STANDARD.md.*

## 1. Role & Objective
**Persona:** Forecasting Methods Architect / Inventory Segmentation Analyst.
**Decision supported:** For each item-location, decide:
- which model family to try first,
- which demand distribution to assume for inventory,
- which ABC/XYZ tier (and therefore service target, review cadence, and override eligibility) applies.

**Objective:** Produce a deterministic, reproducible segmentation table with routing codes. It feeds `nextq-statistical-baseline-forecasting`, `nextq-intermittent-demand-forecasting`, `nextq-ml-causal-demand-forecasting`, and the inventory skills. **Routing is a starting hypothesis.** The final model is always confirmed by a rolling-origin backtest (GS-M-04/16).

## 2. Gold Standard Rules Enforced
| Rule | Inlined standard |
|---|---|
| **GS-M-03** Demand classification [STD] | Compute on the **cleaned, unconstrained** history (`train_mask = true`) at the planning bucket. $\text{ADI} = \dfrac{\#\text{periods}}{\#\{t: y_t>0\}}$, and $\text{CV}^2 = \left(\dfrac{\sigma(y_t\mid y_t>0)}{\mu(y_t\mid y_t>0)}\right)^2$ using the population SD ($\text{ddof}=0$) of non-zero sizes. **Smooth:** ADI < 1.32 & CV² < 0.49. **Erratic:** ADI < 1.32 & CV² ≥ 0.49. **Intermittent:** ADI ≥ 1.32 & CV² < 0.49. **Lumpy:** ADI ≥ 1.32 & CV² ≥ 0.49. **Caveats:** the cut-offs were derived to choose Croston vs. SBA (Syntetos, Boylan & Croston 2005). Kostenko & Hyndman (2006) show the exact boundary is non-linear (limits p = 4/3, v = 0.5) and that SBA is better over most of the region. Treat the quadrant as **routing only**, and use SBA (not Croston) as the intermittent default. Do **not** classify with < 2 non-zero observations or < 1 full seasonal cycle; route those series to `NEW_SPARSE`. |
| **GS-M-04** Routing rules | Smooth/erratic: ETS/ARIMA (weekly: STL+ETS or DHR; daily: multi-seasonal + holiday dummies). Erratic with zeros: also test SBA. Intermittent/lumpy: SBA default, TSB if obsolescence risk, ADIDA/iMAPA. Causal drivers present: dynamic regression / ML. Always confirm with a backtest. |
| **GS-M-08** Count / zero-inflated distribution | **Small counts:** max ≤ 10 per period or zero share ≥ 30% [RoT] → do not assume normal. Variance-to-mean ratio $\text{VMR} = \sigma^2/\mu$ over all train periods: VMR ≈ 1 (0.8–1.2 [RoT]) → **Poisson**; VMR > 1.2 → **Negative Binomial**. Observed zero share > the expected zero share under the fitted Poisson/NB by > 10 percentage points [RoT] → **ZIP/ZINB or hurdle**. Lumpy (CV² ≥ 0.49 & ADI ≥ 1.32) → **compound (Poisson-gamma / stuttering Poisson)** candidate. Expected zero share: Poisson $\mathit{e}^{-\mu}$; NB $\left(\frac{r}{r+\mu}\right)^r$ with $\mathit{r}=\mu^2/(\sigma^2-\mu)$. |
| **GS-G-09** ABC × XYZ × class | **ABC** by trailing-12-month (or 52-week) value = Σ(qty × unit cost). Cumulative share: A ≤ 80%, B ≤ 95%, C rest [RoT]. **XYZ** by forecastability, assigned on the CV of period demand $\text{CV}=\sigma(y)/\mu(y)$ (population SD, all train periods): X ≤ 0.5, Y ≤ 1.0, Z > 1.0 [RoT]. The in-sample naïve scale $\mathit{s}$ (GS-K-04) and the relative naïve error $\mathit{s}/\bar y$ are reported alongside it as a secondary forecastability indicator. The segment sets review frequency, CSL target tier, override eligibility, and model family. |
| **GS-K-04** MASE scale | Report the in-sample naïve scale $\mathit{s} = \frac{1}{T-m}\sum_{t=m+1}^{T}\lvert y_t - y_{t-m}\rvert$ ($\mathit{m}=1$ if non-seasonal) as the forecastability denominator downstream skills must reuse. If $\mathit{s} = 0$ (constant series), flag `SCALE_ZERO` and fall back to WAPE. |
| **GS-D-02 / GS-D-12** | Structural zeros are excluded from ADI/CV². Lifecycle `NEW` (< 1 cycle) → `NEW_SPARSE`. `PHASE_OUT` → TSB / run-out. |
| **GS-G-10** | Standard output contract. |

**Obsolescence-risk flag [RoT]:** trailing zero run ≥ max(3 × ADI, 1/4 of a seasonal cycle), OR the TSB-style decline test (last-26-week non-zero share < 50% of the prior-26-week share). Either one routes to **TSB**.

## 3. Required Inputs & Data Schema
**`demand_clean`** (from `nextq-demand-history-cleansing`):

| Column | Type | Unit | Req. |
|---|---|---|---|
| `item_id`, `location_id` (`channel`) | string | n/a | Y |
| `period_start` | date | fiscal bucket | Y |
| `period_grain` | enum WEEK/MONTH/DAY | n/a | Y |
| `demand_clean` | decimal ≥ 0 | base UoM | Y |
| `train_mask` | bool | n/a | Y (false = structural → excluded) |
| `period_state`, `needs_model_recon` | enum / bool | n/a | Y |
| `lifecycle_status` | enum | n/a | Y |
| `promo_flag`, `event_flags`, `price` | n/a | n/a | N (driver-presence test) |

**`item_cost`**: `item_id, location_id, unit_cost` (currency per base UoM; standard cost). If it is absent, run ABC on units and flag `ABC_ON_UNITS`.

**Minimum history:** ≥ 1 full seasonal cycle (52 weeks / 12 months) **and** ≥ 2 non-zero periods in the train mask. Otherwise the series is `NEW_SPARSE`. Recommended: ≥ 2 cycles for seasonality detection.

**Seasonality test (for routing only):** seasonal strength from STL, $\mathit{F}_S = \max\left(0, 1 - \frac{\operatorname{Var}(R)}{\operatorname{Var}(S+R)}\right)$ (FPP3 §4.3). Flag `SEASONAL` when $\mathit{F}_S ≥ 0.64$ [RoT] and history ≥ 2 cycles. Not computed for ADI ≥ 1.32.

## 4. Execution Workflow
1. **Load and filter.** Keep rows with `train_mask = true`. Record `n_periods`, `n_nonzero`, `pct_reconstructed`, and `needs_model_recon_share`. If `needs_model_recon_share > 0.30`, flag `LOW_CONFIDENCE` (GS-M-11 carry-through).
2. **Eligibility gate.**
   - `n_nonzero < 2` OR `n_periods < m` → `segment = NEW_SPARSE`, route `R_NEW_ANALOG`. Skip steps 3–5.
   - All zeros → `NO_DEMAND`, route `R_OBSOLESCENCE_REVIEW`.
3. **ADI and CV²** per GS-M-03. Use the population SD, and set CV² = 0 when there is one distinct non-zero size. Use **≥** at the cut-offs, so ADI = 1.32 counts as intermittent (decision: ties go to the intermittent side, which routes to the safer SBA).
4. **Quadrant:**
```
class = SMOOTH        if ADI < 1.32 and CV2 < 0.49
        ERRATIC       if ADI < 1.32 and CV2 >= 0.49
        INTERMITTENT  if ADI >= 1.32 and CV2 < 0.49
        LUMPY         if ADI >= 1.32 and CV2 >= 0.49
```
5. **Distribution (GS-M-08):** VMR, zero-share test, and the `SMALL_COUNT` flag. Output `inv_distribution ∈ {NORMAL, POISSON, NEG_BIN, ZI_POISSON, ZI_NEG_BIN, COMPOUND_POISSON}`. Use `NORMAL` only if not SMALL_COUNT, class ∈ {SMOOTH, ERRATIC}, and CV ≤ 0.5. Otherwise use the count family.
6. **Seasonality and drivers:**
   - `SEASONAL` flag (above).
   - `DRIVER_RICH` = promo_flag share ≥ 5% of periods OR price CV ≥ 0.05 OR event flags present [RoT].
7. **Obsolescence flag** (rule above).
8. **ABC:**
   - Sort item-locations by value descending.
   - Compute cumulative share $\mathit{c}_i$. Assign A if $\mathit{c}_i ≤ 0.80$, B if $\mathit{c}_i ≤ 0.95$, else C. The item that crosses a boundary takes the higher class.
9. **XYZ:**
   - Compute CV of period demand and the in-sample naïve scale $\mathit{s}$.
   - Assign X if CV ≤ 0.5, Y if CV ≤ 1.0, else Z.
   - Intermittent and lumpy series are Z by definition unless CV ≤ 1.0 (they are then flagged Y).
10. **Routing code** (first match wins):

| Condition | Route | Candidate set handed to backtest |
|---|---|---|
| NEW_SPARSE | `R_NEW_ANALOG` | analog / likeness curves (B4), naïve |
| NO_DEMAND or obsolescence & ADI ≥ 1.32 | `R_TSB` | TSB, SBA, zero forecast, naïve |
| INTERMITTENT or LUMPY | `R_INTERMITTENT` | SBA (default), TSB, ADIDA, iMAPA, Croston (reference only), NF1/NF2 |
| SMOOTH/ERRATIC & DRIVER_RICH | `R_CAUSAL` | dynamic regression / ML (A5) **plus** the R_STAT set |
| ERRATIC & zero share > 0 | `R_STAT_PLUS_SBA` | ETS, ARIMA, SBA, NF1/NF2 |
| SMOOTH/ERRATIC, weekly grain | `R_STAT_WEEKLY` | STL+ETS, DHR (Fourier, K by AICc), ETS, NF1/NF2 |
| SMOOTH/ERRATIC, daily grain | `R_STAT_DAILY` | multi-seasonal STL / DHR / Prophet-style + holiday dummies, NF2 (weekly lag) |
| SMOOTH/ERRATIC, monthly | `R_STAT` | ETS (AICc), ARIMA (AICc), NF1/NF2, combination |

11. **Governance tier (GS-G-09)** [RoT, configurable]:
    - AX/AY: weekly review, CSL tier 1, overrides allowed with reason codes.
    - AZ/BX/BY: weekly review, tier 2.
    - BZ/CX/CY: bi-weekly review, tier 3.
    - CZ / intermittent C: monthly review, tier 3, overrides disabled except data-error correction.
12. **Emit** the segmentation table and report (Section 6). Persist `seg_run_id` so downstream skills cite it.

## 5. Verification Snippets
```python
import numpy as np, pandas as pd

def classify(y: np.ndarray, m: int):
    y = np.asarray(y, float); nz = y[y > 0]
    if len(nz) < 2 or len(y) < m:
        return dict(segment="NEW_SPARSE")
    adi = len(y) / len(nz)
    cv2 = 0.0 if len(np.unique(nz)) == 1 else (nz.std(ddof=0) / nz.mean()) ** 2
    cls = ("SMOOTH" if cv2 < .49 else "ERRATIC") if adi < 1.32 else ("INTERMITTENT" if cv2 < .49 else "LUMPY")
    mu, var = y.mean(), y.var(ddof=0); vmr = var / mu if mu > 0 else np.nan
    p0_obs = (y == 0).mean()
    if vmr <= 1.2:
        dist, p0_exp = "POISSON", np.exp(-mu)
    else:
        r = mu**2 / (var - mu); dist, p0_exp = "NEG_BIN", (r / (r + mu)) ** r
    if p0_obs - p0_exp > 0.10:
        dist = "ZI_" + dist
    small = (y.max() <= 10) or (p0_obs >= 0.30)
    if not small and cls in ("SMOOTH", "ERRATIC") and (y.std(ddof=0) / mu) <= 0.5:
        dist = "NORMAL"
    s = np.mean(np.abs(y[m:] - y[:-m])) if len(y) > m else np.mean(np.abs(np.diff(y)))
    return dict(ADI=adi, CV2=cv2, demand_class=cls, VMR=vmr, zero_share=p0_obs,
                inv_distribution=dist, small_count=small, naive_scale=s, scale_zero=(s == 0))

def abc(values: pd.Series, a=.80, b=.95):
    v = values.sort_values(ascending=False); c = v.cumsum() / v.sum()
    prev = c.shift(fill_value=0)              # boundary-crossing item takes higher class
    return pd.Series(np.where(prev < a, "A", np.where(prev < b, "B", "C")), index=v.index)
```
```sql
-- ADI / CV² cross-check in SQL (population SD)
SELECT item_id, location_id,
       COUNT(*)::float / NULLIF(SUM(CASE WHEN demand_clean > 0 THEN 1 END),0)            AS adi,
       POWER(STDDEV_POP(CASE WHEN demand_clean > 0 THEN demand_clean END)
             / NULLIF(AVG(CASE WHEN demand_clean > 0 THEN demand_clean END),0), 2)       AS cv2
FROM demand_clean WHERE train_mask GROUP BY 1,2;
```
**Test vectors:**
- `y=[0,0,5,0,0,5,0,0,5]` → ADI = 3.0, CV² = 0 → INTERMITTENT.
- `y=[10,12,9,11,10,13,8,12,11,10,9,12]` with m = 12 → ADI = 1, CV² ≈ 0.02 → SMOOTH.

## 6. Standardized Output Format
```markdown
# Demand Pattern Segmentation — <scope> — seg_run_id <id>

## Executive Summary
- <N> item-locations classified (<P> periods, window <start–end>, grain <WEEK>); <x>% NEW_SPARSE, <y>% NO_DEMAND.
- Mix by count / by value: Smooth a%/a'%, Erratic b%/b'%, Intermittent c%/c'%, Lumpy d%/d'%.
- <k> series need count distributions (normal assumption prohibited, GS-M-08); <j> flagged obsolescence risk → TSB.
- Routing: R_STAT …, R_INTERMITTENT …, R_CAUSAL …, R_NEW_ANALOG ….

## Metadata
| Level | Grain | Time window | Weighting | Lag | Source run | Thresholds |
|---|---|---|---|---|---|---|
| SKU-Location | WEEK | <start–end> | counts & value (unit cost) | n/a (history) | cleansing run <id> | ADI 1.32 / CV² 0.49; ABC 80/95; XYZ CV 0.5/1.0 |

## Segment Matrix (count | value share)
| Class \ ABC | A | B | C | Total |
|---|---|---|---|---|
| Smooth | | | | |
| Erratic | | | | |
| Intermittent | | | | |
| Lumpy | | | | |
| NEW_SPARSE / NO_DEMAND | | | | |

## ABC × XYZ
| | X | Y | Z |
|---|---|---|---|
| A | | | |
| B | | | |
| C | | | |

## Item Detail (top N by value + all exceptions)
| item | loc | ADI | CV² | class | VMR | zero % | inv_distribution | seasonal F_S | ABC | XYZ | route | tier | flags |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|

## Exceptions
| ID | Trigger (rule) | item | loc | Detail | Value at risk | Action | Owner |
|---|---|---|---|---|---|---|---|
| | LOW_CONFIDENCE (>30% reconstructed, GS-M-11) | | | | | reconstruct before modeling | |
| | Boundary case (|ADI−1.32|<0.05 or |CV²−0.49|<0.05) | | | | | backtest both routes | |
| | Class migration vs. prior run (e.g., Smooth→Lumpy) | | | | | review demand change / data | |
| | OBSOLESCENCE_RISK (A or B value) | | | | | TSB + inventory-health review | |
| | SCALE_ZERO (constant series) | | | | | use WAPE; verify data | |

## Recommendations
1. …

## Data-Quality Log
| item | loc | periods excluded (structural) | % reconstructed | needs_model_recon % | notes |
|---|---|---|---|---|---|

## Reproducibility
seg_run_id, input run ids, thresholds, code version (§5), seed n/a (deterministic).
```

## 7. Edge Cases
- **Boundary values:** a value exactly at a cut-off goes to the ≥ side. Series within ±0.05 of either cut-off are flagged `BOUNDARY`, and both routes go to backtest.
- **Class changes over time:** also compute the class on the last 52 weeks (or 12 months). If it differs from the full-history class, use the recent class for routing and flag `REGIME_CHANGE`.
- **Heavily reconstructed history:** the classification still runs, but it carries `LOW_CONFIDENCE`. Never reclassify on raw (censored) sales.
- **Daily grain:** ADI/CV² at daily grain overstates intermittency. Also classify at weekly aggregate and report both, routing on the planning bucket.
- **Negative or zero unit cost:** exclude from the ABC value ranking and flag `COST_INVALID`.
