---
name: nextq-safety-stock-optimization
description: >-
  [Next Quarter Supply Chain Suite] Sets safety stock and reorder points for cycle-service-level or fill-rate targets
  from the standard deviation of forecast error over the protection interval (measured directly at lead time, or
  scaled only when errors are uncorrelated) plus lead-time variability. Uses the normal loss function where valid, and
  Poisson, negative-binomial, gamma or simulated lead-time demand for intermittent, small-count or high-variability
  items. Segments targets by ABC/XYZ, never equates cycle service level with fill rate, and spot-checks results by
  simulation. Load when the user asks to calculate, review or optimize safety stock, reorder points or buffer
  inventory, set service-level targets, size stock for a fill-rate goal, or explain a safety stock change. Inputs:
  forecast errors by lag, forecasts, lead times and variability, review periods, order quantities, segments. Triggers:
  "safety stock", "reorder point", "service level", "fill rate target".
metadata:
  author: "Next Quarter"
  organization: "Next Quarter (nextq)"
  suite: "Next Quarter Supply Chain Intelligence"
  version: "1.0.0"
---

# Safety Stock Optimization
*Part of the **Next Quarter (nextq)** Supply Chain Skill Suite — Governed by SUPPLY_CHAIN_GOLD_STANDARD.md.*

## 1. Role & Objective
**Persona:** Inventory Planner / Inventory Optimization Analyst.
**Decision supported:** How much safety stock, and what reorder point, does each item-location need to meet its service target at minimum inventory? What does a target change cost?

**Objective:** Deterministic, auditable safety stock (SS) and reorder points (ROP) per item-location, which:
- are built on **forecast-error** uncertainty over the protection interval,
- use the correct distribution per demand type,
- report both cycle service level (CSL) and fill rate (never equated),
- are spot-checked by simulation, and
- come with the inventory investment delta.

`nextq-replenishment-policy-design` consumes these results for (s, Q), (R, S) and (s, S) parameters.

## 2. Gold Standard Rules Enforced
| Rule | Inlined standard |
|---|---|
| **GS-M-12** CSL target [STD] | CSL = P(no stockout per replenishment cycle) $=\Phi(k)$, so $k=\Phi^{-1}(\text{CSL})$. $\sigma_{DL}=\sqrt{\bar L\,\sigma_d^2+\bar d^{\,2}\sigma_L^2}$, $SS=k\,\sigma_{DL}$, $ROP=\bar d\,\bar L+SS$. Periodic review (R, S): replace $L$ with $R+L$. **$\sigma_d$ must be the standard deviation of forecast error at the relevant lag and bucket** (e.g. RMSE of lag-L forecasts), **not** the standard deviation of raw demand (Silver, Pyke & Thomas). $\sigma_L=\sigma_1\sqrt L$ only if errors are uncorrelated; otherwise measure errors directly at horizon L. **No normal-k SS** for intermittent or count items (use GS-M-08), or when $\sigma_{DL}/\mu_{DL}>0.5$ [RoT]. |
| **GS-M-13** Fill rate (β) [STD] | Unit normal loss $G(k)=\phi(k)-k[1-\Phi(k)]$. **(s, Q):** $\beta=1-\sigma_{DL}G(k)/Q$, so solve $G(k)=(1-\beta)Q/\sigma_{DL}$. **(R, S):** $\beta=1-\sigma_{R+L}G(k)/(\bar dR)$. If $k<0$ (large Q), SS may be negative: floor it at the policy minimum and report it. **Never equate CSL and fill rate**; the same $k$ gives different values of each. |
| **GS-M-08** Count / intermittent | Small counts (max ≤ 10 per period or zero share ≥ 30% [RoT], the same rule as `nextq-intermittent-demand-forecasting`) or ADI ≥ 1.32 use a lead-time demand **distribution**: Poisson if VMR ≤ 1.2; otherwise NB with per-period size $r=\mu^2/(\sigma^2-\mu)$ scaled by the protection interval. Fit on non-censored, non-structural periods. Structural zeros are excluded. |
| **GS-M-10** Probabilistic | SS for non-normal cases is a **quantile of the lead-time demand distribution** (sum of sample paths, not summed quantiles). |
| **GS-M-11** Censoring | Forecast errors are measured against **unconstrained** actuals (`nextq-unconstrained-demand-reconstruction`). Errors against stockout-truncated sales understate $\sigma$ and feed the under-stocking loop. More than 30% reconstructed → `LOW_CONFIDENCE`. |
| **GS-K-08 / GS-K-09** | Planned CSL $=\Phi(k)$ (cycle qualifier always stated). Unit fill rate = units filled from stock on first pass / units **demanded** (unconstrained). Both are reported for every policy. |
| **GS-G-09** Segmentation | Targets by ABC × XYZ [RoT defaults]: A = 98%, B = 95%, C = 90% CSL (or the equivalent fill-rate targets). Critical items may override with a reason. Review frequency follows the segment. |
| **GS-G-11** Verification | Every safety stock is spot-checked against simulation. A deviation of more than 2 pp between simulated and target service [RoT] → use the simulation-based ROP and flag `CLOSED_FORM_MISMATCH`. |
| **GS-G-10 / GS-D-13** | Standard output; every parameter and override logged. |

