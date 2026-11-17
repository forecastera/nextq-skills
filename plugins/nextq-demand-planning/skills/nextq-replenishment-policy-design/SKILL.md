---
name: nextq-replenishment-policy-design
description: >-
  [Next Quarter Supply Chain Suite] Designs and tunes inventory replenishment policies per SKU-location: chooses
  (s,Q), (R,S) or (s,S) review policies, sizes order quantities with EOQ plus pack, MOQ and shelf-life rounding,
  converts safety stock into reorder points and order-up-to levels, sets single-period newsvendor quantities, and
  validates every policy by demand simulation for cycle service level, unit fill rate and total cost. Load when the
  user asks to set reorder points, order-up-to or min/max levels, pick a review period or order quantity, compare
  continuous vs periodic review, size a seasonal or perishable buy, or check that a policy hits its service target.
  Inputs: demand forecast and error, lead time, safety-stock outputs, costs, pack and MOQ rules, shelf life. Triggers:
  "reorder point", "min/max", "EOQ", "order-up-to", "review period", "newsvendor".
metadata:
  author: "Next Quarter"
  organization: "Next Quarter (nextq)"
  suite: "Next Quarter Supply Chain Intelligence"
  version: "1.0.0"
---

# Replenishment Policy Design
*Part of the **Next Quarter (nextq)** Supply Chain Skill Suite — Governed by SUPPLY_CHAIN_GOLD_STANDARD.md.*

## 1. Role & Objective
**Persona:** Inventory Policy Designer / Replenishment Planning Lead.
**Decision supported:** Which replenishment policy and parameters (s, Q, R, S, or a one-time quantity) should each item-location run, so that it meets its service target at the lowest ordering-plus-holding cost?

**Objective:** A policy record per item-location that
- is built on `nextq-safety-stock-optimization` outputs,
- respects operational constraints (MOQ, pack, shelf life, review calendar),
- is verified by simulation against its service target (cycle service level and fill rate reported separately), and
- is costed, ready for the planning system and for `nextq-inventory-health-optimization` (max-stock limits).

## 2. Gold Standard Rules Enforced
| Rule | Inlined standard |
|---|---|
| **GS-M-12** CSL policies [STD] | $\mathit{k}=\Phi^{-1}(\text{CSL})$, $\sigma_{DL}=\sqrt{\bar L\sigma_d^2+\bar d^2\sigma_L^2}$, $\mathit{SS}=k\sigma_{DL}$, $\mathit{ROP}=\bar d\bar L+\mathit{SS}$. Periodic review (R, S): the protection interval is $\mathit{R}+L$, so $\mathit{S}=\bar d(R+L)+\mathit{SS}_{R+L}$. $\sigma_d$ = forecast-error σ at lag and bucket, not demand σ. No normal-k for intermittent or count items, or when $\sigma_{DL}/\mu_{DL}>0.5$ [RoT]. **SS values come from `nextq-safety-stock-optimization`; they are not recomputed here.** |
| **GS-M-13** Fill rate [STD] | $\mathit{G}(k)=\phi(k)-k[1-\Phi(k)]$. (s, Q): $\beta=1-\sigma_{DL}G(k)/Q$. (R, S): $\beta=1-\sigma_{R+L}G(k)/(\bar dR)$. A negative $\mathit{k}$ is floored and reported. **CSL ≠ fill rate:** both are reported for every policy. |
| **GS-M-14** Newsvendor [STD] | Critical ratio $\mathit{CR}=C_u/(C_u+C_o)$, with $\mathit{C}_u$ = lost margin + goodwill and $\mathit{C}_o$ = cost − salvage + holding. $\mathit{Q}^*=F^{-1}(\mathit{CR})$ from the forecast **distribution** (Poisson / NB / empirical for counts). Used for seasonal, perishable and one-shot buys. Report expected leftover $\mathit{E}[(Q-D)^+]$, expected shortage $\mathit{E}[(D-Q)^+]$ and fill rate $1-E[(D-Q)^+]/E[D]$. |
| **GS-M-10** Probabilistic | Policies are evaluated on simulated demand paths. Lead-time totals come from summed paths, never summed quantiles. |
| **GS-K-08 / GS-K-09** | CSL = cycles without stockout / cycles (cycle qualifier always stated). Unit fill rate = units filled from on-hand at the time of demand / units demanded (unconstrained). |
| **GS-K-11** DOS | Average on-hand from the simulation is converted to **forward** days of supply against the forecast (stated as forward, on-hand only). |
| **GS-G-09** Segmentation | Policy family and targets follow ABC × XYZ × demand class (rule table below). Default targets A/B/C = 98/95/90% [RoT], as in the safety stock skill. |
| **GS-G-11 / GS-G-10** | Every policy is spot-checked by simulation; standard output. |

