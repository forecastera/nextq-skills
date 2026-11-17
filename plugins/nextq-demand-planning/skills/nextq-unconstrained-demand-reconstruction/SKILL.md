---
name: nextq-unconstrained-demand-reconstruction
description: >-
  [Next Quarter Supply Chain Suite] Rebuilds true (unconstrained) demand where sales were truncated by stockouts,
  allocations, order cuts or short-ships, completing the reconstruction nextq-demand-history-cleansing starts: keeps
  its orders and availability-scaling results, then estimates remaining censored periods with Tobit (right-censored
  Normal regression) or censored Poisson EM / negative-binomial likelihood for low counts, imputing conditional
  expectations, with model fallback, substitution netting, masking validation, lost-sales estimates and full lineage.
  Load when the user asks to estimate lost sales, unconstrain or de-censor demand, correct history for stockouts or
  allocations, run EM or Tobit on censored sales, or check how much history was reconstructed. Inputs: cleaned history
  with period states and needs_model_recon flags, availability, orders, backorders, cuts, substitute groups. Triggers:
  "lost sales", "unconstrained demand", "stockout correction", "censored demand", "Tobit", "EM".
metadata:
  author: "Next Quarter"
  organization: "Next Quarter (nextq)"
  suite: "Next Quarter Supply Chain Intelligence"
  version: "1.0.0"
---

# Unconstrained Demand Reconstruction
*Part of the **Next Quarter (nextq)** Supply Chain Skill Suite — Governed by SUPPLY_CHAIN_GOLD_STANDARD.md.*

## 1. Role & Objective
**Persona:** Demand Data Scientist / Demand Signal Owner.
**Decision supported:** What would customers have bought if product had been available? How much demand was lost, and how much of it moved to substitutes? Is the reconstructed history good enough to train forecasts and set inventory policy?

**Objective:**
- Complete GS-M-11 for every censored period: methods (a), (b), (c), (d) in priority order, consistent with `nextq-demand-history-cleansing`.
- Produce an unconstrained `demand_clean` that downstream skills train on and score against.
- Net substitution so category demand is not double counted, and quantify lost sales.
- Validate the method by masking, and log every change (GS-D-13).

**Division of labour with `nextq-demand-history-cleansing`:**
- Cleansing detects censoring, assigns period states, and applies **only** (a) orders and (b) availability scaling (`INTERIM_RECON`).
- Rows it cannot resolve carry `needs_model_recon = true`. This skill resolves them with (a′)/(c)/(d) and leaves (a)/(b) results untouched.
- Both skills use identical thresholds: material censoring at in-stock < 95%, ×2 cap, $\mathit{a}_t<0.5$ routed to (c), and LOW_CONFIDENCE above 30% reconstructed.

