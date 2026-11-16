---
name: nextq-new-product-lifecycle-forecasting
description: >-
  [Next Quarter Supply Chain Suite] Forecasts items across their lifecycle where statistical models fail: new-product
  launches via attribute-similar analogs (shape x scale launch curves with distribution ramps, P10/P90 ranges,
  early-read re-scaling), supersessions (history chaining with conversion factors), and phase-outs (TSB decay, run-out
  and end-of-life excess). Load when the user asks to forecast a new product, launch or line extension, find analog or
  like items, build a launch curve, chain history from an old to a new SKU, plan a phase-out or run-out, or decide
  when a new item can switch to statistical forecasting. Inputs: item attributes, analog launch histories, planned
  distribution and price, supersession links, inventory and EOL dates. Triggers: "new product forecast", "NPI",
  "launch forecast", "like item", "analog", "supersession", "phase-out", "end of life", "run-out".
metadata:
  author: "Next Quarter"
  organization: "Next Quarter (nextq)"
  suite: "Next Quarter Supply Chain Intelligence"
  version: "1.0.0"
---

# New Product & Lifecycle Forecasting
*Part of the **Next Quarter (nextq)** Supply Chain Skill Suite — Governed by SUPPLY_CHAIN_GOLD_STANDARD.md.*

## 1. Role & Objective
**Persona:** Lifecycle / NPI Demand Planner.
**Decision supported:** What volume should we plan for a launch (point and range), and how quickly should it be revised as actuals arrive? How do we carry history across a SKU replacement? How do we run an item down without stranding inventory?

**Objective:**
- Produce governed lifecycle forecasts for items that cannot yet (or can no longer) be forecast by `nextq-statistical-baseline-forecasting` / `nextq-intermittent-demand-forecasting`, each with a documented analog set, expected range, and handoff date.
- Chain supersession history so downstream models see one continuous demand stream.
- Produce phase-out forecasts that decay to zero (TSB) with a run-out excess estimate.

## 2. Gold Standard Rules Enforced
| Rule | Inlined standard |
|---|---|
| **GS-D-12** Lifecycle & master data | New items use analogs or likeness-based curves: **shape × scale**, with a documented analog and expected range. Supersessions chain history from old to new SKU with a **conversion factor**. Phase-outs switch to **TSB** or a run-out plan. Minimum-history gates for statistical models are set here (below) and documented. |
| **GS-D-02** States | Pre-launch and post-discontinuation periods are **STRUCTURAL_ZERO**: masked, never trained on. Launch weeks with in-stock < 95% are censored and must be reconstructed before an item is used as an analog (`nextq-unconstrained-demand-reconstruction`). |
| **GS-M-06** TSB | Demand-occurrence probability $\pi$ updated **every** period: $y_t>0:\ \pi_t=\pi_{t-1}+\beta(1-\pi_{t-1}),\ z_t=z_{t-1}+\alpha(y_t-z_{t-1})$; $y_t=0:\ \pi_t=\pi_{t-1}(1-\beta),\ z_t=z_{t-1}$. Forecast $\hat y=\pi_t z_t$. $\alpha,\beta\in[0.05,0.3]$ [RoT]. TSB is **required** for obsolescence / EOL decisions (it decays during zero runs; Croston/SBA do not). |
| **GS-M-10** Probabilistic | Point forecasts are insufficient. Launch ranges come from the **dispersion of analog scales** (weighted empirical P10/P90, ≥ 3 analogs) and are reported with the point. Intervals over a horizon come from summed paths, never summed quantiles. |
| **GS-G-01** Overrides | Launch overrides need a reason code and evidence. **UP > 20%** vs. the analog forecast requires secondary approval (optimism bias is strongest for launches; Fildes et al. 2009 found upward adjustments less likely to improve accuracy). Materiality: reject $\lvert\Delta\rvert<\max(10\%\cdot F, 1\cdot\text{MAD})$; for launches without MAD, use 10% only. |
| **GS-G-09** Segmentation | Items in LAUNCH/GROWTH get segment `NEW` (weekly review, override-eligible, CSL target per business rule) until handoff; they are excluded from ABC/XYZ until 26 weeks of history [RoT]. |
| **GS-K-05 / GS-K-01** | Launch accuracy is scored **cumulatively** over the first $H$ periods: cumulative Bias% $=(\sum F-\sum A)/\sum A$ (+ = over) and WAPE $=\sum\lvert A-F\rvert/\sum A$, at fixed lag (forecast frozen at launch − lead time). |
| **GS-G-10 / GS-G-11 / GS-D-13** | Standard output; reproducible from analog IDs and parameters; every chained or rescaled value logged (before, after, rule, timestamp). |