**Policy family rule [RoT, configurable]:**

| Condition | Policy |
|---|---|
| Single selling season, perishable, or one-shot buy | Newsvendor |
| Joint ordering / fixed supplier calendar | (R, S), with R = supplier cycle |
| Intermittent or lumpy (ADI ≥ 1.32) with MOQ | (s, S) |
| A items with continuous visibility | (s, Q) |
| Otherwise | (R, S), with R = the review calendar rounded from EOQ/D |

**Order quantity:**
- $\mathit{EOQ}=\sqrt{2DK/h}$, with $\mathit{h}$ = holding rate × unit cost.
- Rounded up to the pack multiple and to at least the MOQ.
- Capped at the shelf-life quantity (max days × daily forecast). If the MOQ exceeds the cap → `MOQ_EXCEEDS_SHELF_LIFE`.
- (s, S): $\mathit{S}=s+Q_{EOQ}$ [RoT].

**Simulation timing (deterministic):**
- Each period: receive orders due → serve demand (backorder or lost-sales mode) → review and order.
- An order placed at the end of period $\mathit{t}$ arrives at the start of $\mathit{t}+L+1$, so exactly $\mathit{L}$ periods of demand fall within the lead time.
- A cycle is the time between consecutive receipts; it has a stockout if any demand in it went unfilled.

**Acceptance rules [RoT]:**
- A policy passes if simulated service ≥ target − 0.5 pp.
- (s, Q) and (s, S) reviewed per period overshoot their trigger: expected undershoot $\mathit{E}[U]\approx(\mu^2+\sigma^2)/(2\mu)$ per review period (a renewal-theory approximation). Raise $\mathit{s}$ by $\mathit{E}[U]$ first, then **bisection-search** $\mathit{s}$ (or $\mathit{S}$) on common random demand paths until the target is met.

## 3. Required Inputs & Data Schema
| Table | Columns | Notes |
|---|---|---|
| `ss_results` (from `nextq-safety-stock-optimization`) | `item_id, location_id, route, mu_period, sigma_d, L, sigma_L, ss_L, ss_RL, target_type, target, implied_csl, implied_fill` | do not recompute |
| `costs` | `unit_cost, order_cost K, holding_rate, Cu, Co (newsvendor), salvage` | annual holding rate |
| `constraints` | `moq, pack_size, shelf_life_days, max_days, supplier_review_period R, order_days` | |
| `forecast` | `period, F, quantiles or distribution params` | simulation input |
| `segmentation` | `abc, xyz, demand_class, ADI, season_flag, perishable_flag` | policy family |
| `current_policy` | `policy, s, Q, R, S` | comparison |

**Grain:** item × location × bucket (the same bucket as the lead time, which may be fractional; simulation uses an integer L at a finer grain if needed). **Minimum:** a simulation of 40 seasonal cycles (e.g. 2,080 weeks) [RoT].

## 4. Execution Workflow
1. **Policy family** from the rule table.
2. **Order quantity:** EOQ → round to pack and MOQ → shelf-life cap (flags). For newsvendor items, skip to step 6.
3. **Levels** from SS results:
   - (s, Q): $\mathit{s}=\mu L+\mathit{SS}_L$
   - (R, S): $\mathit{S}=\mu(R+L)+\mathit{SS}_{R+L}$
   - (s, S): $\mathit{s}=\mu L+\mathit{SS}_L$ and $\mathit{S}=s+Q$

   Apply the undershoot adjustment to (s, Q) and (s, S).