## 2. Gold Standard Rules Enforced
| Rule | Inlined standard |
|---|---|
| **GS-M-11** Censoring & reconstruction (mandatory) | Recorded sales $=\min(D_t,\text{available}_t)$. Training on raw sales during stockouts biases forecasts down and creates a self-reinforcing under-stocking loop. **Detect:** censored if on-hand ≤ 0 (start or end) OR in-stock < 100% OR backorders > 0 OR order cut / short-ship > 0. **Material** if in-stock < 95% [RoT]. **Reconstruct, first applicable:** **(a)** orders (requested qty and date) when recorded. **(b)** availability scaling $\hat D_t=S_t/a_t$ (profile-weighted in-stock fraction), uplift capped at ×2; if $\mathit{a}_t<0.5$ use (c) [RoT]. **(c)** model-based: treat $\mathit{S}_t$ as right-censored and estimate by **EM or Tobit-type likelihood**; for substitutable assortments, EM with choice models (Vulcano, van Ryzin & Ratliff 2012). **(d)** last resort: fitted / interpolated value, marked imputed. **Document** the share reconstructed per SKU; > 30% → **LOW_CONFIDENCE** [RoT]. **Substitution:** net reconstructed demand against measurable substitute uplift (no double count). **KPI rule:** score accuracy against unconstrained actuals; report constrained (sales) accuracy separately. |
| **GS-D-02** States | Only CENSORED periods are reconstructed. TRUE_ZERO stays 0, STRUCTURAL_ZERO stays masked, and MISSING is imputed by cleansing (GS-D-03), never by this skill. |
| **GS-D-06** Substitution / cannibalization | Substitute uplift during the out-of-stock window $\mathit{U}_j=(A_j-B_j)^+$ over the same periods (same need-state, substitute in stock). Transferred demand $\mathit{T}=\min(L_i,\sum_jU_j)$, where $\mathit{L}_i=\sum(\hat D_i-S_i)$ is the lost demand of item $\mathit{i}$. |
| **GS-M-08** Count demand | Low counts (mean < 10 or many zeros [RoT]) must not use a Normal model. Use censored Poisson (VMR ≈ 1) or negative binomial (VMR > 1) likelihoods. Structural zeros are excluded from fitting; sampling zeros are kept. |
| **GS-D-13** Lineage | Every reconstructed value is logged with **before, after, rule ID, method, parameters, timestamp**. Raw data are immutable. |
| **GS-K-09** Fill rate | Unit fill rate $=$ units shipped from stock on first pass / units **demanded**. The denominator uses unconstrained (reconstructed) demand. This skill publishes the reconstructed denominator for `nextq-service-level-performance-analysis`. |
| **GS-K-13** Reporting | Disclose % of periods reconstructed (by method), % censored, and excluded zeros, next to every accuracy figure that uses this output. |
| **GS-G-10 / GS-G-11** | Standard output; masking validation makes the reconstruction reproducible and testable. |

**Model-based (c) specification:**
- **Engine routing [RoT]** (GS-M-08) on exact (uncensored) observations:
  - count model if mean < 10 or zero share > 30%: `EM_POISSON` if 0.8 ≤ VMR ≤ 1.2, else `EM_NB`
  - otherwise `TOBIT`
- **Tobit:** $\mathit{D}_t\sim N(x_t'\beta,\sigma^2)$.
  - Exact rows contribute $\log\phi\big((y_t-x_t'\beta)/\sigma\big)-\log\sigma$. Censored rows contribute $\log\big(1-\Phi((S_t-x_t'\beta)/\sigma)\big)$.
  - Impute the **conditional expectation** $\mathit{E}[D\mid D\ge S]=\mu+\sigma\,\phi(\alpha)/(1-\Phi(\alpha))$, with $\alpha=(S-\mu)/\sigma$.
  - Default regressors: intercept, Fourier seasonal terms (K = 2 at m = 52 when ≥ 2 cycles, else none), promo flag, log price index.
- **Censored Poisson EM:** $\mu_t=\lambda w_t$, where $\mathit{w}_t$ is the seasonal/promo profile (mean 1) from `nextq-statistical-baseline-forecasting`.
  - E-step: $\mathit{E}[D\mid D\ge s]=\mu\,P(D\ge s-1)/P(D\ge s)$.
  - M-step: $\lambda=\sum\hat D_t/\sum w_t$. Iterate to a relative change below $10^{-10}$.
- **Censored NB2:** maximum likelihood with mean $\lambda w_t$ and size $\mathit{k}$. Censored rows contribute $\log P(D\ge S)$. Impute $\mathit{E}[D\mid D\ge S]$ by summing the pmf to the $1-10^{-9}$ quantile.
- **Exact observations** for fitting are OBSERVED, TRUE_ZERO and ORDERS rows. AVAIL_SCALE and model rows enter as **censored at their sales** (their (b) values stay in the output but are not treated as exact).
- **Floor:** $\hat D_t\ge S_t$ always (asserted).
- **Review flag** `RECON_LARGE` when $\hat D_t>3S_t$ with $\mathit{S}_t>0$ [RoT]. This is not a cap: a period with almost no availability can legitimately triple.

## 3. Required Inputs & Data Schema
**`demand_clean`** (output of `nextq-demand-history-cleansing`, one item-location, sorted):

