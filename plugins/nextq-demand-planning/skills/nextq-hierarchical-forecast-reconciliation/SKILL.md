---
name: nextq-hierarchical-forecast-reconciliation
description: >-
  [Next Quarter Supply Chain Suite] Makes forecasts coherent across product, location, channel and customer
  hierarchies and across time buckets, using bottom-up, top-down (historical or forecast proportions), middle-out, and
  MinT optimal reconciliation (OLS, WLS structural, WLS variance, MinT-shrink), with non-negativity, probabilistic
  (sample-path) reconciliation and temporal hierarchies. Load when the user asks to reconcile or roll up /
  disaggregate forecasts, make SKU forecasts add up to category, region or total, choose bottom-up vs top-down vs
  middle-out vs MinT, fix incoherent S&OP vs SKU numbers, or align weekly and monthly forecasts. Inputs: base
  forecasts and in-sample residuals for every node, a hierarchy / grouping table, and actuals for evaluation, at
  SKU-Location-Week/Month. Triggers: "reconcile", "forecasts don't add up", "hierarchical forecasting", "MinT",
  "top-down split", "disaggregate the forecast".
metadata:
  author: "Next Quarter"
  organization: "Next Quarter (nextq)"
  suite: "Next Quarter Supply Chain Intelligence"
  version: "1.0.0"
---

# Hierarchical Forecast Reconciliation
*Part of the **Next Quarter (nextq)** Supply Chain Skill Suite — Governed by SUPPLY_CHAIN_GOLD_STANDARD.md.*

## 1. Role & Objective
**Persona:** Hierarchical Forecasting Architect.
**Decision supported:** What is the single coherent set of forecasts that every level uses: SKU-DC for replenishment, brand-region for S&OP, total for finance? Which reconciliation method minimizes error across levels?

**Objective:** Take base forecasts from `nextq-statistical-baseline-forecasting`, `nextq-intermittent-demand-forecasting`, and `nextq-ml-causal-demand-forecasting` (any mix) and return **coherent** forecasts (means, and optionally sample paths/quantiles) for all nodes. The method is chosen by rolling-origin accuracy per level, with coherence verified to $10^{-6}$ relative.