4. **Simulate** with seeded demand paths (forecast ± error σ, using the same distribution route as the SS skill: normal, Poisson/NB, or gamma/empirical) and lead-time draws. Record CSL, fill rate, average on-hand, orders and stockout units.
5. **Search** if service < target − 0.5 pp: bisect $\mathit{s}$ (or $\mathit{S}$) on common random paths, and record the simulated level that replaces the closed form (`SIM_ADJUSTED`).
6. **Newsvendor:**
   - $\mathit{CR}=C_u/(C_u+C_o)$ and $\mathit{Q}^*=F^{-1}(\mathit{CR})$.
   - Report leftover, shortage and fill rate.
   - Multi-store buys are allocated by store forecast shares, and the sum of rounded store quantities must equal $\mathit{Q}^*$ (largest remainder).
7. **Cost:** ordering $= \text{orders/yr}\times K$; holding $= \bar I\times c\times h$. Shortage is treated as a **service constraint**, not a cost, unless the user supplies a shortage cost.
8. **Choose:** lowest cost among candidate policies meeting the target; if none meets it, the highest-service policy, flagged `TARGET_UNREACHABLE`.
9. **Publish:** policy record, max stock (= $\mathit{s}+Q$ or $\mathit{S}$) for inventory health, forward DOS of average on-hand, and the delta vs. the current policy (cost, inventory value, service).
10. **Output** (Section 6).