| Column | Use |
|---|---|
| `item_id, location_id, channel, period_start, period_grain` | keys |
| `demand_raw_base` (= sales $\mathit{S}_t$), `demand_clean` | lower bound; current value |
| `period_state` ∈ {OBSERVED, TRUE_ZERO, STRUCTURAL_ZERO, MISSING→imputed, CENSORED} | GS-D-02 |
| `censored_material`, `recon_method` ∈ {NULL, ORDERS, AVAIL_SCALE}, `needs_model_recon`, `imputed_flag`, `train_mask` | cleansing hand-off |
| `on_hand_qty, in_stock_pct, backorder_qty, order_cut_qty, orders_qty_base` | evidence |
| `promo_flag, price, list_price, event_flags` | regressors |

**`profile`**: `item_id, location_id, period_start, w_t` (seasonal × promo profile, mean 1) from the baseline skill. If absent, use $\mathit{w}_t=1$ and flag `FLAT_PROFILE`.

**`substitute_groups`**: `item_id, need_state_id`, plus substitute baselines $\mathit{B}_j$ and actuals $\mathit{A}_j$ for the out-of-stock periods.

**Minimum:** ≥ 8 exact observations per series for (c) [RoT]; otherwise (d). ≥ 2 seasonal cycles for Fourier terms.

## 4. Execution Workflow
1. **Re-verify the hand-off:**
   - Recompute the censoring test with the same rule as cleansing and assert agreement: any disagreement → `HANDOFF_MISMATCH` (stop for that series).
   - Recompute (a)/(b) for rows with `recon_method` set and assert equality (tolerance $10^{-9}$).
   - This skill never overwrites (a)/(b) results.
2. **Resolve (a′) implied orders:** rows with `needs_model_recon` where backorder or cut > 0, $\mathit{a}_t$ unknown or = 1, and no order data: $\hat D_t=S_t+\text{backorder}+\text{cut}$ (`ORDERS_IMPLIED`). When $\mathit{a}_t<0.5$, shortage logs undercount walk-away demand, so go to (c).
3. **Engine routing** on exact observations (Section 2).
4. **Fit (c):**
   - Tobit, Poisson-EM or NB-MLE per series. Seasonal/promo structure comes through $\mathit{X}$ (Tobit) or $\mathit{w}_t$ (counts).
   - The fit fails or there are fewer than 8 exact observations → (d).
5. **Impute** each remaining censored row with $\mathit{E}[D\mid D\ge S_t]$ under the fitted model (`TOBIT`, `EM_POISSON`, `EM_NB`). Assert $\hat D_t\ge S_t$.
6. **(d) Fallback:** $\hat D_t=\max(\hat\mu_t,S_t)$ from the baseline model's fitted value (`IMPUTED_MODEL`), with `imputed_flag = true`.
7. **Substitution netting** per out-of-stock episode (contiguous censored periods):
   - Lost $\mathit{L}_i=\sum(\hat D_i-S_i)$ and substitute uplift $\mathit{U}_j$ (in-stock substitutes only).
   - $\mathit{T}=\min(L_i,\sum U_j)$, allocated pro-rata to $\mathit{U}_j$.
   - **Item $\mathit{i}$ keeps $\hat D_i$** (what customers wanted of $\mathit{i}$). **Each substitute's history is reduced by its allocation** (`SUBSTITUTE_DEFLATED`, logged), so category demand is not double counted.
   - Category net lost sales $=L_i-T$.
   - Where choice-model data exist (assortment sets, transaction-level choices), the Vulcano et al. (2012) EM is the preferred estimator. It is not implemented in §5 and must be run as a separate documented model.
8. **Validation (masking):**
   - On clean periods, censor about 25% at random caps (30–90% of demand, seeded), reconstruct, and report bias % and WAPE % vs. truth, alongside the bias of using raw sales.
   - Accept the engine if $\lvert\text{bias}_{\text{recon}}\rvert<\lvert\text{bias}_{\text{raw}}\rvert$ and $\lvert\text{bias}_{\text{recon}}\rvert\le10\%$ [RoT]. Otherwise drop to the next engine (NB → Poisson → (d)) and flag it.