**Error-σ rules (deterministic):**
1. **Preferred:** RMSE of cumulative forecast errors over the protection interval measured directly, from origins at lag = lead time, with ≥ 13 origins [RoT].
2. **Otherwise:** per-period RMSE × $\sqrt{L}$, but only if the lag-1 autocorrelation of errors satisfies $\lvert\rho_1\rvert<2/\sqrt n$. If not, flag `ERROR_AUTOCORR` and require the direct method within one cycle.
3. RMSE (not std) is used, so bias is inside $\sigma$. Persistent bias is also routed to `nextq-demand-exception-management` (fix the bias; do not buffer it forever).
4. $\bar d$ = the **forecast** mean over the protection interval, not the historical average.

## 3. Required Inputs & Data Schema
| Table | Columns | Notes |
|---|---|---|
| `forecast_errors` | `item_id, location_id, origin, lag, horizon_len, A_unc, F, e = A − F` | at lag = LT; cumulative-horizon errors preferred |
| `forecast` | `item_id, location_id, period, F` | next R + L periods |
| `lead_times` | `item_id, location_id, supplier_id, L_mean (periods, fractional), L_sd, L_empirical (optional distribution)` | from receipts history |
| `policy` | `policy_type ∈ {sQ, RS, sS}, Q, R, MOQ, min_ss, max_days (shelf life)` | |
| `targets` | `segment (ABC×XYZ), target_type ∈ {CSL, FILL}, target` | GS-G-09 |
| `segmentation` | `abc, xyz, demand_class, ADI, CV2, VMR, zero_share` | from `nextq-demand-pattern-segmentation` |
| `history_clean` | `item_id, location_id, period, demand_clean, period_state` | for count fits |
| `costs` | `unit_cost, holding_rate` | investment |

**Units:** lead time in the forecast bucket. Day-based lead times are converted as $L_{\text{weeks}}=L_{\text{days}}/7$, keeping the fraction (`nextq-time-bucket-calendar-alignment`).

## 4. Execution Workflow
1. **Assemble the protection interval:** PI = L (continuous review) or R + L (periodic). $\mu_{DL}=\sum_{t\in PI}F_t$.
2. **Error σ** per the rules above, and the censoring share.
3. **$\sigma_{DL}$** $=\sqrt{\bar L\sigma_d^2+\bar d^2\sigma_L^2}$ (per-period σ_d). With the direct method, the measured cumulative-error σ replaces the $\bar L\sigma_d^2$ term; the lead-time variability term is added separately.
4. **Route:**
   - `COUNT_DIST` (intermittent or small counts): Poisson / NB over PI.
   - `GAMMA_OR_EMPIRICAL` ($\sigma_{DL}/\mu_{DL}>0.5$): moment-matched gamma, or the empirical / bootstrap error distribution.
   - Otherwise `NORMAL`.
5. **Target** from the segment.
6. **Compute SS / ROP:**

   | Route | CSL target | Fill-rate target |
   |---|---|---|
   | NORMAL | $k=\Phi^{-1}(\text{CSL})$, SS $=k\sigma_{DL}$ | solve $G(k)$ for (s, Q) or (R, S) |
   | COUNT_DIST | smallest $s$ with $P(X\le s)\ge$ CSL; SS $=s-\mu$ | smallest $s$ with $1-E[(X-s)^+]/Q\ge\beta$ |
   | GAMMA_OR_EMPIRICAL | quantile minus mean | expected shortage computed from the distribution |

7. **Constraints:**
   - SS ≥ `min_ss`, and SS ≥ 0 unless negative SS is allowed by policy (report it).
   - ROP + Q must not exceed `max_days` × daily forecast (shelf life). If it does → `SHELF_LIFE_BOUND`, and report the achievable service.