## 5. Verification Snippets
```python
import numpy as np
from scipy import stats

def eoq(D_annual, K_order, unit_cost, holding_rate):
    """EOQ = sqrt(2 D K / h), h = holding_rate * unit_cost (per unit per year)."""
    return float(np.sqrt(2 * D_annual * K_order / (holding_rate * unit_cost)))

def round_order_qty(q, moq=0, pack=1, shelf_cap=None):
    """Round up to pack multiple, at least MOQ; cap by shelf-life quantity (flag if MOQ exceeds the cap)."""
    q = max(q, moq); q = pack * np.ceil(q / pack)
    flag = None
    if shelf_cap is not None and q > shelf_cap:
        q2 = pack * np.floor(shelf_cap / pack)
        flag = "MOQ_EXCEEDS_SHELF_LIFE" if q2 < moq else "SHELF_LIFE_CAPPED"
        q = max(q2, moq) if flag == "SHELF_LIFE_CAPPED" else q
    return float(q), flag

def policy_levels(policy, mu_period, ss_L=None, ss_RL=None, L=1, R=1, Q=None):
    """Policy parameters from nextq-safety-stock-optimization outputs (SS over L or R+L):
    sQ: s = mu*L + SS_L ; RS: S = mu*(R+L) + SS_RL ; sS: s = mu*L + SS_L, S = s + Q (Q = EOQ) [RoT]."""
    if policy == "sQ": return dict(s=mu_period * L + ss_L, Q=Q)
    if policy == "RS": return dict(R=R, S=mu_period * (R + L) + ss_RL)
    if policy == "sS": s = mu_period * L + ss_L; return dict(s=s, S=s + Q)
    raise ValueError(policy)

def newsvendor(cu, co, dist):
    """GS-M-14: CR = Cu/(Cu+Co); Q* = F^-1(CR). dist: frozen scipy distribution of single-period demand.
    Returns Q*, CR, expected leftover E[(Q-D)^+], expected shortage E[(D-Q)^+], fill rate 1 - E[short]/E[D]."""
    cr = cu / (cu + co); q = float(dist.ppf(cr))
    if hasattr(dist.dist, "pmf"):
        x = np.arange(0, int(dist.ppf(1 - 1e-10)) + 2); p = dist.pmf(x)
    else:
        x = np.linspace(dist.ppf(1e-8), dist.ppf(1 - 1e-8), 20001); p = dist.pdf(x); p = p / p.sum()
    left = float(np.sum(np.maximum(q - x, 0) * p)); short = float(np.sum(np.maximum(x - q, 0) * p))
    return dict(Q=q, CR=cr, exp_leftover=left, exp_short=short, fill_rate=1 - short / dist.mean())

def simulate(policy, params, demand, L, backorder=True, init=None):
    """Discrete-period inventory simulation (GS-G-11 check of a policy).
    Timing per period t: receive orders due at t -> serve demand -> review at end of t and order.
    An order placed at end of t arrives at start of t+L+1, so L full periods of demand fall inside the lead time
    (protection interval L for sQ/sS, R+L for RS). Captures undershoot that closed forms ignore.
    Returns CSL (replenishment cycles without stockout / cycles), unit fill rate, mean on-hand, number of orders."""
    D = np.asarray(demand, float); T = len(D)
    on_hand = float(init if init is not None else params.get("S", params.get("s", 0) + params.get("Q", 0)))
    back = 0.0; pipe = {}; filled = 0.0; cycles = cyc_out = 0; cycle_flag = False; orders = 0; oh_sum = 0.0
    for t in range(T):
        if t in pipe:
            q = pipe.pop(t); on_hand += q
            serve = min(back, on_hand) if backorder else 0.0; back -= serve; on_hand -= serve
            cycles += 1; cyc_out += cycle_flag; cycle_flag = False
        d = D[t]; f = min(d, on_hand); filled += f; on_hand -= f
        if d - f > 0:
            cycle_flag = True
            if backorder: back += d - f
        oh_sum += on_hand
        ip = on_hand - back + sum(pipe.values())
        q = 0.0
        if policy == "sQ" and ip <= params["s"]:
            q = params["Q"] * np.ceil((params["s"] - ip) / params["Q"] + 1e-12)
        elif policy == "RS" and t % params["R"] == 0:
            q = max(params["S"] - ip, 0.0)
        elif policy == "sS" and ip <= params["s"]:
            q = params["S"] - ip
        if q > 0:
            pipe[t + L + 1] = pipe.get(t + L + 1, 0.0) + q; orders += 1
    return dict(csl=1 - cyc_out / max(cycles, 1), fill_rate=filled / D.sum(), avg_on_hand=oh_sum / T, orders=orders)

def total_cost(sim, T_years, K_order, unit_cost, holding_rate):
    """Ordering + holding cost per year (shortage is handled as a service constraint, not a cost)."""
    return sim["orders"] / T_years * K_order + sim["avg_on_hand"] * unit_cost * holding_rate

def choose_policy(candidates, target, kind="fill_rate", tol_pp=0.5):
    """Lowest total cost among policies whose simulated service >= target - 0.5 pp [RoT]; else highest service."""
    ok = [c for c in candidates if c["sim"][kind] >= target - tol_pp / 100]
    pool = ok if ok else candidates
    key = (lambda c: c["cost"]) if ok else (lambda c: -c["sim"][kind])
    return min(pool, key=key)["name"], bool(ok)

def undershoot_adjust(s, mu_period, sd_period):
    """Periodic-review of an (s,Q)/(s,S) trigger: IP crosses s by an undershoot U. Renewal-theory approximation
    E[U] ~ (mu^2 + sigma^2) / (2 mu) per review period [STD approximation]; raise s by E[U] and re-simulate."""
    return s + (mu_period ** 2 + sd_period ** 2) / (2 * mu_period)

def search_reorder_point(policy, base, demand, L, target, kind="csl", lo=0.0, hi=None, tol=0.5, max_iter=60):
    """Bisection on s (sQ/sS; S moves with s for sS) or S (RS) until simulated `kind` >= target, using the SAME
    fixed demand paths (common random numbers) so the search is deterministic. Returns level and its simulation."""
    key = "S" if policy == "RS" else "s"
    hi = hi if hi is not None else 3 * base[key] + 10 * np.std(demand) * (L + 1)
    def run(x):
        p = dict(base); p[key] = x
        if policy == "sS": p["S"] = x + (base["S"] - base["s"])
        return simulate(policy, p, demand, L)
    if run(hi)[kind] < target: return hi, run(hi)
    for _ in range(max_iter):
        mid = 0.5 * (lo + hi)
        if run(mid)[kind] >= target: hi = mid
        else: lo = mid
        if hi - lo < tol: break
    return hi, run(hi)

if __name__ == "__main__":                                  # self-test: reproduces the documented test vector
    rng = np.random.default_rng(1); dem = np.maximum(rng.normal(100, 20, 2080), 0); L = 2
    assert abs(eoq(5200, 50, 10, 0.25) - 456.07) < 0.01
    assert round_order_qty(456.07, moq=300, pack=24) == (480.0, None)
    assert round_order_qty(456.07, moq=300, pack=24, shelf_cap=450) == (432.0, "SHELF_LIFE_CAPPED")
    assert round_order_qty(456.07, moq=600, pack=24, shelf_cap=500)[1] == "MOQ_EXCEEDS_SHELF_LIFE"
    nv = newsvendor(6, 2, stats.norm(100, 20)); assert abs(nv["Q"] - 113.49) < 0.01 and abs(nv["fill_rate"] - 0.970) < 0.001
    nvp = newsvendor(3, 1, stats.poisson(4)); assert nvp["Q"] == 5 and abs(nvp["fill_rate"] - 0.897) < 0.001
    k = stats.norm.ppf(0.95)
    rs = policy_levels("RS", 100, ss_RL=k * 20 * np.sqrt(3), L=L, R=1); sim_rs = simulate("RS", rs, dem, L)
    sq = policy_levels("sQ", 100, ss_L=k * 20 * np.sqrt(2), L=L, Q=400); sim_sq = simulate("sQ", sq, dem, L)
    s_adj, sim_s = search_reorder_point("sQ", sq, dem, L, 0.95)
    assert abs(rs["S"] - 356.98) < 0.01 and abs(sim_rs["csl"] - 0.945) < 0.001 and abs(sim_rs["fill_rate"] - 0.9925) < 1e-4
    assert abs(sq["s"] - 246.52) < 0.01 and abs(sim_sq["csl"] - 0.434) < 0.001
    s_u = undershoot_adjust(sq["s"], 100, 20); assert abs(s_u - 298.52) < 0.01
    assert abs(simulate("sQ", dict(sq, s=s_u), dem, L)["csl"] - 0.857) < 0.001
    assert abs(s_adj - 322.87) < 0.01 and sim_s["csl"] >= 0.95 and abs(sim_s["fill_rate"] - 0.998) < 0.001
    cands = [dict(name=n, sim=sm, cost=total_cost(sm, 40, 50, 10, 0.25)) for n, sm in (("sQ", sim_sq), ("RS", sim_rs))]
    assert [round(c["cost"]) for c in cands] == [1026, 2746] and choose_policy(cands, 0.99) == ("RS", True)
    print("self-test passed")
```
**Test vector** (verified by the `__main__` self-test above; 2,080 weeks of N(100, 20²) demand, seed 1, L = 2, target 95% CSL):
- **Order quantity:**
  - EOQ(D = 5,200, K = 50, c = 10, h = 25%) = **456.07** → pack 24 / MOQ 300 → **480**.
  - Shelf cap 450 → **432** (`SHELF_LIFE_CAPPED`).
  - MOQ 600 with cap 500 → `MOQ_EXCEEDS_SHELF_LIFE`.