9. **Share and confidence:**
   - `pct_reconstructed` = periods with any method (a, a′, b, c, d) / non-structural periods (same definition as cleansing).
   - > 30% → `LOW_CONFIDENCE`.
   - Set `needs_model_recon = false` on resolved rows.
10. **Lineage and publish:** write before/after/rule/method/params/timestamp rows; publish `demand_clean` (unconstrained), `lost_sales`, and substitute deductions.
11. **Output** (Section 6).

## 5. Verification Snippets
```python
import numpy as np, pandas as pd
from scipy import optimize, stats

CENSOR_MATERIAL, CAP, MIN_AVAIL, LOW_CONF = 0.95, 2.0, 0.5, 0.30      # identical to nextq-demand-history-cleansing

def detect_censoring(df):
    """GS-M-11 detection, same rule as nextq-demand-history-cleansing Step 4.3."""
    cens = (df.on_hand_qty.le(0) | df.in_stock_pct.lt(1.0) |
            df.backorder_qty.fillna(0).gt(0) | df.order_cut_qty.fillna(0).gt(0))
    return cens, cens & df.in_stock_pct.lt(CENSOR_MATERIAL)

def recon_ab(sales, orders, a, backorder=0.0, cut=0.0):
    """(a) orders; (a') shipped + backorder + cut when only shortage logs exist; (b) S/a capped x2 for 0.5 <= a < 1;
    else (None, None) -> model (c)/(d). (a)/(b) reproduce nextq-demand-history-cleansing Step 5 exactly."""
    if orders is not None and np.isfinite(orders):
        return orders, "ORDERS"
    if a is not None and MIN_AVAIL <= a < 1:
        return min(sales / a, CAP * sales), "AVAIL_SCALE"
    if (backorder or 0) + (cut or 0) > 0 and (a is None or a >= 1):
        return sales + (backorder or 0) + (cut or 0), "ORDERS_IMPLIED"
    return None, None

# ---------- (c) model-based: Tobit (continuous) ----------
def tobit_fit(y, cens, X):
    """Right-censored Normal regression: D ~ N(X b, s^2); for censored rows only D >= y is known."""
    y, cens, X = np.asarray(y, float), np.asarray(cens, bool), np.asarray(X, float)
    def nll(theta):
        b, ls = theta[:-1], theta[-1]; s = np.exp(ls); mu = X @ b
        ll_u = stats.norm.logpdf(y[~cens], mu[~cens], s)
        ll_c = stats.norm.logsf(y[cens], mu[cens], s)
        return -(ll_u.sum() + ll_c.sum())
    b0, *_ = np.linalg.lstsq(X[~cens], y[~cens], rcond=None)
    s0 = np.std(y[~cens] - X[~cens] @ b0) + 1e-6
    res = optimize.minimize(nll, np.r_[b0, np.log(s0)], method="L-BFGS-B")
    return res.x[:-1], float(np.exp(res.x[-1])), bool(res.success)

def tobit_cond_mean(s, mu, sigma):
    """E[D | D >= s] = mu + sigma * phi(al) / (1 - Phi(al)), al = (s - mu)/sigma (inverse Mills ratio)."""
    al = (np.asarray(s, float) - mu) / sigma
    return mu + sigma * np.exp(stats.norm.logpdf(al) - stats.norm.logsf(al))

# ---------- (c) model-based: censored count EM / MLE (GS-M-08 low counts) ----------
def poisson_censored_em(y, cens, w, tol=1e-10, max_iter=500):
    """lambda_t = lam * w_t (w = seasonal/exposure profile). E-step: E[D|D>=s] = mu * P(D>=s-1)/P(D>=s)."""
    y, cens, w = np.asarray(y, float), np.asarray(cens, bool), np.asarray(w, float)
    lam = y.sum() / w.sum()
    for _ in range(max_iter):
        mu = lam * w
        ed = y.copy()
        s = y[cens]
        ed[cens] = mu[cens] * stats.poisson.sf(s - 2, mu[cens]) / np.maximum(stats.poisson.sf(s - 1, mu[cens]), 1e-300)
        new = ed.sum() / w.sum()
        if abs(new - lam) < tol * max(lam, 1):
            lam = new; break
        lam = new
    mu = lam * w
    ed = y.copy()
    ed[cens] = mu[cens] * stats.poisson.sf(y[cens] - 2, mu[cens]) / np.maximum(stats.poisson.sf(y[cens] - 1, mu[cens]), 1e-300)
    return lam, ed

def nb_censored_fit(y, cens, w):
    """NB2 with mean mu_t = lam*w_t, size k; censored rows contribute log P(D >= y)."""
    y, cens, w = np.asarray(y, float), np.asarray(cens, bool), np.asarray(w, float)
    def nll(th):
        lam, k = np.exp(th); mu = lam * w; p = k / (k + mu)
        return -(stats.nbinom.logpmf(y[~cens], k, p[~cens]).sum() + stats.nbinom.logsf(y[cens] - 1, k, p[cens]).sum())
    res = optimize.minimize(nll, np.log([max(y.mean(), 0.1), 5.0]), method="Nelder-Mead", options=dict(xatol=1e-8, fatol=1e-10, maxiter=4000))
    lam, k = np.exp(res.x); mu = lam * w; p = k / (k + mu)
    ed = y.copy()
    for i in np.where(cens)[0]:
        grid = np.arange(int(y[i]), int(stats.nbinom.ppf(1 - 1e-9, k, p[i])) + 2)
        pm = stats.nbinom.pmf(grid, k, p[i]); ed[i] = (grid * pm).sum() / pm.sum()
    return lam, k, ed

def choose_engine(y_uncens):
    """GS-M-08 routing [RoT]: counts if mean < 10 or zero share > 30%; Poisson if 0.8 <= VMR <= 1.2 else NB; else Tobit."""
    y = np.asarray(y_uncens, float); m = y.mean(); v = y.var(ddof=1)
    if m < 10 or (y == 0).mean() > 0.30:
        return "EM_POISSON" if 0.8 <= v / max(m, 1e-9) <= 1.2 else "EM_NB"
    return "TOBIT"

# ---------- substitution netting (GS-M-11 step 4 / GS-D-06) ----------
def net_substitution(lost_i, sub_uplift):
    """Transferred T = min(lost_i, sum_j uplift_j^+), allocated to substitutes pro-rata.
    Item i keeps its unconstrained demand; each substitute's history is reduced by its transferred share."""
    up = {j: max(u, 0.0) for j, u in sub_uplift.items()}; tot = sum(up.values())
    T = min(max(lost_i, 0.0), tot)
    alloc = {j: (T * u / tot if tot > 0 else 0.0) for j, u in up.items()}
    return T, alloc, max(lost_i, 0.0) - T          # transferred, per-substitute deduction, true category lost sales

def masking_validation(y_true, cap_frac, seed=0, engine="EM_POISSON", w=None):
    """Artificially censor clean periods (sales = min(D, cap)) and measure recovery bias / WAPE."""
    rng = np.random.default_rng(seed); y_true = np.asarray(y_true, float); n = len(y_true)
    w = np.ones(n) if w is None else np.asarray(w, float)
    pick = rng.random(n) < cap_frac
    cap = np.floor(y_true * rng.uniform(0.3, 0.9, n))
    sales = np.where(pick, np.minimum(y_true, cap), y_true)
    cens = pick & (cap < y_true)
    _, ed = poisson_censored_em(sales, cens, w) if engine == "EM_POISSON" else (None, nb_censored_fit(sales, cens, w)[2])
    m = cens
    return dict(n_censored=int(m.sum()),
                bias_raw_pct=100 * (sales[m].sum() - y_true[m].sum()) / y_true[m].sum(),
                bias_recon_pct=100 * (ed[m].sum() - y_true[m].sum()) / y_true[m].sum(),
                wape_recon_pct=100 * np.abs(ed[m] - y_true[m]).sum() / y_true[m].sum())
```
**Test vector** (verified):
- **Detection** on 4 rows (on-hand 5/0/3/2, in-stock 1/0.4/0.97/1, backorder 0/0/0/1): censored = [F, T, T, T]; material = [F, T, F, F].
- **(a)/(b)/(a′):**
  - S = 50, a = 0.8 → 62.5 `AVAIL_SCALE`.
  - a = 1.0 with backorder 7 → 57 `ORDERS_IMPLIED`.
  - a = 0.4 → model (c).
  - orders 42 → 42 `ORDERS`.