## 2. Gold Standard Rules Enforced
| Rule | Inlined standard |
|---|---|
| **GS-D-07** Coherent mapping [STD] | Every coherent method is $\tilde{\boldsymbol y}_h = \boldsymbol S\boldsymbol G\hat{\boldsymbol y}_h$. Here $\boldsymbol S$ is the $n\times m$ summing matrix ($n$ total nodes, $m$ bottom series; $\boldsymbol y_t=\boldsymbol S\boldsymbol b_t$), and $\boldsymbol G$ ($m\times n$) maps base forecasts to the bottom level (FPP3 §11.3). **Bottom-up:** $\boldsymbol G=[\boldsymbol 0_{m\times(n-m)}\mid \boldsymbol I_m]$. **Top-down:** $\boldsymbol G=[\boldsymbol p\mid \boldsymbol 0]$ with proportions $p_j$. **Middle-out:** forecasts at middle level $k$ summed upward, top-down below. Top-down and middle-out apply **only to strictly nested hierarchies**, not grouped (crossed) structures. |
| **GS-D-07** Top-down proportions [STD] | (i) **Average historical proportions** $p_j=\frac1T\sum_{t=1}^{T}\frac{y_{j,t}}{y_t}$. (ii) **Proportions of historical averages** $p_j=\frac{\sum_t y_{j,t}/T}{\sum_t y_t/T}$. (iii) **Forecast proportions** (default top-down): $p_j=\prod_{\ell=0}^{K-1}\frac{\hat y^{(\ell)}_{j,h}}{\hat S^{(\ell+1)}_{j,h}}$, i.e. split each parent by its children's base-forecast shares, level by level. **All top-down methods are biased**: they violate $\boldsymbol S\boldsymbol G\boldsymbol S=\boldsymbol S$ (Hyndman et al. 2011). |
| **GS-D-08** MinT [STD] | Wickramasuriya, Athanasopoulos & Hyndman (2019): $\tilde{\boldsymbol y}_h=\boldsymbol S(\boldsymbol S'\boldsymbol W_h^{-1}\boldsymbol S)^{-1}\boldsymbol S'\boldsymbol W_h^{-1}\hat{\boldsymbol y}_h$. This minimizes $\operatorname{tr}(\boldsymbol V_h)$ with $\boldsymbol V_h=\boldsymbol S\boldsymbol G\boldsymbol W_h\boldsymbol G'\boldsymbol S'$, subject to $\boldsymbol S\boldsymbol G\boldsymbol S=\boldsymbol S$ (unbiasedness). Choose $\boldsymbol W_h$ in this priority: **(1) `mint_shrink`** $\hat{\boldsymbol W}=\lambda\hat{\boldsymbol D}+(1-\lambda)\hat{\boldsymbol W}_1$, with $\hat{\boldsymbol D}=\operatorname{diag}(\hat{\boldsymbol W}_1)$ and $\hat\lambda=\frac{\sum_{i\neq j}\widehat{\operatorname{Var}}(\hat r_{ij})}{\sum_{i\neq j}\hat r_{ij}^2}$ on the residual correlations $\hat r_{ij}$ (default when $m$ is large relative to $T$). **(2) `wls_var`** $\boldsymbol W=\operatorname{diag}(\hat{\boldsymbol W}_1)$. **(3) `wls_struct`** $\boldsymbol W=\boldsymbol\Lambda=\operatorname{diag}(\boldsymbol S\boldsymbol 1)$ (use when residuals are unavailable, e.g. judgmental bases). **(4) `ols`** $\boldsymbol W=\boldsymbol I$. $\hat{\boldsymbol W}_1=\frac1T\sum_t\boldsymbol e_t\boldsymbol e_t'$ uses **one-step in-sample residuals of every node**, stacked in the same order as $\boldsymbol y$. |
| **GS-D-08** Non-negativity | Reconciled forecasts can be negative. Fix: set negative **bottom** values to 0 and re-aggregate bottom-up from the clipped bottom level (coherent by construction). Alternatively, use a constrained solver. **Log** the count and volume of clipped values. |
| **GS-M-05** Means add, medians don't | Reconcile **means**. Back-transformed (log/Box-Cox) bases must be bias-adjusted to the mean before reconciliation (FPP3 §5.6). |
| **GS-M-10** Probabilistic reconciliation | If the base forecasts are Gaussian $N(\hat\mu,\hat\Sigma)$, then the reconciled forecasts are $N(\boldsymbol S\boldsymbol G\hat\mu,\ \boldsymbol S\boldsymbol G\hat\Sigma\boldsymbol G'\boldsymbol S')$. Otherwise reconcile each of $B$ simulated **sample paths** $(\boldsymbol S\boldsymbol G\hat{\boldsymbol y}^{[b]})$ and take quantiles (Panagiotelis et al. 2023; FPP3 §11.5). Never sum quantiles. |
| **GS-D-09** Temporal hierarchies | Weekly ↔ 4-weekly ↔ quarterly (or day ↔ week) form a temporal hierarchy (Athanasopoulos et al. 2017). The same $\boldsymbol S\boldsymbol G$ machinery applies with $\boldsymbol S$ built from the time-aggregation matrix. Calendar months that are not whole weeks must first be converted with day apportionment (`nextq-time-bucket-calendar-alignment`). |
| **GS-M-16 / GS-K-04 / GS-K-01** | Method choice by rolling-origin backtest, **scored per level** with scale-free MASE (seasonal-naïve training scale per node) and WAPE per level. Never compare RMSE across levels, because aggregates dominate. Report "All series" as the mean of per-node MASE. |
| **GS-G-11** Verification | $\lvert \text{parent}-\sum\text{children}\rvert\le10^{-6}\cdot\max(\lvert\text{parent}\rvert,1)$ at every node and period. For any non-top-down $\boldsymbol G$: $\lVert\boldsymbol S\boldsymbol G\boldsymbol S-\boldsymbol S\rVert_\infty<10^{-8}$. |
| **GS-G-03** Exceptions | Unreconciled delta > 0.5% at any node in the **incoming** base set (flags upstream disagreement) [RoT]. Reconciliation adjustment $\lvert\tilde y-\hat y\rvert/\hat y$ > 25% at a node [RoT]. Non-negativity clipping > 1% of volume [RoT]. |
| **GS-G-10** | Standard output contract. |

**Default method-selection rule (deterministic):**
1. Candidates: {BU, TD-forecast-proportions (hierarchies only), MO at the user-declared level (hierarchies only), OLS, WLS-struct, WLS-var, MinT-shrink}.
2. Pick the lowest **mean per-node MASE across all levels** at the decision lag.
3. Tie (< 1% relative) → prefer MinT-shrink, then WLS-var, then WLS-struct, then OLS, then BU (unbiased methods first).
4. **Override:** if the user mandates a level (e.g., finance top-down), apply it and report the accuracy cost vs. the winner.

## 3. Required Inputs & Data Schema
**`hierarchy`**: one row per bottom series.

| Column | Type | Req. | Notes |
|---|---|---|---|
| `bottom_id` | string | Y | e.g. `item_id|location_id` |
| `level_1` … `level_K` | string | Y | Nested parents from top (e.g., `TOTAL`, `region`, `brand`) |
| `group_*` | string | N | Crossed attributes (grouped structure: channel, customer). A grouped structure disables TD/MO |

**`base_forecasts`**: one row per node × target period × origin.

| Column | Type | Unit | Req. |
|---|---|---|---|
| `node_id` | string (bottom_id or aggregate key like `region=EU|brand=X`) | n/a | Y |
| `level` | string | n/a | Y |
| `origin` | date | n/a | Y |
| `target_period` | date | fiscal bucket | Y |
| `lag` | int ≥ 1 | periods | Y |
| `fcst_mean` | decimal ≥ 0 | base UoM (or value currency, never mixed) | Y |
| `model`, `forecast_version` | string | n/a | Y |
| `path_id`, `fcst_path` | int, decimal | n/a | N (sample paths for probabilistic reconciliation) |

**`residuals`**: `node_id, period, resid_1step` (in-sample one-step $e_t$, same window for every node). Required for WLS-var and MinT. If absent, only BU/TD/MO/OLS/WLS-struct are allowed, and the run is flagged `NO_RESIDUALS`.

**`actuals`**: `bottom_id, period, actual_unconstrained` (aggregates are computed via $\boldsymbol S$, never loaded separately).

**Granularity:** one time grain per run. All bases must be in the **same unit** (units or value) and on the **same calendar**.

**Minimum:**
- ≥ 3 origins for method selection (GS-M-16).
- For MinT-shrink, residual window $T$ ≥ 2 seasonal cycles recommended.
- With $T < m$ bottom series, use shrink (sample covariance is singular).

## 4. Execution Workflow
1. **Build $\boldsymbol S$.** Rows run from top to bottom (total, level 1, …, bottom identity $\boldsymbol I_m$). For grouped structures, add one row per crossed aggregate. Store `node_index`. Validate: every bottom maps to exactly one parent per nested level, and node IDs are unique.
2. **Align bases.** For each origin and lag, stack $\hat{\boldsymbol y}_h$ in `node_index` order.
   - Missing node forecast: BU-fill the aggregate from its children, or flag `MISSING_BASE`. MinT requires every node.
   - Check **incoming incoherence** at each parent, $\delta=\lvert\hat y_{\text{parent}}-\sum\hat y_{\text{children}}\rvert/\hat y_{\text{parent}}$. Flag if > 0.5%.
3. **Estimate $\boldsymbol W$** from residuals (same $T$ for all nodes; drop periods where any node is missing).
   - Compute $\hat{\boldsymbol W}_1$ and shrinkage $\hat\lambda$ (clip to [0,1]).
   - Check positive definiteness. If it fails, increase λ to 1 (i.e., WLS-var) and log it.
4. **Compute $\boldsymbol G$** for each candidate.
   - BU and TD use closed forms. For TD forecast proportions, split recursively by the children's share of the base forecasts. If all children are 0, fall back to average historical proportions; if those are also 0, use an equal split. Log the fallback.
   - Linear methods: $\boldsymbol G=(\boldsymbol S'\boldsymbol W^{-1}\boldsymbol S)^{-1}\boldsymbol S'\boldsymbol W^{-1}$, computed with `solve`, never with explicit inverses of large matrices.
5. **Backtest.** For each origin and lag: $\tilde{\boldsymbol y}=\boldsymbol S\boldsymbol G\hat{\boldsymbol y}$. Apply the non-negativity fix, then score against $\boldsymbol S\boldsymbol b^{\text{actual}}$ per node: MASE (per-node seasonal-naïve training scale), WAPE per level, Bias% per level.
6. **Select the method** by the rule in Section 2. Report the per-level table for **base (incoherent)**, BU, and the winner, at minimum.
7. **Production reconcile** on the latest origin, all horizons. For probabilistic outputs, reconcile each sample path, then take quantiles. Apply the non-negativity fix per path.
8. **Verify** (GS-G-11): the coherence assertion at every node and period, $\boldsymbol S\boldsymbol G\boldsymbol S=\boldsymbol S$ for unbiased methods, and no NaN.
9. **Adjustment audit:** per node, $\Delta=\tilde y-\hat y$ and $\Delta\%$. Flag $\lvert\Delta\%\rvert$ > 25%. Report the top-N value-at-risk nodes.
10. **Output** (Section 6). Persist `forecast_version = REC_<method>_<run_id>`.

## 5. Verification Snippets
```python
import numpy as np, pandas as pd

def build_S(hier: pd.DataFrame, levels: list[str]):
    """hier: one row per bottom_id with nested level columns (top -> bottom parent). Returns S, node list."""
    bottoms = hier.bottom_id.tolist(); m = len(bottoms); rows, nodes = [], []
    rows.append(np.ones(m)); nodes.append(("TOTAL", "TOTAL"))
    for lv in levels:
        for key, idx in hier.groupby(lv).indices.items():
            r = np.zeros(m); r[idx] = 1; rows.append(r); nodes.append((lv, key))
    S = np.vstack(rows + [np.eye(m)]); nodes += [("bottom", b) for b in bottoms]
    return S, nodes

def shrink_cov(E):
    """Schafer-Strimmer style shrinkage of residual covariance toward its diagonal (MinT-shrink).
    E: T x n residual matrix (rows = periods, same window for all nodes)."""
    T, n = E.shape
    Ec = E - E.mean(0)
    W1 = Ec.T @ Ec / T
    sd = np.sqrt(np.diag(W1)); sd[sd == 0] = 1e-12
    Xs = Ec / sd
    R = Xs.T @ Xs / T
    w = np.einsum("ti,tj->tij", Xs, Xs)                       # T x n x n
    var_r = ((w - w.mean(0)) ** 2).sum(0) * T / (T - 1) ** 3   # Schafer-Strimmer Var-hat(r_ij)
    off = ~np.eye(n, dtype=bool)
    lam = np.clip(var_r[off].sum() / (R[off] ** 2).sum(), 0, 1) if (R[off] ** 2).sum() > 0 else 1.0
    D = np.diag(np.diag(W1))
    return lam * D + (1 - lam) * W1, lam

def G_matrix(S, method, W=None):
    n, m = S.shape
    if method == "bu":
        return np.hstack([np.zeros((m, n - m)), np.eye(m)])
    if method == "ols":
        W = np.eye(n)
    elif method == "wls_struct":
        W = np.diag(S.sum(1))
    elif method in ("wls_var", "mint_shrink", "mint_cov"):
        assert W is not None
        if method == "wls_var":
            W = np.diag(np.diag(W))
    Winv_S = np.linalg.solve(W, S)                            # W^{-1} S
    return np.linalg.solve(S.T @ Winv_S, Winv_S.T)            # (S'W^-1 S)^-1 S'W^-1

def reconcile(S, G, yhat, nonneg=True):
    b = G @ yhat
    clipped = 0.0
    if nonneg and (b < 0).any():
        clipped = -b[b < 0].sum(); b = np.clip(b, 0, None)    # clip bottom, re-aggregate (coherent)
    return S @ b, clipped

def assert_coherent(S, ytilde, m, tol=1e-6):
    b = ytilde[-m:]; agg = S @ b
    assert np.all(np.abs(agg - ytilde) <= tol * np.maximum(np.abs(ytilde), 1)), "incoherent"

def assert_unbiased(S, G, tol=1e-8):
    assert np.abs(S @ G @ S - S).max() < tol, "SGS != S (biased mapping)"
```
```python
# Self-test (executed): 2-region x 2-item hierarchy, noisy incoherent bases
rng = np.random.default_rng(7)
hier = pd.DataFrame({"bottom_id": ["A1", "A2", "B1", "B2"], "region": ["A", "A", "B", "B"]})
S, nodes = build_S(hier, ["region"])                         # n=7, m=4
E = rng.normal(0, 1, (60, 7)) * np.array([4, 3, 3, 1, 1, 1, 1])
W, lam = shrink_cov(E)
yhat = np.array([105., 52., 49., 26., 25., 24., 26.])        # incoherent: 52+49 != 105
for meth in ("bu", "ols", "wls_struct", "wls_var", "mint_shrink"):
    G = G_matrix(S, meth, W); yt, _ = reconcile(S, G, yhat)
    assert_coherent(S, yt, 4); assert_unbiased(S, G)
```
```sql
-- Coherence check on stored reconciled output (must return zero rows)
SELECT p.node_id, p.target_period, p.fcst_rec, SUM(c.fcst_rec) AS child_sum
FROM reconciled p
JOIN hierarchy_edges e ON e.parent_id = p.node_id
JOIN reconciled c ON c.node_id = e.child_id AND c.target_period = p.target_period AND c.origin = p.origin
GROUP BY p.node_id, p.target_period, p.fcst_rec
HAVING ABS(p.fcst_rec - SUM(c.fcst_rec)) > 1e-6 * GREATEST(ABS(p.fcst_rec), 1);
```

## 6. Standardized Output Format
```markdown
# Hierarchical Reconciliation — <hierarchy name> — run REC_<method>_<run_id>

## Executive Summary
- Structure: <n> nodes / <m> bottom series / <K> levels (<hierarchical | grouped>); bases from <models>.
- Selected method: <MinT-shrink (λ=0.42)>; mean per-node MASE <x> vs base <y> vs bottom-up <z> at lag ℓ.
- Incoming incoherence: <p>% of parents off by >0.5% (largest: <node>, <Δ%>).
- Non-negativity clipping: <u> units (<v>% of volume); largest reconciliation adjustments: <nodes>.
- Coherence verified at all <n×H> node-periods (tolerance 1e-6): PASS.

## Metadata
| Level(s) | Grain | Lag | Time window | Origins | Horizon | Weighting | Unit | Residual window T |
|---|---|---|---|---|---|---|---|---|
| Total → Region → Brand → SKU-DC | WEEK | <ℓ> | <start–end> | <n> | <H> | per-level WAPE volume-weighted; MASE mean per node | base UoM | <T> |

## Accuracy by Level and Method (lag ℓ)
| Level | Base (incoherent) MASE | BU | TD-fp | MO | OLS | WLS-struct | WLS-var | MinT-shrink | WAPE % (selected) | Bias % (selected, +=over) |
|---|---|---|---|---|---|---|---|---|---|---|
| Total | | | | | | | | | | |
| Region | | | | | | | | | | |
| Brand | | | | | | | | | | |
| Bottom (SKU-DC) | | | | | | | | | | |
| **All series (mean)** | | | | | | | | | | |

## Reconciliation Adjustments (top N by |Δ| value)
| node | level | period | base | reconciled | Δ units | Δ % | flag |
|---|---|---|---|---|---|---|---|

## Exceptions
| ID | Trigger (rule, threshold) | node | period | value | action | owner |
|---|---|---|---|---|---|---|
| | Incoming incoherence >0.5% (GS-G-03) | | | | align upstream versions/units | |
| | |Δ%| >25% reconciliation adjustment | | | | review base model at node | |
| | Clipping >1% volume (GS-D-08) | | | | check negative bases / intermittent bottoms | |
| | NO_RESIDUALS (WLS-var/MinT unavailable) | | | | provide one-step residuals | |
| | W not PD → λ forced to 1 | | | | lengthen residual window | |
| | Mandated method ≠ winner (cost <x> MASE) | | | | governance decision | |

## Recommendations
1. …

## Data-Quality Log
| item | issue | treatment |
|---|---|---|
| missing node bases | BU-filled / excluded | |
| proportion fallbacks (TD) | hist-avg / equal split | |
| unit/calendar conversions | | |

## Reproducibility
S matrix hash; node order; method; λ; residual window; origins; tolerance; code §5; library versions.
```

## 7. Edge Cases
- **Grouped (crossed) structures:** TD and MO are disabled (no unique parent path). Use BU or MinT family only.
- **All-zero intermittent bottoms under a positive parent:** MinT can push negatives. Apply the clip-and-re-aggregate fix. If clipping exceeds 1% of volume, prefer WLS-struct or BU for that sub-tree and log it.
- **Units vs. value:** never reconcile a mix. Reconcile units, then price at the bottom level for value (`nextq-demand-financial-reconciliation`).
- **Judgmental overrides at an aggregate** (from `nextq-consensus-demand-review`): treat the override as the base for that node, use WLS-struct (no residuals), or **lock** the node by setting a tiny $W_{ii}$ (e.g., $10^{-6}\times$ the median diagonal) so reconciliation preserves it. Log `LOCKED_NODE`.
- **Large $m$** (> 10⁴ bottoms): use sparse $\boldsymbol S$ and an iterative or block solver, or reconcile per top-level sub-tree when sub-trees are independent. Document the approximation.
- **Temporal + cross-sectional:** reconcile temporally first, then cross-sectionally (or cross-temporally, if tooling allows), and verify coherence in both dimensions.
- **New nodes** with no residual history: give them WLS-struct weights within MinT by imputing $W_{ii}$ from the median residual variance of their level, and flag `IMPUTED_W`.