8. **Report both service measures:** for every result, the implied CSL and the implied fill rate (e.g. a fill-rate-based $k$ → CSL $=\Phi(k)$).
9. **Spot check:**
   - Monte Carlo lead-time demand (forecast + error σ, empirical L distribution, 200,000 draws, seed fixed).
   - If |simulated − target| > 2 pp, replace the ROP with the simulated quantile (`CLOSED_FORM_MISMATCH`). This is typical with skewed or bimodal lead times.
10. **Investment:** ΔSS × unit cost, plus holding cost per year = ΔSS × cost × holding rate. Also the change vs. current SS, and its drivers (σ, L, σ_L, target).
11. **Output** (Section 6).

## 5. Verification Snippets
```python
import numpy as np
from scipy import stats, optimize

def lt_error_sigma(errors_1step=None, errors_cum_L=None, L=1.0, n_min=13):
    """GS-M-12: sigma of FORECAST ERROR over the protection interval (not sigma of demand).
    Preferred: RMSE of cumulative L-period forecast errors measured directly (origin-based).
    Fallback: sigma_1 * sqrt(L) ONLY if 1-step errors are uncorrelated (|rho1| < 2/sqrt(n)); else ERROR_AUTOCORR.
    RMSE (not std) is used so bias is included; bias itself is routed to exception management."""
    if errors_cum_L is not None and len(errors_cum_L) >= n_min:
        e = np.asarray(errors_cum_L, float); return float(np.sqrt(np.mean(e ** 2))), "DIRECT_LT"
    e = np.asarray(errors_1step, float); n = len(e)
    if n < n_min: return np.nan, "INSUFFICIENT_ERRORS"
    rho1 = np.corrcoef(e[:-1], e[1:])[0, 1]
    s1 = float(np.sqrt(np.mean(e ** 2)))
    if abs(rho1) >= 2 / np.sqrt(n): return s1 * np.sqrt(L), "ERROR_AUTOCORR"     # flagged: measure directly
    return s1 * np.sqrt(L), "SQRT_L"

def sigma_dl(sigma_d_per_period, L_mean, d_mean, sigma_L):
    """sigma_DL = sqrt(L * sigma_d^2 + d^2 * sigma_L^2); sigma_d = per-period forecast-error sigma; d = FORECAST mean."""
    return float(np.sqrt(L_mean * sigma_d_per_period ** 2 + d_mean ** 2 * sigma_L ** 2))

def G(k):
    """Unit normal loss function G(k) = phi(k) - k (1 - Phi(k))."""
    return stats.norm.pdf(k) - k * stats.norm.sf(k)

def ss_csl(csl, s_dl):
    k = stats.norm.ppf(csl); return k * s_dl, k

def k_fill_rate(beta, s_dl, Q=None, d_mean=None, R=None):
    """(s,Q): G(k) = (1-beta) Q / sigma_DL. (R,S): G(k) = (1-beta) d R / sigma_{R+L}. Solve for k (may be < 0)."""
    target = (1 - beta) * (Q if Q is not None else d_mean * R) / s_dl
    return optimize.brentq(lambda k: G(k) - target, -6, 6)

def route(y_hist, adi, mu_dl, s_dl, max_small=10, zero_share=0.30):
    """Normal formulas are NOT allowed for intermittent (ADI >= 1.32), small counts (max <= 10 per period or zero
    share >= 30%, same rule as nextq-intermittent-demand-forecasting) or CV_DL = sigma_DL/mu_DL > 0.5."""
    y = np.asarray(y_hist, float)
    if adi >= 1.32 or y.max() <= max_small or (y == 0).mean() >= zero_share:
        return "COUNT_DIST"
    if mu_dl > 0 and s_dl / mu_dl > 0.5:
        return "GAMMA_OR_EMPIRICAL"
    return "NORMAL"

def count_ltd(y_hist, rate, PI):
    """Lead-time (protection-interval) demand distribution, same parameterisation as the intermittent skill:
    VMR <= 1.2 -> Poisson(rate*PI); else NB with size r = PI * mu^2/(var-mu) per-period dispersion."""
    y = np.asarray(y_hist, float); mu, var = y.mean(), y.var(ddof=0); m = rate * PI
    if mu <= 0 or var / mu <= 1.2: return stats.poisson(m), "POISSON"
    r = PI * mu ** 2 / (var - mu); return stats.nbinom(r, r / (r + m)), "NEG_BIN"

def ss_discrete_csl(dist, csl):
    """Smallest s with P(LTD <= s) >= CSL; SS = s - mean (can be fractional / negative -> report)."""
    s = int(dist.ppf(csl)); return s, s - dist.mean()

def fill_rate_discrete(dist, s, Q):
    """(s,Q) fill rate = 1 - E[(X - s)^+] / Q per cycle, X = lead-time demand (discrete, exact sum)."""
    x = np.arange(int(s) + 1, int(dist.ppf(1 - 1e-10)) + 2)
    esc = np.sum((x - s) * dist.pmf(x)); return 1 - esc / Q

def gamma_ltd(mu_dl, s_dl):
    """Moment-matched gamma for continuous high-CV lead-time demand [RoT]."""
    k = (mu_dl / s_dl) ** 2; return stats.gamma(k, scale=s_dl ** 2 / mu_dl)

def simulate_ltd(rop, Q, d_mean, sd_d, L_values, L_probs, n=200_000, seed=7):
    """GS-G-11 spot check of the closed form: Monte Carlo lead-time demand LTD = sum of L per-period demands
    (forecast + error, sd_d) with L drawn from its empirical distribution. Returns simulated
    CSL = P(LTD <= ROP) and (s,Q) fill rate = 1 - E[(LTD - ROP)^+]/Q."""
    rng = np.random.default_rng(seed)
    L = rng.choice(L_values, size=n, p=L_probs)
    ltd = rng.normal(d_mean * L, sd_d * np.sqrt(L))          # sum of L iid normal periods
    return float(np.mean(ltd <= rop)), float(1 - np.mean(np.maximum(ltd - rop, 0)) / Q)

def rop_by_simulation(csl, d_mean, sd_d, L_values, L_probs, n=200_000, seed=7):
    """Used when the closed form fails the spot check (|CSL_sim - target| > 2 pp [RoT]): ROP = CSL-quantile of
    simulated LTD (same draws as simulate_ltd)."""
    rng = np.random.default_rng(seed)
    L = rng.choice(L_values, size=n, p=L_probs)
    ltd = rng.normal(d_mean * L, sd_d * np.sqrt(L))
    return float(np.quantile(ltd, csl))
```
**Test vector** (executed):
- **Error σ:**
  - White-noise errors (σ = 20, n = 52) over L = 4 → **43.17**, `SQRT_L`.
  - AR(1) errors with ρ = 0.7 → `ERROR_AUTOCORR` (49.26, must move to the direct method).
  - 26 direct cumulative errors → **53.56**, `DIRECT_LT`.