**Lifecycle parameters [RoT, configurable]:**
- **Analogs:** $k=3$–5 analogs with similarity ≥ 0.60. Each must have ≥ $H$ weeks of clean post-launch history (default $H=26$).
- **Early-read weight:** $w=n/(n+n_0)$ with $n_0=4$.
- **Handoff to statistical:** ≥ 26 non-structural weeks for non-seasonal items, ≥ 52 for seasonal. Blend linearly from week 13 to 26: $F=(1-\omega)F_{\text{analog}}+\omega F_{\text{stat}}$, $\omega=(n-13)/13$. Hand off fully only if stat MASE ≤ analog MASE on the last 8 weeks.
- **Phase-out:** TSB from the phase-out announcement date, with α = β = 0.1 unless optimized.

## 3. Required Inputs & Data Schema
**`item_attributes`**: `item_id, category, subcategory, brand, price_band | price, pack_size, flavor/variant, channel, launch_date, lifecycle_status ∈ {PRE_LAUNCH, LAUNCH, GROWTH, MATURE, PHASE_OUT, DISCONTINUED, SUPERSEDED}, eol_date`.

**`analog_history`** (from `nextq-demand-history-cleansing` + reconstruction): `item_id, location_id, week_since_launch (1..), demand_clean, stores_ranged, in_stock_pct, period_state`. Curves use **per-store** demand $= \text{demand}/\text{stores\_ranged}$ to separate rate of sale from distribution build.

**`launch_plan`**: `item_id, week_since_launch, planned_stores, planned_price, promo_flag` (launch promos are modeled via `nextq-promotion-lift-cannibalization-planning`).

**`supersession_links`**: `old_item_id, new_item_id, link_type ∈ {REPLACE_1_1, CONSOLIDATE_N_1, SPLIT_1_N}, conversion_factor` (new units per old unit, e.g. pack 12 → 6 gives 2.0), `split_share` (for 1:N, sums to 1 per old item), `effective_date`.

**`inventory`** (phase-out): `item_id, location_id, on_hand, on_order, eol_date, successor_item_id, transfer_rate` (share of demand expected to move to successor) [RoT default 0.5 if unknown, flagged].

**`attribute_weights`**: per attribute; numeric attributes need a `range` for Gower scaling.

## 4. Execution Workflow
1. **Classify lifecycle state** (from master data and history):
   - PRE_LAUNCH: launch date in the future.
   - LAUNCH: < 13 clean weeks.
   - GROWTH: 13–25 clean weeks.
   - MATURE: ≥ 26 clean weeks (hand off to statistical skills).
   - PHASE_OUT: announced.
   - DISCONTINUED / SUPERSEDED.

   Periods before launch or after EOL are STRUCTURAL_ZERO.
2. **Analog selection (likeness):**
   - Compute weighted Gower similarity $S(i,a)=\sum_k w_k s_k/\sum_k w_k$ over non-missing attributes, where categorical $s_k=\mathbb 1[x=y]$ and numeric $s_k=1-\min(\lvert x-y\rvert/\text{range}_k,1)$.
   - Keep the top $k$ with $S ≥ 0.60$ and ≥ $H$ clean weeks.
   - If fewer than 3 qualify: use the **category pooled curve** (all launches in the subcategory in the last 3 years [RoT]), flagged `POOLED_ANALOG` and `LOW_CONFIDENCE`.
   - Document analog IDs, similarities, and the planner's accept/replace decision (a planner-chosen analog is an override: reason code required).
3. **Align analogs:**
   - Week 1 is the first ranged week with in-stock ≥ 95%.
   - Reconstruct censored launch weeks before use.
   - Convert to per-store demand.
   - If launch seasons differ, deseasonalize each analog by the category seasonal index (seasonal index at its calendar week), then re-seasonalize to the new item's calendar weeks.
4. **Shape × scale:**
   - Shape $\text{Sh}(t)=\sum_a \tilde w_a\,c_a(t)/\sum_{t\le H}c_a(t)$ with $\tilde w_a=S_a/\sum S$, so $\sum_t \text{Sh}(t)=1$.
   - Scale per store $\bar C=\sum_a\tilde w_a\sum_{t\le H}c_a(t)$.
   - Forecast $F(t)=\bar C\cdot\text{stores}(t)\cdot\text{Sh}(t)\cdot(P_{\text{new}}/P_{\text{analog}})^{\varepsilon}$. Apply the price term only when a category elasticity $\varepsilon$ is supplied (otherwise flag `NO_PRICE_ADJ`).
   - For durables with no analog, a Bass diffusion curve may be used. Cumulative adopters $N(t)=m\frac{1-e^{-(p+q)t}}{1+(q/p)e^{-(p+q)t}}$. Parameters $p,q$ come from analogs; never use generic values without a `GENERIC_BASS` flag.
