---
name: nextq-intermittent-demand-forecasting
description: >-
  [Next Quarter Supply Chain Suite] Forecasts intermittent, lumpy, slow-moving and obsolescence-prone demand (spare
  parts, MRO, long-tail SKUs, low-volume store-SKUs) with SBA (default), TSB, Croston (reference only), ADIDA and
  iMAPA temporal aggregation, and fits count / zero-inflated lead-time demand distributions for inventory. Load when
  the user asks to forecast items with many zero periods, slow movers, spare parts, sporadic or lumpy demand, to
  choose between Croston / SBA / TSB, to forecast end-of-life or dying items, or to get a lead-time demand
  distribution for a slow mover. Inputs: cleaned demand history and segmentation routes R_INTERMITTENT / R_TSB at
  SKU-Location-Week/Month (from nextq-demand-history-cleansing and nextq-demand-pattern-segmentation), lead time and
  review period. Triggers: "intermittent demand", "Croston", "SBA", "TSB", "lots of zeros", "slow moving", "spare
  parts forecast", "lumpy", "obsolete stock".
metadata:
  author: "Next Quarter"
  organization: "Next Quarter (nextq)"
  suite: "Next Quarter Supply Chain Intelligence"
  version: "1.0.0"
---

# Intermittent Demand Forecasting
*Part of the **Next Quarter (nextq)** Supply Chain Skill Suite — Governed by SUPPLY_CHAIN_GOLD_STANDARD.md.*

## 1. Role & Objective
**Persona:** Intermittent-Demand and Service-Parts Forecasting Specialist.
**Decision supported:**
- What is the expected demand rate per period for sparse items?
- Is demand dying (obsolescence)?
- What lead-time demand distribution should inventory policy use?

**Objective:** Produce per-series rate forecasts from the best-backtested intermittent estimator, an obsolescence signal, and a fitted lead-time demand distribution (quantiles and sample paths). Croston-type point forecasts are never used directly as normal-distribution means for safety stock.

**Scope:** routes `R_INTERMITTENT`, `R_TSB`, and the SBA arm of `R_STAT_PLUS_SBA`.