- **Tobit** (seed 1): 104 weeks, $\mathit{D}=100+20\sin-10\cos+N(0,8^2)$, 20 periods censored at 60–95% of demand.
  - Estimates β = [99.62, 20.21, −9.63] (true [100, 20, −10]) and σ = 6.80. Naïve OLS on sales gives intercept 95.16.
  - Censored-period total: raw sales bias **−23.3%** → Tobit conditional mean **+0.7%**.
- **Poisson EM:** y = [3,5,2,4,1,4,3,6] with periods 3 and 5 censored → λ = 4.2243. Imputed 2 → 4.507 and 1 → 4.287.
- **Masking validation** (seed 3, 25% censored):
  - Poisson(6), 104 weeks: raw bias −46.8% → reconstructed +7.4% (WAPE 25.6%).
  - NB (mean 6, size 3): raw bias −50.6% → reconstructed +5.7% (WAPE 44.1%; high dispersion limits per-period accuracy).
  - Both pass the acceptance rule.
- **Routing:** Poisson sample → `EM_POISSON`; mean-100 series → `TOBIT`.
- **Substitution netting:**
  - Lost 40 with uplift {J1 15, J2 10, J3 −5} → T = 25 (J1 15, J2 10, J3 0); category lost = 15.
  - Lost 10 → T = 10 (J1 6, J2 4); category lost = 0.