5. **Range (GS-M-10):**
   - P10/P90 = weighted empirical quantiles of analog scales × stores × shape (≥ 3 analogs).
   - With pooled analogs, widen to the P5/P95 of all subcategory launches.
   - Report the cumulative-$H$ point (similarity-weighted mean) with P10/P90.
6. **Early-read re-scaling (weeks 1–12):**
   - Observed scale $\hat C_{\text{obs}}=\sum_{t\le n}A_t/\sum_{t\le n}\text{Sh}(t)$ (total units, same distribution basis).
   - Updated $C=w\hat C_{\text{obs}}+(1-w)C_{\text{prior}}$ with $w=n/(n+n_0)$.
   - Shape stays fixed until week 8; after that, the planner may switch to the best-fit analog subset (logged).
   - Censored launch weeks are reconstructed first; never re-scale on stockout-truncated sales.
7. **Handoff:** apply the blend from week 13 to 26 and the MASE-gated full handoff (Section 2 parameters). Log the handoff date.
8. **Supersession chaining:**
   - Resolve chains transitively (A→B→C, factors multiply) and reject cycles (`SUPERSESSION_CYCLE`).
   - Chained history: $H_{\text{new}}(t)=\sum_{\text{old}} H_{\text{old}}(t)\cdot f_{\text{old}}\cdot \text{share}_{\text{old}\to\text{new}}$.
   - During an overlap window (both selling), sum old (converted) + new. Never take max or duplicate.
   - Old-item periods after `effective_date` become STRUCTURAL_ZERO for the old item.
9. **Phase-out / run-out:**
   - Switch to TSB at the announcement. Forecast to EOL; after EOL the forecast is 0 (structural).
   - Excess at EOL $=\max(\text{on\_hand}+\text{on\_order}-\sum_{t\le \text{EOL}}F_t,0)$. Exhaustion period = first $t$ where cumulative $F ≥$ position.
   - Flag `RUN_OUT_EXCESS` if excess > 0 (value at cost) and `EARLY_STOCKOUT` if exhaustion precedes EOL by > lead time.
   - Successor uplift: $F_{\text{succ}}(t)\mathrel{+}=\text{transfer\_rate}\cdot F_{\text{old,counterfactual}}(t)$ for $t > \text{EOL}$, flagged `TRANSFER`. If the successor's own forecast is analog-based on **replacement** analogs (`REPLACE_1_1`), the transfer is already in that curve, so do not add it (no double count).
10. **Governance:** launch overrides use the GS-G-01 rules (UP > 20% needs approval). Launch forecasts are frozen at launch − lead time for cumulative accuracy scoring.
11. **Output** (Section 6).