## 2. Gold Standard Rules Enforced
| Rule | Inlined standard |
|---|---|
| **GS-M-03** Classification | ADI $=\#\text{periods}/\#\{y_t>0\}$; CV² of non-zero sizes. Intermittent: ADI ≥ 1.32 & CV² < 0.49. Lumpy: ADI ≥ 1.32 & CV² ≥ 0.49. The cut-offs are **routing heuristics** for Croston vs. SBA (SBC 2005). Kostenko & Hyndman (2006) show SBA is better over most of the region, so **SBA is the default**. |
| **GS-M-06** Estimators [STD] | Notation: $\mathit{z}$ = smoothed non-zero size, $\mathit{p}$ = smoothed interval, $\mathit{q}$ = periods since last demand. **Croston (1972):** update only when $\mathit{y}_t>0$: $\mathit{z}_t=\alpha y_t+(1-\alpha)z_{t-1}$, $\mathit{p}_t=\alpha q+(1-\alpha)p_{t-1}$, forecast $\mathit{z}_t/p_t$. Positively biased (Syntetos & Boylan 2001), so **reference only**. **SBA (Syntetos & Boylan 2005):** $\hat y=(1-\alpha/2)\,z_t/p_t$, where $\alpha$ is the interval smoothing constant. **Default.** **TSB (Teunter, Syntetos & Babai 2011):** occurrence probability $\pi$ updated **every** period. If $\mathit{y}_t>0$: $\pi_t=\pi_{t-1}+\beta(1-\pi_{t-1})$ and $\mathit{z}_t=z_{t-1}+\alpha(y_t-z_{t-1})$. If $\mathit{y}_t=0$: $\pi_t=\pi_{t-1}+\beta(0-\pi_{t-1})$ and $\mathit{z}_t=z_{t-1}$. Forecast $\hat y=\pi_t z_t$. Unbiased and decays during zero runs, so it is **required** for obsolescence or end-of-life risk. **Parameters:** $\alpha,\beta\in[0.05,0.30]$ [RoT], optimized **out of sample**. Croston's original choice was α = 0.1. **No analytic prediction intervals** for Croston-type methods (Shenstone & Hyndman 2005): distributions come from GS-M-08 or bootstrap. |
| **GS-M-07** Temporal aggregation [STD] | **ADIDA:** aggregate into non-overlapping buckets of size $\mathit{k}$ (default $\mathit{k}=\text{round}(\text{ADI})$, min 2), forecast with SBA/SES, and disaggregate equally ($\hat y/k$ per base period). **iMAPA** (Petropoulos & Kourentzes 2015, JORS; R `tsintermittent::imapa`): levels $1..K$ ($\mathit{K}$ = min(12, ⌊n/4⌋) [RoT]). At each level, choose Croston/SBA/SES, then combine by mean (or median). |
| **GS-M-08** Count distributions | Never assume normal for small counts (max ≤ 10 per period or zero share ≥ 30% [RoT]). VMR ≈ 1 (0.8–1.2) → Poisson; VMR > 1.2 → NB with $\mathit{r}=\mu^2/(\sigma^2-\mu)$, $\mathit{p}=r/(r+\mu)$. Observed zero share exceeds the model's by > 10 pp → zero-inflated (ZIP/ZINB) or hurdle. Lumpy → compound Poisson-gamma (stuttering). Fit on **non-censored, non-structural** periods over the protection interval $\mathit{R}+L$. Check: observed vs. expected zero share and VMR. |
| **GS-M-10** Probabilistic | Lead-time demand quantiles from (i) the fitted distribution with the mean rate = selected forecast × $\left(R+L\right)$, or (ii) block-bootstrap of historical $\left(R+L\right)$-period sums scaled to the forecast rate. **Sum sample paths, not quantiles.** Score with pinball loss and coverage. |
| **GS-M-16** Rolling origin | Expanding-window origins; ≥ 3 origins, or 1 full seasonal cycle of origins [RoT]. Score per lag. Parameters are re-optimized using only data ≤ origin. |
| **GS-K-04** MASE / RMSSE | MASE scale $=\frac{1}{T-1}\sum\lvert y_t-y_{t-1}\rvert$ (m = 1; intermittent series are not seasonally modeled). **Selection metric: RMSSE** $=\sqrt{\text{mean}(e^2)/\text{mean}((y_t-y_{t-1})^2)}$, because **MAE-type metrics reward an all-zero forecast** on intermittent data. RMSSE targets the mean rate. MASE, Bias%, and pinball are reported alongside. **MAPE is prohibited** (GS-K-02). |
| **GS-K-05** Bias | $\text{Bias\%}=\sum(F-A)/\sum A\times100$ (positive = over-forecast). Gate: \|Bias%\| ≤ 15% over the evaluation window [RoT; wider than the smooth-series ±10% because sparse windows are noisy]. |
| **GS-K-07** FVA | The chosen method must beat NF1 and a **zero-or-mean benchmark**: the in-sample mean rate (an MA over the full window, which is the naïve rate) on RMSSE. Otherwise flag `NAIVE_WINS` and use the mean rate. |
| **GS-K-14** Coverage | 80%/95% lead-time intervals within ±5 pp of nominal [RoT]. |
| **GS-G-03/10** | Exceptions and standard output contract. |

**Obsolescence rule** (feeds `nextq-inventory-health-optimization`) [RoT]:
- **Obsolescence risk:** TSB $\pi_T < 0.5 \times \pi$ (12 months, or 52 weeks, earlier), OR trailing zero run ≥ max(3 × ADI, ¼ seasonal cycle).
- **Dead:** TSB $\pi_T < 0.02$.

## 3. Required Inputs & Data Schema
**`demand_clean`** (from `nextq-demand-history-cleansing`): `item_id, location_id, period_start, demand_clean (≥0, base UoM), train_mask, period_state, needs_model_recon, lifecycle_status`.

**`segmentation`** (from `nextq-demand-pattern-segmentation`): `route ∈ {R_INTERMITTENT, R_TSB, R_STAT_PLUS_SBA}, ADI, CV2, VMR, zero_share, inv_distribution, OBSOLESCENCE_RISK, ABC`.