- **Newsvendor:**
  - Normal(100, 20), Cu = 6, Co = 2 → CR 0.75, Q* = **113.49**, leftover 16.47, shortage 2.98, fill rate 0.970.
  - Poisson(4), Cu = 3, Co = 1 → Q* = 5, fill rate 0.897.
- **(R, S)** with R = 1: S = 356.98 → simulated CSL **0.945** (target 0.95, within 0.5 pp) and fill rate 0.992. This confirms the timing convention.
- **(s, Q)** with Q = 400:
  - Closed-form s = 246.52 → simulated CSL **0.434**. The undershoot is about half a week of demand, larger than SS.
  - Undershoot-adjusted s = 298.52 → CSL 0.857.
  - Bisection search → s = **322.87**, CSL **0.952**, fill rate 0.998 (`SIM_ADJUSTED`).
- **Choice at a 99% fill-rate target:**

  | Policy | Cost/yr | Simulated fill rate | Result |
  |---|---|---|---|
  | (s, Q), unadjusted | 1,026 | 0.949 | misses target |
  | (R, S) | 2,746 | 0.9925 | chosen |

```sql
-- Policy record sanity checks (each query must return zero rows)
SELECT item_id, location_id FROM policy_record WHERE policy = 'sQ' AND (q < moq OR MOD(q, pack_size) <> 0);
SELECT item_id, location_id FROM policy_record WHERE policy = 'sS' AND big_s <= small_s;
SELECT item_id, location_id FROM policy_record
WHERE (COALESCE(small_s,0) + COALESCE(q,0) > max_days * daily_fcst OR COALESCE(big_s,0) > max_days * daily_fcst)
  AND COALESCE(flag,'') NOT IN ('MOQ_EXCEEDS_SHELF_LIFE','SHELF_LIFE_CAPPED');
SELECT item_id, location_id FROM policy_record WHERE sim_service < target - 0.005 AND COALESCE(flag,'') <> 'TARGET_UNREACHABLE';
```