## 5. Verification Snippets
```python
import numpy as np, pandas as pd

def gower_similarity(new, analog, weights, ranges):
    """Weighted Gower similarity in [0,1]. Categorical: 1 if equal; numeric: 1-|x-y|/range."""
    num = den = 0.0
    for k, w in weights.items():
        x, y = new.get(k), analog.get(k)
        if x is None or y is None:            # missing attribute: excluded from both num and den
            continue
        s = (1.0 - min(abs(x - y) / ranges[k], 1.0)) if k in ranges else float(x == y)
        num += w * s; den += w
    return num / den if den > 0 else 0.0

def launch_curve(analog_curves, sims, H):
    """analog_curves: dict id -> per-store weekly demand from launch week 1 (clean/reconstructed).
    Returns similarity-weighted normalized shape (sums to 1 over H) and per-store scale stats."""
    shapes, scales, w = [], [], []
    for a, c in analog_curves.items():
        c = np.asarray(c[:H], float)
        assert len(c) == H and c.sum() > 0, f"analog {a} lacks {H} weeks of history"
        shapes.append(c / c.sum()); scales.append(c.sum()); w.append(sims[a])
    w = np.asarray(w) / np.sum(w)
    shape = np.average(np.vstack(shapes), axis=0, weights=w)
    scale_mean = float(np.average(scales, weights=w))
    return shape, scale_mean, np.asarray(scales), w

def npi_forecast(shape, scale_per_store, stores, price_ratio=1.0, elasticity=None):
    """F(t) = scale_per_store * stores(t) * shape(t) * price adj. stores may be a ramp (array)."""
    adj = price_ratio ** elasticity if elasticity is not None else 1.0
    return scale_per_store * adj * np.asarray(stores, float) * shape

def npi_interval(shape, scales, w, stores, q=(0.1, 0.9)):
    """Empirical weighted quantiles of analog scales (needs >= 3 analogs) -> P10/P90 paths."""
    assert len(scales) >= 3, "INSUFFICIENT_ANALOGS for empirical interval"
    o = np.argsort(scales); s, cw = scales[o], np.cumsum(w[o])
    lo, hi = (s[np.searchsorted(cw, qq)] for qq in q)
    return lo * np.asarray(stores) * shape, hi * np.asarray(stores) * shape

def early_read_update(shape, prior_scale_total, actuals, n0=4):
    """Re-scale after n observed weeks: obs scale = sum(A)/sum(shape[:n]); weight n/(n+n0) [RoT n0=4]."""
    n = len(actuals); cum_shape = shape[:n].sum()
    obs = np.sum(actuals) / cum_shape
    w = n / (n + n0)
    return w * obs + (1 - w) * prior_scale_total, w, obs

def chain_supersession(links):
    """links: list of (old, new, factor). Returns old -> (terminal new, cumulative factor). Detects cycles."""
    nxt = {o: (n, f) for o, n, f in links}
    out = {}
    for o in nxt:
        seen, cur, fac = {o}, o, 1.0
        while cur in nxt:
            n, f = nxt[cur]; fac *= f
            if n in seen: raise ValueError(f"SUPERSESSION_CYCLE at {n}")
            seen.add(n); cur = n
        out[o] = (cur, fac)
    return out

def tsb(y, alpha=0.1, beta=0.1, pi0=None, z0=None):
    """GS-M-06 TSB: pi updated every period; forecast pi*z."""
    y = np.asarray(y, float); nz = y[y > 0]
    pi = (y > 0).mean() if pi0 is None else pi0
    z = nz.mean() if z0 is None else z0
    for v in y:
        if v > 0: pi += beta * (1 - pi); z += alpha * (v - z)
        else:     pi += beta * (0 - pi)
    return pi * z, pi, z

def run_out(on_hand, on_order, fcst_until_eol):
    """Excess at EOL = position - cumulative forecast to EOL; exhaustion period index (or None)."""
    pos = on_hand + on_order; cum = np.cumsum(fcst_until_eol)
    idx = int(np.argmax(cum >= pos)) if (cum >= pos).any() else None
    return max(pos - cum[-1], 0.0), idx
```
**Test vector** (executed):
- **New item** {snack, brand X, $3.00, 6-pack}. Weights: category 3, brand 2, price 2 (range $2.00), pack 1 (range 12).
  - Similarities: A = 0.975, B = 0.6875, C = 0.8125. All three are ≥ 0.60, so all qualify.
- **8-week per-store curves:** A = [2,4,6,6,5,5,4,4], B = [1,3,5,5,5,4,4,3], C = [3,5,5,4,4,3,3,3].
  - Shape = [0.0640, 0.1263, 0.1667, 0.1557, 0.1448, 0.1246, 0.1136, 0.1044]. Scale = 32.364 units/store.
- **Distribution ramp** 50 → 100 stores: 8-week forecast = 3,132.8 units. The 100-store P10/P90 = 3,000 / 3,600.
- **Early read** after 2 weeks [250, 420]: observed total scale 3,521.9 → updated 3,331.6 (w = 0.333).
- **Chain:** A→B (1.0), B→C (0.5) gives A and B → C with factor 0.5. A→B→A raises `SUPERSESSION_CYCLE at A`.
- **TSB** on [3,0,2,0,0,0,0,0,0,0]: π = 0.156, z = 2.495, forecast 0.390.
- **Run-out:** position 50 vs. 6 × 5 forecast → excess 20, no exhaustion before EOL.