```sql
-- Hand-off check: censoring rule must match cleansing; list rows still needing model reconstruction
SELECT item_id, location_id, period_start, demand_raw_base AS sales, in_stock_pct, backorder_qty, order_cut_qty,
       (on_hand_qty <= 0 OR in_stock_pct < 1 OR COALESCE(backorder_qty,0) > 0 OR COALESCE(order_cut_qty,0) > 0) AS censored_recheck,
       period_state = 'CENSORED'                                                                                 AS censored_cleansing,
       needs_model_recon,
       CASE WHEN COALESCE(backorder_qty,0) + COALESCE(order_cut_qty,0) > 0 AND COALESCE(in_stock_pct,1) >= 1
            THEN 'ORDERS_IMPLIED' ELSE 'MODEL_C' END                                                           AS planned_method
FROM demand_clean
WHERE needs_model_recon
   OR (on_hand_qty <= 0 OR in_stock_pct < 1 OR COALESCE(backorder_qty,0) > 0 OR COALESCE(order_cut_qty,0) > 0)
      <> (period_state = 'CENSORED');                                  -- any row here with a mismatch = HANDOFF_MISMATCH
```

## 6. Standardized Output Format
```markdown
# Unconstrained Demand Reconstruction — <scope> — run RECON_<run_id>

## Executive Summary
- <x>% of item-location-periods censored (<y>% material); reconstructed by method: ORDERS <a>%, AVAIL_SCALE <b>%, ORDERS_IMPLIED <a2>%, TOBIT/EM <c>%, IMPUTED_MODEL <d>%.
- Estimated lost demand <L> units (<v> value); <T> units transferred to substitutes; category net lost <L−T>.
- Masking validation: bias raw <r>% → reconstructed <q>% (WAPE <w>%); <n_fail> series fell back an engine.
- <n_lc> series LOW_CONFIDENCE (>30% reconstructed); <n_mm> HANDOFF_MISMATCH.

## Metadata (mandatory)
| Level | Grain | Time window | Signal | Engines / routing | Thresholds | Weighting | Lag | Profile source |
|---|---|---|---|---|---|---|---|---|
| SKU-Location | WEEK | <start–end> | POS / orders | Tobit · EM-Poisson · NB (GS-M-08 routing) | material 0.95 · cap ×2 · a<0.5→(c) · low-conf 0.30 | volume | n/a (history) | baseline <version> |

## KPI Table
| Metric | Value | Rule |
|---|---|---|
| % periods censored / material | | GS-M-11 |
| % reconstructed by method | | GS-K-13 |
| Lost demand units / value | | GS-M-11 |
| Transferred to substitutes | | GS-D-06 |
| Masking bias raw → recon (%) | | GS-G-11 |
| Masking WAPE recon (%) | | GS-K-01 |
| Unit fill rate (unconstrained denominator) | | GS-K-09 |

## Series Detail
| item | loc | engine | exact obs | censored periods | λ / β, σ | lost units | transferred | pct_reconstructed | masking bias | flags |
|---|---|---|---|---|---|---|---|---|---|---|

## Exceptions
| ID | Trigger (rule) | item | loc | metric | value | action | owner |
|---|---|---|---|---|---|---|---|
| | LOW_CONFIDENCE >30% (GS-M-11) | | | pct | | fix availability; widen intervals | Demand planner |
| | HANDOFF_MISMATCH (GS-D-13) | | | | | re-run cleansing | Data owner |
| | Masking validation failed (GS-G-11) | | | bias | | fallback engine | Data scientist |
| | RECON_LARGE (>3× sales) | | | ratio | | review episode | Demand planner |
| | Chronic stockout (lost > 10% of demand, 3 cycles) [RoT] | | | | | inventory policy review | Supply planner |

## Recommendations
1. …

## Data-Quality Log
| item | issue (no on-hand, no profile, FLAT_PROFILE, <8 exact obs, substitutes unavailable) | treatment |
|---|---|---|

## Lineage Extract (GS-D-13)
| item | loc | period | before | after | method | rule | params | timestamp |
|---|---|---|---|---|---|---|---|---|

## Reproducibility
Thresholds (0.95 / 2.0 / 0.5 / 0.30), routing rule, regressors, profile version, optimizer settings, masking seed, code §5.
```