- **Normal (s, Q)**, with d̄ = 100/period, σ_d = 20, L = 4, σ_L = 1: σ_DL = **107.70**.
  - CSL 95%: k = 1.6449, SS = **177.16**, ROP = 577.16.
  - Fill rate 99% with Q = 400: k = **1.3942**, SS = 150.16, G(k) = 0.037139. The implied CSL is only **91.8%**, which shows CSL ≠ fill rate.
  - Q = 2,000 at 95% fill gives k = −0.81, so negative SS is floored and reported.
- **Routing:**
  - Intermittent sample → `COUNT_DIST`.
  - CV_DL = 0.77 → `GAMMA_OR_EMPIRICAL`.
  - CV_DL = 0.23 → `NORMAL`.
- **Count distributions:**
  - VMR > 1.2 history, rate 1.1 over PI = 4 → NB, mean 4.4, s = 9, SS = 4.6, fill rate (Q = 10) = 0.9882.
  - Poisson case, mean 4 → s = 8, SS = 4.0.
- **Gamma:** μ = 100, σ = 80 → SS **156.97**, vs. a normal 131.59, which would under-protect a skewed distribution.
- **Simulation spot check:**
  - With σ_L = 0, closed-form CSL 95% simulates at **0.9497**, and 99% fill rate at **0.9900**, both within 2 pp. That fill-rate policy's simulated CSL is only 0.817.
  - With a bimodal lead time (3 or 5 periods, σ_L = 1), the closed form simulates at **0.9787** (+2.9 pp) → `CLOSED_FORM_MISMATCH`. The simulated ROP is 557.34 (SS 157.34 instead of 177.16).