```sql
-- Analog eligibility: clean launch weeks and censoring share per candidate analog (H = 26)
SELECT item_id,
       COUNT(*) FILTER (WHERE period_state <> 'STRUCTURAL_ZERO' AND week_since_launch <= 26)       AS clean_weeks,
       AVG(CASE WHEN in_stock_pct < 0.95 THEN 1.0 ELSE 0 END) FILTER (WHERE week_since_launch <= 26) AS censored_share,
       SUM(demand_clean / NULLIF(stores_ranged,0)) FILTER (WHERE week_since_launch <= 26)          AS scale_per_store
FROM analog_history
GROUP BY item_id
HAVING COUNT(*) FILTER (WHERE period_state <> 'STRUCTURAL_ZERO' AND week_since_launch <= 26) >= 26;
```

## 6. Standardized Output Format
```markdown
# Lifecycle Forecast — <scope> — run NPL_<run_id>

## Executive Summary
- <N_new> launches forecast (point <u> units over H=26 wks, range P10 <lo> – P90 <hi>); <n_pool> on POOLED_ANALOG (low confidence).
- <N_early> early reads re-scaled: <up>/<down> vs prior; largest change <item> (<x>%).
- <N_sup> supersession chains applied (<n_cycle> cycles rejected); <N_po> phase-outs: EOL excess <units> (<value> at cost).
- <N_hand> items handed off to statistical forecasting this cycle.

## Metadata (mandatory)
| Level | Grain | Horizon H | Time window (analogs) | Similarity method / threshold | Analogs k | Weighting | Lag (scoring) | Early-read n0 |
|---|---|---|---|---|---|---|---|---|
| SKU-Location (store-count basis) | WEEK | 26 | <launch years> | weighted Gower / 0.60 | 3–5 | similarity-weighted | launch − LT | 4 |

## KPI Table (launch accuracy, cumulative H, frozen at launch − LT)
| Segment | N launches | Cum. WAPE % | Cum. Bias % (+=over) | % within P10–P90 | Override FVA (WAPE pts) |
|---|---|---|---|---|---|

## Launch Forecasts
| item | launch wk | analogs (sim) | stores plan | P10 | point | P90 | early-read w | updated point | state | flags |
|---|---|---|---|---|---|---|---|---|---|---|

## Supersession Chains
| old item | new item | type | factor | share | effective | chained periods | overlap periods | check |
|---|---|---|---|---|---|---|---|---|

## Phase-Out / Run-Out
| item | loc | EOL | TSB π | TSB z | fcst to EOL | position | excess units | excess value | exhaustion wk | successor transfer |
|---|---|---|---|---|---|---|---|---|---|---|

## Exceptions
| ID | Trigger (rule) | item | metric | value | action | owner |
|---|---|---|---|---|---|---|
| | POOLED_ANALOG / LOW_CONFIDENCE (GS-D-12) | | analogs ≥0.60 | | review analog choice | |
| | UP override >20% unapproved (GS-G-01) | | | | approval required | |
| | Early read outside P10–P90 (GS-M-10) | | | | re-plan supply | |
| | RUN_OUT_EXCESS / EARLY_STOCKOUT (GS-D-12) | | | | markdown / cancel PO / expedite | |
| | SUPERSESSION_CYCLE (GS-D-13) | | | | fix master data | |

## Recommendations
1. …

## Data-Quality Log
| item | issue (censored launch weeks reconstructed / missing attributes / no store counts / generic transfer rate) | treatment |
|---|---|---|

## Reproducibility
Analog IDs + similarities, attribute weights/ranges, H, n0, handoff rule, TSB α/β, supersession table version, code §5.
```

## 7. Edge Cases
- **No store counts** for analogs: use total-demand curves and flag `NO_DISTRIBUTION_BASIS`. The scale then assumes the same distribution as the analog.
- **Analog launched with a promo** but the new item launches without one (or vice versa): remove analog promo lift using the `nextq-promotion-lift-cannibalization-planning` baseline before shaping.
- **Censored launch** (stockout in the first weeks): early-read re-scaling uses reconstructed demand. If more than 30% of the observed weeks are reconstructed, keep $w$ at most 0.25 [RoT] and flag it.
- **New item cannibalizes the old range** (line extension): forecast the incremental and the cannibalized volume separately (GS-D-06), and reduce the affected items' forecasts in consensus with reason NEW_LISTING.
- **1:N split** without split shares: use the new items' planned distribution shares and flag `ASSUMED_SPLIT`.
- **Re-launch / re-listing** after a gap longer than one cycle: treat it as a new launch with the old item as the top analog; do not chain across the gap.
- **Item never reaching 26 clean weeks** (short-life or seasonal one-shot): stays on analog forecasting through EOL; inventory uses the newsvendor logic in `nextq-replenishment-policy-design`.
- **Negative or zero analog scale** (returns-dominated): exclude that analog and log it.