**`policy_params`** (optional; needed for lead-time distributions): `item_id, location_id, lead_time_periods L (decimal), lead_time_sd σ_L, review_period R`.

**Run parameters:**
- `grain` (WEEK default; MONTH allowed), `horizon_H`, `decision_lag`, `n_origins`, `alpha_grid = beta_grid = {0.05, 0.10, 0.15, 0.20, 0.25, 0.30}`, `quantiles`, `seed`.

**Minimum history:** ≥ 2 non-zero periods **and** ≥ 1 full seasonal cycle (52 weeks / 12 months). Recommended: ≥ 8 non-zero demands for stable parameters. With fewer than 8, fix α = β = 0.1 (Croston's default) instead of optimizing, and flag `FEW_DEMANDS`.

**Censoring:** periods with `needs_model_recon = true` are **excluded from parameter optimization and scoring** (treated as missing). Zero sales during stock-out are not true zeros. They still advance the TSB clock, but no update is made: π and z are held. If > 30% of periods are censored → `LOW_CONFIDENCE`.

## 4. Execution Workflow
1. **Gate.**
   - Confirm the route.
   - Apply the history minimum; send `NEW_SPARSE` back to `R_NEW_ANALOG`.
   - All zeros in the last 2 cycles → forecast 0 with `DEAD` flag and route to inventory-health.
2. **Initialization (deterministic):**
   - Croston/SBA: $\mathit{z}_0$ = first non-zero size; $\mathit{p}_0$ = index (1-based) of the first non-zero period.
   - TSB: $\pi_0$ = non-zero share in the first $\min(n, m)$ periods; $\mathit{z}_0$ = mean of non-zero sizes in that window.
   - The initialization window is excluded from scoring.
3. **Candidates:**
   - NF1, MEAN-rate, Croston (reference), SBA($\alpha$), TSB($\alpha,\beta$), ADIDA(k)+SBA, iMAPA (mean combination).
   - Add SES($\alpha$) if `R_STAT_PLUS_SBA`.
4. **Rolling-origin backtest.**
   - For each origin, grid-search parameters on data ≤ origin (in-sample one-step RMSSE on the last ⅓ of the training window, which is an internal holdout [decision]).
   - Forecast $\mathit{h}=1..H_{\mathrm{bt}}$. Intermittent methods give flat forecasts, so the per-lag score differs only by the actuals.
5. **Score:** RMSSE (selection), MASE, Bias%, pinball (P50/P90 from the GS-M-08 distribution), and portfolio WAPE (reported; computed on aggregate sums).
6. **Select:**
   - Pick the lowest mean RMSSE across origins.
   - **Gates:** beats NF1 **and** MEAN on RMSSE, and |Bias%| ≤ 15%.
   - Tie (< 1% relative) → prefer SBA, then TSB, then ADIDA, then iMAPA (simplicity).
   - **Obsolescence override:** if `OBSOLESCENCE_RISK` or route `R_TSB`, select **TSB** unless another method beats it by > 5% RMSSE **and** has a lower |Bias%| [RoT].
7. **Final fit** on all valid history, then forecast the rate $\hat y$ for $\mathit{h}=1..H$ (flat; ADIDA/iMAPA may carry level structure).
8. **Distribution (GS-M-08):**
   - Fit Poisson/NB/ZI/compound on the valid history.
   - Rescale the mean to $\hat y$ per period, keeping the dispersion (NB $\mathit{r}$, or the ratio $\sigma^2/\mu$) from history.
   - Lead-time demand over $\mathit{R}+L$: sum $\mathit{R}+L$ independent draws, or bootstrap blocks. When $\sigma_L>0$, also draw $\mathit{L}$ from its distribution.
   - Output P50/P80/P90/P95/P99 and 1,000 seeded sample paths.
9. **Obsolescence metrics:** current $\pi_T$, $\pi$ a year earlier, trailing zero run, flags.
10. **Output** (Section 6). Persist `forecast_version = INT_<run_id>` with lag snapshots for FVA.

## 5. Verification Snippets (reference implementations)
```python
import numpy as np

def croston_core(y, alpha, alpha_p=None):
    """Returns per-period one-step forecasts (z/p) and final (z, p). Update only on demand."""
    alpha_p = alpha if alpha_p is None else alpha_p
    y = np.asarray(y, float); nz = np.flatnonzero(y > 0)
    if len(nz) == 0: return np.zeros(len(y)), (0.0, np.inf)
    z, p, q = y[nz[0]], nz[0] + 1.0, 1
    f = np.full(len(y), np.nan)
    for t in range(nz[0] + 1, len(y)):
        f[t] = z / p
        if y[t] > 0:
            z = alpha * y[t] + (1 - alpha) * z
            p = alpha_p * q + (1 - alpha_p) * p
            q = 1
        else:
            q += 1
    return f, (z, p)

def croston(y, alpha=0.1):   # reference only (positively biased)
    f, (z, p) = croston_core(y, alpha); return f, z / p

def sba(y, alpha=0.1):       # GS default: (1 - alpha/2) * z/p ; alpha = interval smoothing constant
    f, (z, p) = croston_core(y, alpha); k = 1 - alpha / 2
    return f * k, k * z / p

def tsb(y, alpha=0.1, beta=0.1, init_window=None, valid=None):
    """valid: bool mask; False (censored) periods hold state (no update)."""
    y = np.asarray(y, float); n = len(y)
    valid = np.ones(n, bool) if valid is None else np.asarray(valid, bool)
    w = init_window or min(n, 12)
    nzw = y[:w][y[:w] > 0]
    pi = (y[:w] > 0).mean(); z = nzw.mean() if len(nzw) else (y[y > 0].mean() if (y > 0).any() else 0.0)
    f = np.full(n, np.nan)
    for t in range(n):
        f[t] = pi * z
        if not valid[t]: continue
        if y[t] > 0:
            pi += beta * (1 - pi); z += alpha * (y[t] - z)
        else:
            pi += beta * (0 - pi)
    return f, pi * z, pi

def adida(y, k, base=sba, **kw):
    y = np.asarray(y, float); n = (len(y) // k) * k
    agg = y[len(y) - n:].reshape(-1, k).sum(1)        # align buckets to the most recent period
    _, f_agg = base(agg, **kw)
    return f_agg / k

def rmsse(y_true, y_pred, y_train):
    d = np.mean(np.diff(np.asarray(y_train, float)) ** 2)
    return np.sqrt(np.mean((np.asarray(y_true) - np.asarray(y_pred)) ** 2) / d) if d > 0 else np.nan
```
```python
# GS-M-08 distribution selection and lead-time quantiles (Poisson / NB), seeded
# numpy only (no scipy needed)
def lead_time_quantiles(y_hist, rate, R_plus_L, qs=(.5,.8,.9,.95,.99), n=10000, seed=42):
    y = np.asarray(y_hist, float); mu, var = y.mean(), y.var(ddof=0)
    rng = np.random.default_rng(seed); vmr = var / mu if mu > 0 else 1.0
    mean_LT = rate * R_plus_L
    if vmr <= 1.2:
        draws = rng.poisson(mean_LT, n); dist = "POISSON"
    else:
        r_per = mu**2 / (var - mu)                    # NB size per period (dispersion from history)
        r = r_per * R_plus_L                          # sum of i.i.d. NB with common p -> NB(r*n, p)
        p = r / (r + mean_LT); draws = rng.negative_binomial(r, p, n); dist = "NEG_BIN"
    return dist, {q: float(np.quantile(draws, q)) for q in qs}
```
**Test vectors** (verified against the code above):
- `y=[0,0,3,0,0,3,0,0,3,0,0,3]` with α = 0.1: Croston final rate = 1.0 (z = 3, p = 3), and SBA = 0.95.
- TSB on the same series with α = β = 0.1 and a 12-period init window gives final $\pi z ≈ 1.077$ ($\pi ≈ 0.359$, $\mathit{z} = 3$). Appending one zero gives ≈ 0.969.
- The same `y` followed by 26 zeros: the TSB rate decays by $\left(1-\beta\right)^{26} ≈ 0.065×$ for β = 0.1, while SBA stays at 0.95. This is the obsolescence check.
- `adida(y, k=3)` = 0.95 per base period.

## 6. Standardized Output Format
```markdown
# Intermittent Demand Forecast — <scope> — run INT_<run_id>

## Executive Summary
- <N> intermittent/lumpy series (<x>% of items, <y>% of value); method mix SBA a%, TSB b%, ADIDA c%, iMAPA d%, Mean e%.
- vs NF1: RMSSE <r1> vs <r0>; Bias <b>% (<over/under>); 90% lead-time PI coverage <c>%.
- <k> series OBSOLESCENCE_RISK, <d> DEAD (TSB π<0.02) → <units>/<value> inventory exposure.
- Normal-distribution safety stock prohibited for <s> series (GS-M-08): distribution file attached.

## Metadata
| Level | Grain | Lag | Time window | Origins | Weighting | Selection metric | Actuals basis |
|---|---|---|---|---|---|---|---|
| SKU-Location | WEEK | <ℓ> | <start–end> | <n> | RMSSE unweighted mean; WAPE volume-weighted | RMSSE | unconstrained; censored periods excluded (<p>%) |

## Accuracy by Method (portfolio, lag ℓ)
| Method | RMSSE (mean) | MASE (median) | Bias % (+=over) | Pinball P90 | 90% PI coverage | WAPE % (agg) | # selected |
|---|---|---|---|---|---|---|---|
| NF1 | | | | | | | |
| Mean rate | | | | | | | |
| Croston (ref) | | | | | | | — |
| SBA | | | | | | | |
| TSB | | | | | | | |
| ADIDA | | | | | | | |
| iMAPA | | | | | | | |

## Series Detail
| item | loc | class | ADI | CV² | method | α | β | rate/period | π_T | zero run | distribution | LT P50 | LT P95 | RMSSE | Bias % | flags |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|

## Exceptions
| ID | Trigger (rule) | item | loc | value | action | owner |
|---|---|---|---|---|---|---|
| | NAIVE_WINS (GS-K-07) | | | | use mean rate; review | |
| | \|Bias\|>15% (GS-K-05) | | | | check censoring/regime | |
| | OBSOLESCENCE_RISK / DEAD (TSB) | | | | inventory-health review, stop replenishment? | |
| | Coverage off >5pp (GS-K-14) | | | | refit distribution / bootstrap | |
| | FEW_DEMANDS (<8 non-zero) | | | | fixed α=β=0.1; low confidence | |
| | LOW_CONFIDENCE (>30% censored) | | | | reconstruct demand (C2) | |

## Recommendations
1. …

## Data-Quality Log
| item | loc | censored periods excluded | structural excluded | init window | notes |

## Reproducibility
Parameters (grids, K, k for ADIDA, seed), input run ids, code §5, library versions.
```

## 7. Edge Cases
- **Single repeated size** (CV² = 0): Croston/SBA are well defined, and NB is undefined if VMR ≤ 1, so use Poisson.
- **Demand ramping up** (ADI falling over time): reclassify on the last 52 weeks. If the series is now smooth, re-route to `nextq-statistical-baseline-forecasting` (`REGIME_CHANGE`).
- **Seasonal intermittent items** (for example, winter spare parts): use ADIDA at the seasonal bucket or a seasonal-profile multiplier on the SBA rate. Validate by backtest, and flag `SEASONAL_INTERMITTENT`.
- **One huge order dominates:** if one size is > 50% of total volume, confirm with cleansing whether it is `BULK_ONE_OFF`. If confirmed, exclude it from size smoothing but keep it as an occurrence.
- **Lead time shorter than one bucket:** compute lead-time demand at a finer grain if available. Otherwise use a fractional $\mathit{R}+L$ (the Poisson mean scales linearly) and flag it.
- **Monthly grain with ADI near 1:** prefer weekly classification. Monthly aggregation often hides intermittency; report both.