## 7. Edge Cases
- **Full-period stockout with zero sales** ($\mathit{S}_t=0$, $\mathit{a}_t=0$): the conditional expectation reduces to the model mean $\mu_t$, which is correct. Never leave the period at 0 (that would be a false TRUE_ZERO).
- **Long stockout runs** (more than one cycle censored): the model has no recent exact level. Use (d) with the pre-stockout level and flag `LONG_OUTAGE`. Exclude the run from training if more than 30% of the series (cleansing then marks LOW_CONFIDENCE).
- **Allocation / rationing in B2B** (orders recorded): method (a) always wins. Never model what was recorded.
- **Promotion during a stockout:** include `promo_flag` in $\mathit{X}$ / $\mathit{w}_t$, otherwise lift is under-estimated. The promo skill then reads the reconstructed demand flagged `CENSORED_PROMO`.
- **Substitute also out of stock:** exclude it from $\mathit{U}_j$. If all substitutes are out, $\mathit{T}=0$ and the whole loss is category lost sales.
- **Phantom inventory** (on-hand > 0 but zero sales for an unusual run): not censored by the rule. Flag `PHANTOM_SUSPECT` when P(0 run | model) < 1% and route to `nextq-demand-anomaly-shift-detection`. Do not reconstruct without evidence.
- **Daily grain:** availability $\mathit{a}_t$ must be weighted by the intraday/day-of-week demand profile. A stockout on a Saturday removes more demand than one on a Tuesday.
- **Returns-netted sales** below zero in a censored period: floor $\mathit{S}_t$ at 0 for the censoring bound and log it.