```sql
-- Direct lead-time error sigma per item-location: cumulative error of forecasts made at origin o for (o+1 .. o+L)
WITH cum AS (
  SELECT item_id, location_id, origin,
         SUM(a_unc) - SUM(f) AS e_cum_L, COUNT(*) AS periods
  FROM forecast_errors
  WHERE lag BETWEEN 1 AND :L                        -- forecasts from the same origin covering the protection interval
  GROUP BY 1,2,3
  HAVING COUNT(*) = :L)
SELECT item_id, location_id, COUNT(*) AS n_origins, SQRT(AVG(e_cum_L * e_cum_L)) AS rmse_lt,
       COUNT(*) >= 13 AS direct_ok
FROM cum GROUP BY 1,2;
```

## 6. Standardized Output Format
```markdown
# Safety Stock Optimization — <scope> — run SS_<run_id>

## Executive Summary
- <N> item-locations: SS <units> (<value> at cost), Δ vs current <+/-x> (<holding cost/yr>); drivers: σ <a>%, L <b>%, σ_L <c>%, target <d>%.
- Routes: NORMAL <n1>, COUNT_DIST <n2>, GAMMA/EMPIRICAL <n3>; <m> CLOSED_FORM_MISMATCH replaced by simulation.
- Service: planned CSL <x>% / fill rate <y>% (both reported; not equated); <s> SHELF_LIFE_BOUND items cannot reach target.
- Confidence: <e>% direct-LT error σ, <ac> ERROR_AUTOCORR, <lc> LOW_CONFIDENCE (censored >30%).

## Metadata (mandatory)
| Level | Grain | Error lag / method | Time window (errors) | Protection interval | Distribution routing | Targets | Weighting | Simulation |
|---|---|---|---|---|---|---|---|---|
| SKU-Location | WEEK | lag L / DIRECT_LT or SQRT_L | <start–end> (<n> origins) | L or R+L (fractional) | GS-M-08 rule | ABC 98/95/90 CSL | units; value at cost | 200k draws, seed 7, ±2 pp |

## KPI Table
| Segment | N | SS units | SS value | Δ value | Planned CSL | Planned fill rate | Simulated CSL | Simulated fill rate | Days of cover (SS) |
|---|---|---|---|---|---|---|---|---|---|

## Item Detail
| item | loc | route | μ_DL | σ_d (method) | L / σ_L | σ_DL | target (type) | k / s | SS | ROP | implied CSL | implied FR | sim check | flags |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|

## Exceptions
| ID | Trigger (rule) | item | metric | value | action | owner |
|---|---|---|---|---|---|---|
| | ERROR_AUTOCORR (GS-M-12) | | ρ1 | | measure direct LT errors | Inventory analyst |
| | CLOSED_FORM_MISMATCH (GS-G-11) | | Δpp | | use simulated ROP | Inventory analyst |
| | Negative SS floored (GS-M-13) | | k | | review Q / policy | Inventory planner |
| | SHELF_LIFE_BOUND | | achievable service | | accept lower target / smaller Q | Category / supply |
| | LOW_CONFIDENCE (GS-M-11) | | % reconstructed | | fix availability; review | Demand planner |
| | Bias inflating σ (GS-K-05) | | Bias % | | exception queue | Demand planner |

## Recommendations
1. …

## Data-Quality Log
| item | issue (few origins, missing σ_L, censored errors, fractional LT, MOQ conflict) | treatment |
|---|---|---|

## Reproducibility
Error snapshot, targets version, routing thresholds, seed, code §5.
```

## 7. Edge Cases
- **No lead-time history:** σ_L = 0 is optimistic. Use the supplier's contractual ±range as a uniform distribution, and flag `LT_ASSUMED`.
- **Skewed or bimodal lead times** (air vs. sea, port delays): the closed form may misstate service. Simulate with the empirical L distribution; the spot check enforces this.
- **Seasonal demand:** time-phase SS by period, using σ and μ over each period's forward protection interval. Never apply one annual SS to peak and off-peak.
- **New items:** use analog error σ scaled by the analog/new volume ratio, inflated by 1.25 [RoT], flagged until 13 origins exist (`nextq-new-product-lifecycle-forecasting`).
- **MOQ much larger than demand** (k < 0 under fill rate): SS is floored. Report that cycle stock already delivers the service, and consider a CSL-based floor for A items.
- **Phase-out items:** target SS = 0 after the last-buy date, with run-out handled by the lifecycle skill.
- **Multi-echelon networks:** this skill sets single-echelon SS per location. Upstream DC buffers serving several stores need pooled-σ logic and are flagged `MULTI_ECHELON_REVIEW`. That is out of scope; it is not silently summed.
- **Censored errors:** if the share of reconstructed actuals exceeds 30%, keep SS from the reconstructed error σ, but cap SS reductions at 20% per cycle [RoT] until confidence returns.