## 6. Standardized Output Format
```markdown
# Replenishment Policy Design — <scope> — run POL_<run_id>

## Executive Summary
- <N> item-locations: (s,Q) <a>, (R,S) <b>, (s,S) <c>, newsvendor <d>; <m> SIM_ADJUSTED, <u> TARGET_UNREACHABLE.
- Cost (ordering + holding) <C>/yr vs current <C0> (<Δ>%); average inventory <I> (<value>), forward DOS <dos>.
- Simulated service: CSL <x>% / fill rate <y>% vs targets (both reported).
- Constraints binding: MOQ <n1>, shelf life <n2>.

## Metadata (mandatory)
| Level | Grain | Lead time basis | Time window (simulation) | Demand distribution | Service measure | Weighting | Seed | SS source |
|---|---|---|---|---|---|---|---|---|
| SKU-Location | WEEK | L = <L> (+σ_L) | 2,080 periods | normal / Poisson / NB / empirical | CSL & fill rate | units; cost at unit cost | <seed> | SS run <id> |

## KPI Table
| Segment | Policy mix | Sim CSL | Sim fill rate | Target (type) | Avg on-hand | Fwd DOS | Orders/yr | Cost/yr | Δ cost vs current |
|---|---|---|---|---|---|---|---|---|---|

## Policy Records
| item | loc | policy | s | Q | R | S | closed-form level | sim-adjusted level | sim CSL | sim FR | cost/yr | max stock | flags |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|

## Newsvendor Buys
| item | season | CR | Q* | exp leftover | exp shortage | fill rate | store split check |
|---|---|---|---|---|---|---|---|

## Exceptions
| ID | Trigger (rule) | item | metric | value | action | owner |
|---|---|---|---|---|---|---|
| | TARGET_UNREACHABLE (GS-M-12/13) | | sim service | | review LT / MOQ / target | Inventory lead |
| | MOQ_EXCEEDS_SHELF_LIFE | | | | renegotiate MOQ / accept waste | Procurement |
| | SIM_ADJUSTED >10% vs closed form [RoT] | | Δ level | | review distribution | Analyst |
| | Negative SS floored (GS-M-13) | | k | | cycle stock gives service | Inventory planner |

## Recommendations
1. …

## Data-Quality Log
| item | issue (missing costs → defaults, fractional LT simulated at finer grain, no σ_L) | treatment |
|---|---|---|

## Reproducibility
SS run ID, costs version, constraints, seed, number of periods, acceptance tolerance, code §5.
```

## 7. Edge Cases
- **Lost-sales environments** (retail shelves): simulate with `backorder=False`. The fill rate is then the share of demand served, and stockout demand is lost, not carried. Use reconstructed demand (`nextq-unconstrained-demand-reconstruction`) as the simulation input.
- **Lead time longer than the review period with multiple orders outstanding:** the inventory position includes all open orders. The simulation handles this by construction.
- **Fractional lead times:** simulate at a finer grain (days), or round up and flag `LT_ROUNDED_UP` (conservative).
- **Seasonal items** with an off-season: time-phase the levels by period. The simulation uses the seasonal forecast path, not a stationary mean.
- **Joint replenishment** (shared truck, supplier minimum): R is set by the supplier group. Item-level S values are computed per item, and the group minimum is checked after the order sum.
- **Intermittent items:** use (s, S) with the count distribution from the SS skill. EOQ on a tiny mean can be below 1, in which case Q = max(MOQ, pack) and is flagged.
- **Perishables with FEFO:** the newsvendor or (R, S) with an S cap at shelf-life demand. Expiry losses are counted as holding loss in the simulation when shelf life is shorter than the protection interval.
