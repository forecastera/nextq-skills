# Supply Chain Gold Standard Reference (v1.0)

**Status:** v1.0, in use. This is the master reference. Each skill in `skills/` inlines the rules it enforces, so every skill works standalone.
**Scope:** Mandatory standards that all 25 demand skills must enforce. The domains are Demand Forecasting, Demand Planning, Demand Sensing, and Demand-Supply Matching.
**How to use:** Each rule has a stable ID: `GS-M-xx` for math and statistics, `GS-K-xx` for KPIs, `GS-D-xx` for data hygiene and granularity, and `GS-G-xx` for governance. Every `SKILL.md` must cite the IDs it enforces, and its output must report any rule it could not satisfy.

**Evidence labels used throughout:**
- **[STD]**: a definition or formula from a primary or authoritative source (textbook, peer-reviewed paper, or professional body).
- **[EVID]**: an empirical finding from published research.
- **[RoT]**: a practitioner rule of thumb or a configurable house default. It is *not* an industry benchmark, so organizations should calibrate it.

> No industry "good accuracy" benchmark numbers (for example, "WAPE < 30% is world-class") appear in this document. Published benchmarks depend heavily on industry, level, lag, and metric definition. Skills must benchmark against the **naïve forecast (FVA)** and against the organization's **own history**, not against generic external numbers.

---

## 0. Notation and Conventions (apply everywhere)

| Symbol | Meaning |
|---|---|
| $A_t$ (or $y_t$) | Actual demand in period $t$. Use unconstrained demand where available; see GS-M-11. |
| $F_t$ (or $\hat y_t$) | Forecast for period $t$, frozen at a stated lag $\ell$ (written $F_{t\mid t-\ell}$) |
| $e_t = A_t - F_t$ | Forecast error. Sign convention: Hyndman & Athanasopoulos, FPP3 §5.8 |
| $n$ | Number of evaluation periods (or item-periods) |
| $m$ | Seasonal period (12 monthly, 52 weekly, 7 daily-weekly) |
| $L$, $R$ | Replenishment lead time and review period, in base time buckets |
| $\Phi,\ \phi$ | Standard normal CDF and PDF |

**GS-M-01 Error-sign and lag convention [STD].** Use $e_t = A_t - F_t$. Every accuracy figure must state: (a) the **lag** at which the forecast was frozen, (b) the **aggregation level** (for example, SKU-DC-week), (c) the **time window**, and (d) the **weighting**. By default, measure at the lag that matches the cumulative lead time or the decision frozen zone [RoT]. A forecast that is not frozen at a known lag must not be scored.

---

## 1. Mathematical & Statistical Rigor (GS-M)

### GS-M-02 Benchmark first [STD]
Every forecasting skill must produce and score these naïve benchmarks:
- **Naïve (NF1, random walk):** $\hat y_{T+h\mid T} = y_T$
- **Seasonal naïve (NF2):** $\hat y_{T+h\mid T} = y_{T+h-m(k+1)}$, where $k=\lfloor (h-1)/m \rfloor$
- **Mean / moving average:** optional third benchmark (SAS/Gilliland allow a simple moving average as a naïve model)

If a candidate model does not beat the appropriate naïve benchmark out of sample, it is not used. This follows FPP3 §5.2 ("If not, the new method is not worth considering.") and Gilliland's FVA framework.

### GS-M-03 Demand-pattern classification (SBC 2005) [STD]
Compute these on the **cleaned, unconstrained** history at the planning bucket:
$$\text{ADI} = \frac{\text{number of periods}}{\text{number of periods with } y_t>0},\qquad \text{CV}^2 = \left(\frac{\sigma(y_t \mid y_t>0)}{\mu(y_t \mid y_t>0)}\right)^2$$

| Class | ADI | CV² | Default method family |
|---|---|---|---|
| Smooth | < 1.32 | < 0.49 | ETS / ARIMA / regression / ML |
| Erratic | < 1.32 | ≥ 0.49 | ETS/ARIMA with robust intervals; ML with drivers; SBA if zeros occur |
| Intermittent | ≥ 1.32 | < 0.49 | SBA / TSB / ADIDA-MAPA |
| Lumpy | ≥ 1.32 | ≥ 0.49 | SBA / TSB / temporal aggregation, plus a compound distribution for inventory |

**Mandatory caveats:**
- The 1.32 and 0.49 cut-offs were derived to choose **between Croston and SBA** (Syntetos, Boylan & Croston, 2005). They are not universal demand laws.
- Kostenko & Hyndman (2006) showed that the exact boundary is non-linear, with limits p = 4/3 and v = 0.5, and that SBA is preferable over most of the region. Skills must therefore treat the quadrant as a **routing heuristic**, confirm the routing with out-of-sample accuracy (GS-M-16), and use SBA rather than Croston as the default intermittent estimator.
- Do not classify a series with fewer than 2 non-zero observations or fewer than one full seasonal cycle. Route it to the **new / sparse-item** path (GS-D-12) [RoT].

### GS-M-04 Model-selection decision rules
1. **Clean → classify → candidate set → rolling-origin backtest → pick on the primary metric → check bias → reconcile.** This order is mandatory.
2. **Smooth / erratic, single series:** ETS selected by minimum AICc over admissible models (Hyndman et al., 2002), and ARIMA via the Hyndman-Khandakar (2008) stepwise AICc search. Use AICc for within-family selection only. Use rolling-origin cross-validation (GS-M-16) to choose **across** families [STD].
3. **Weekly data (non-integer m ≈ 52.18):** STL + non-seasonal ETS, or dynamic harmonic regression (Fourier terms, K chosen by AICc). **Daily data:** multiple seasonality via STL, dynamic harmonic regression, or Prophet-style models, plus explicit holiday/event dummies (FPP3 §13.1) [STD].
4. **Intermittent / lumpy:** SBA (default), TSB (when obsolescence or demand decay is plausible), or ADIDA/MAPA (iMAPA) temporal aggregation (GS-M-06/07).
5. **Causal drivers present** (price, promo, weather, events): regression with ARIMA errors (dynamic regression) or a global ML model (GS-M-09).
6. **Combinations:** when no single model dominates in the backtest, prefer a simple equal-weight average of the top candidates (Bates & Granger 1969; Wang et al. 2023; FPP3 §13.4) [STD/EVID].
7. **Short series:** when there is too little history for holdout, the AICc minimum is acceptable (it proxies one-step out-of-sample MSE). FPP3 §13.7 states that sample-size "magic numbers" such as 30 are unsubstantiated [STD].

### GS-M-05 Exponential smoothing and ARIMA essentials [STD]
- **SES:** $\ell_t = \alpha y_t + (1-\alpha)\ell_{t-1}$, $\hat y_{t+h\mid t}=\ell_t$, $0<\alpha<1$
- **ETS(A,A,A) (additive Holt-Winters):** $\hat y_{t+h|t} = \ell_t + h b_t + s_{t+h-m(k+1)}$
- If a Box-Cox/log transform is used, point forecasts must be **bias-adjusted back to the mean** before aggregation, because medians do not add (FPP3 §5.6). This is mandatory whenever forecasts feed hierarchy sums or financials.
- Residual diagnostics (Ljung-Box, with $\ell=10$ for non-seasonal data or $2m$ for seasonal data, capped at $T/5$) are reported, but they are **not** used to select among models (FPP3 §5.4).

### GS-M-06 Intermittent-demand estimators (exact) [STD]
Let $z$ be the smoothed non-zero demand size, $p$ the smoothed inter-demand interval, and $q$ the periods since the last demand.

- **Croston (1972).** Update only when $y_t>0$:
  $z_t = \alpha y_t + (1-\alpha) z_{t-1}$, $\ p_t = \alpha q + (1-\alpha) p_{t-1}$. Forecast: $\hat y = z_t/p_t$. Croston is **positively biased** (Syntetos & Boylan 2001), so it is not the default.
- **SBA (Syntetos & Boylan 2005).** $\hat y = \left(1-\frac{\alpha}{2}\right)\frac{z_t}{p_t}$, where $\alpha$ is the interval smoothing constant. This is the **default** intermittent estimator.
- **TSB (Teunter, Syntetos & Babai 2011).** Let $\pi$ be the demand-occurrence probability, updated **every** period:
  - If $y_t>0$: $\pi_t=\pi_{t-1}+\beta(1-\pi_{t-1})$ and $z_t=z_{t-1}+\alpha(y_t-z_{t-1})$
  - If $y_t=0$: $\pi_t=\pi_{t-1}+\beta(0-\pi_{t-1})$ and $z_t=z_{t-1}$
  - Forecast: $\hat y=\pi_t z_t$
  
  TSB is unbiased and decays toward zero during long zero runs, so it is **required** for items with obsolescence/end-of-life risk and for dead-stock decisions.
- **Parameters:** $\alpha,\beta \in [0.05, 0.3]$ is a common practical range [RoT]. Croston originally used α = 0.1. Optimize on an out-of-sample error, not in-sample fit.
- **Intervals:** Croston-type methods have no underlying stochastic model, so they have no analytic prediction intervals (Shenstone & Hyndman 2005). Inventory skills must therefore derive lead-time demand distributions from GS-M-08 or from bootstrapping, not from the point forecast.

### GS-M-07 Temporal aggregation [STD]
- **ADIDA:** aggregate non-overlapping buckets (for example, weekly → 4-weekly), forecast at that level, then disaggregate equally or by profile.
- **MAPA (Kourentzes, Petropoulos & Trapero 2014):** fit ETS at multiple aggregation levels and combine the components. For intermittent series, use iMAPA, which selects Croston, SBA, or SES per level and combines by mean or median. Use these methods when the base-level signal is too sparse or the horizon is long.

### GS-M-08 Zero-inflated and count demand [STD/RoT]
- For small counts (roughly a max of 10 or less per period, or many zeros), never assume normal lead-time demand for safety stock [RoT threshold].
- **Distribution choice** on non-censored history over the lead time plus review period:
  - **Poisson** if the variance-to-mean ratio (VMR) is ≈ 1
  - **Negative binomial** if VMR > 1 (over-dispersion)
  - **Zero-inflated (ZIP/ZINB) or hurdle** models when the zero share exceeds what Poisson/NB implies
  - **Compound (e.g., Poisson-gamma / stuttering Poisson)** for lumpy order sizes
- Test the fit by comparing observed and expected zero shares and VMR. Report the chosen distribution.
- Distinguish **structural zeros** (not ranged, store closed, item not listed) from **sampling zeros** (listed, available, and no one bought). Structural zeros are removed from the training data. Sampling zeros are kept (GS-D-02).

### GS-M-09 Machine-learning (global) models
- **Allowed when:** there are many related series, causal features, and enough history. Gradient-boosted trees and global models led the M5 competition (Makridakis, Spiliotis & Assimakopoulos 2022) [EVID].
- **Mandatory controls:**
  - Time-ordered rolling-origin validation only. No random K-fold across time.
  - No look-ahead leakage. Features must be known at forecast time $t-\ell$. Future promo and price are allowed only if they are planned and locked.
  - Must beat NF1/NF2 **and** the best statistical model on the primary metric (FVA ≥ 0).
  - Report feature importance and bias by segment.
  - Use a loss function aligned with the decision: Tweedie or Poisson for counts or intermittent series, quantile (pinball) loss for service-level targets.

### GS-M-10 Probabilistic forecasts [STD]
- Point forecasts are insufficient for inventory decisions. Provide quantiles or intervals.
- **Pinball / quantile score (FPP3 §5.9):**
  $$Q_{p,t}= \begin{cases}2(1-p)(f_{p,t}-y_t) & y_t<f_{p,t}\\ 2p(y_t-f_{p,t}) & y_t\ge f_{p,t}\end{cases}$$
- **Winkler score** for intervals and **CRPS** for full distributions. Use skill scores relative to the naïve model for cross-series comparison.
- **Aggregate-over-horizon intervals** (for example, lead-time demand) must come from **simulated sample paths summed per path**, not from summed quantiles (FPP3 §13.5).

### GS-M-11 Stockout censoring and unconstrained-demand reconstruction (mandatory edge case)
Recorded sales equal $\min(D_t, \text{available}_t)$. Training on raw sales during stockouts **biases forecasts downward** and creates a self-reinforcing under-stocking loop.

1. **Detect censoring.** Flag period $t$ as censored if any of these holds:
   - On-hand at period start or end ≤ 0
   - In-stock rate (share of time or store-days in stock) is below 100%
   - Backorders or lost-order logs are greater than 0
   - Order cut or short-ship is greater than 0
   
   Default: treat a period as **materially censored** if in-stock rate is below 95% [RoT].
2. **Reconstruct.** Use the first applicable method:
   - **(a) Backorder/order data:** if the system records orders (B2B), unconstrained demand = orders received (requested quantity and date), not shipments.
   - **(b) Availability scaling:** $\hat D_t = S_t / a_t$, where $a_t$ is the in-stock fraction weighted by the intraperiod demand profile (for example, the intraday or day-of-week curve). Cap the uplift at ×2 or use method (c) when $a_t<0.5$ [RoT].
   - **(c) Model-based:** treat $S_t$ as right-censored and estimate by EM or a Tobit-type likelihood (for example, Tobit exponential smoothing). For substitutable items, use EM with choice models (Vulcano, van Ryzin & Ratliff 2012).
   - **(d) Last resort:** replace with the model's fitted or interpolated value and mark it as imputed.
3. **Document** the share of history reconstructed per SKU. If more than 30% of periods are reconstructed, flag the forecast as **low confidence** [RoT].
4. **Substitution:** when demand transfers to substitutes, do not double-count. Reconstructed demand for the out-of-stock item must be netted against substitute uplift where it is measurable (GS-D-06).
5. **KPI rule:** score accuracy against unconstrained actuals, and report constrained (sales) accuracy separately.

### GS-M-12 Safety stock: Cycle Service Level target [STD]
- **CSL = P(no stockout per replenishment cycle) = $\Phi(k)$**, so $k=\Phi^{-1}(\text{CSL})$.
- Standard deviation of lead-time demand (continuous review). Treat demand and lead time as independent:
  $$\sigma_{DL}=\sqrt{\bar L\,\sigma_d^2+\bar d^{\,2}\sigma_L^2},\qquad SS = k\,\sigma_{DL},\qquad ROP = \bar d\,\bar L + SS$$
- **Periodic review (R, S):** replace $L$ with $R+L$ (the protection interval).
- **Gold-standard requirement:** $\sigma_d$ must be the standard deviation of **forecast error** at the relevant lag and bucket (for example, RMSE of lag-L forecasts), **not** the standard deviation of raw demand (Silver, Pyke & Thomas). The scaling $\sigma_L=\sigma_1\sqrt{L}$ is valid only if errors are uncorrelated. Otherwise, measure errors directly at horizon L.
- Do not use normal-$k$ safety stock for intermittent or count items (use GS-M-08 distributions), or when $\sigma_{DL}/\mu_{DL} > 0.5$ (risk of negative demand mass) [RoT].

### GS-M-13 Safety stock: Fill-rate (β) target [STD]
- Unit normal loss function: $G(k)=\phi(k)-k\,[1-\Phi(k)]$
- **(s, Q) policy:** $\beta = 1-\dfrac{\sigma_{DL}\,G(k)}{Q}$. Solve $G(k) = (1-\beta)Q/\sigma_{DL}$ for $k$, then $SS=k\sigma_{DL}$.
- **(R, S) policy:** $\beta = 1-\dfrac{\sigma_{R+L}\,G(k)}{\bar d R}$
- When $k<0$ results (large Q), SS may be negative. Floor it at the policy minimum and report it. Never equate CSL and fill rate; the same $k$ yields different values of each.

### GS-M-14 Newsvendor / single-period [STD]
- Critical ratio $CR = \dfrac{C_u}{C_u + C_o}$, where $C_u$ is underage cost (lost margin plus goodwill) and $C_o$ is overage cost (cost − salvage + holding).
- Optimal quantity $Q^* = F^{-1}(CR)$ from the forecast distribution.
- Use for seasonal, perishable, or one-shot buys, and for setting quantile targets (GS-M-10).

### GS-M-15 Forecast combination [STD/EVID]
Use equal weights unless there is at least one year of rolling-origin evidence that the weights are stable. Combination intervals must come from joint sample paths (FPP3 §13.4).

### GS-M-16 Backtesting protocol [STD]
- **Rolling-origin (time-series cross-validation)** with expanding or sliding windows.
- The test horizon must be at least the maximum decision horizon. For single splits, the holdout is about 20% of the sample (FPP3 §5.8).
- Score by horizon/lag. Never average errors across different horizons without reporting each lag (FPP3 §13.8).
- Minimum: 3 forecast origins, or 1 full seasonal cycle of origins for seasonal items [RoT].

---

## 2. Canonical KPIs (GS-K)

Unless stated otherwise, sums run over the evaluation set of item-location-periods at the stated lag (GS-M-01).

| ID | KPI | Exact formula | Use when | Pitfalls / rules |
|---|---|---|---|---|
| **GS-K-01** | **WAPE** (= WMAPE, IBF) | $\displaystyle \text{WAPE}=\frac{\sum_t\lvert A_t-F_t\rvert}{\sum_t \lvert A_t\rvert}\times100$ | **Primary portfolio accuracy metric.** Volume-weighted; robust to zeros at the item level | Undefined if $\sum A=0$. Dominated by high-volume items, so pair it with a per-item MASE. A **value-weighted** variant uses $\lvert A-F\rvert\times$price. Accuracy is reported as $\max(0,100-\text{WAPE})$ only if explicitly labeled. |
| **GS-K-02** | **MAPE** | $\displaystyle \frac{100}{n}\sum_t \left\lvert\frac{A_t-F_t}{A_t}\right\rvert$ | Legacy/communication only, on strictly positive, non-intermittent series | Infinite when $A_t=0$, explosive near 0, and penalizes over-forecasts more heavily (FPP3 §5.8). **Never** use for intermittent items. Excluded zeros must be counted and disclosed. |
| **GS-K-03** | **sMAPE** (M3 form) | $\displaystyle \frac{100}{n}\sum_t \frac{2\lvert A_t-F_t\rvert}{\lvert A_t\rvert+\lvert F_t\rvert}$ | Only for comparability with external studies | Hyndman & Koehler (2006) recommend **against** it: it is unstable near zero, asymmetric, and bounded at 200%. Define 0/0 = 0 and disclose it. Reported, not decided on. |
| **GS-K-04** | **MASE** (and RMSSE) | $\displaystyle q_j=\frac{e_j}{\frac{1}{T-m}\sum_{t=m+1}^{T}\lvert y_t-y_{t-m}\rvert}$, $\ \text{MASE}=\text{mean}(\lvert q_j\rvert)$, with $m=1$ for non-seasonal series | **Primary per-series and cross-series scale-free metric**, valid for intermittent series | The scaling uses the **training** in-sample naïve MAE. MASE < 1 beats the in-sample one-step naïve. The scale is undefined if training data are constant, so fall back to WAPE. RMSSE is used for MSE-optimal (mean) forecasts. |
| **GS-K-05** | **Forecast Bias** | $\displaystyle \text{Bias\%}=\frac{\sum_t (F_t-A_t)}{\sum_t A_t}\times 100$ (positive = **over**-forecast) | Every report, at every level | Opposite sign to $e_t$, so always state the direction in words. Bias offsets at aggregate levels, so compute it at decision level and roll up the absolute values. Mean error (ME) = $\text{mean}(e_t)$. |
| **GS-K-06** | **Tracking Signal** (APICS) | $\displaystyle TS_t=\frac{\sum_{i=1}^{t} (A_i-F_i)}{\text{MAD}_t}$, where $\text{MAD}_t=\frac{1}{t}\sum\lvert A_i-F_i\rvert$ (or smoothed MAD) | Persistent-bias detection per item | Positive TS = under-forecasting. Out of control when $\lvert TS\rvert>4$ (common APICS control limit, [RoT]; range 3.75–6 used). Use a rolling window (for example, the last 6–13 periods) so old errors age out [RoT]. |
| **GS-K-07** | **Forecast Value Added (FVA)** | $\text{FVA}_{s} = \text{Err}(\text{prior step}) - \text{Err}(\text{step } s)$, using the same metric (WAPE by default), the same items, and the same lag. Stair-step: Naïve → Statistical → Analyst override → Consensus → Final | Judging every process step and touchpoint | Positive = value added. NF1/NF2 is the ultimate comparator (Gilliland). Requires storing **every version** of the forecast. Report FVA with a significance check: a paired test or bootstrap over items, or at least ≥ 6 cycles [RoT]. |
| **GS-K-08** | **Cycle Service Level (CSL)** | $\displaystyle \text{CSL}=\frac{\#\text{ replenishment cycles with no stockout}}{\#\text{ cycles}}$; planned CSL $=\Phi(k)$ | Setting/validating safety stock (GS-M-12) | Ignores the size of the shortage and is not a customer metric. Do not report it as "service level" without the "cycle" qualifier. |
| **GS-K-09** | **Fill Rate** | **Unit fill rate:** $\frac{\text{units shipped from stock on first pass}}{\text{units demanded}}$. Line and order fill rate are similar at line or order level. | Customer service from stock (β) | Always state the basis (units, lines, or orders) (ASCM). The denominator is **requested** (unconstrained) quantity. Backfilled shipments do not count as filled. |
| **GS-K-10** | **OTIF** | $\displaystyle \text{OTIF}=\frac{\#\text{ orders (or lines) delivered on time AND in full}}{\#\text{ orders (or lines)}}$ | Customer-facing delivery reliability | **On time** is measured against the agreed date: the customer-requested or committed date (state which) within a stated window, for example −1/+0 days [RoT]. **In full** means delivered quantity ≥ ordered quantity (within any stated tolerance). Both conditions must hold for the same order. Report measurement point (ship vs. receipt). Related: SCOR RL.1.1 Perfect Order Fulfillment (adds documentation and condition). |
| **GS-K-11** | **Days of Supply (DOS)** | **Forward-looking (gold standard):** the number of days until cumulative **forecast** demand exhausts on-hand (plus optionally on-order) inventory. **Simple:** $\text{DOS}=\frac{\text{On-hand units}}{\text{Avg daily forecast demand over next }N\text{ days}}$ | Inventory-position health, excess/shortage | State whether it is backward (history) or forward (forecast) looking, and whether it includes on-order or in-transit. Backward DOS hides seasonality. Use the same bucket calendar (GS-D-09). |
| **GS-K-12** | **Inventory Turns** | $\displaystyle \text{Turns}=\frac{\text{COGS over period (annualized)}}{\text{Average inventory at cost}}$; $\ \text{Days of inventory}=\frac{365}{\text{Turns}}$ | Financial inventory efficiency | Use **cost** in both numerator and denominator. Do not mix COGS with retail-valued inventory. Average inventory should use ≥ 2 points, preferably monthly or 13-point averages. |
| **GS-K-13** | **Reporting rules** | n/a | All KPIs | Report at the **decision level** (for example, SKU-DC-week at lag L) and at aggregate levels, separately. Use volume-weighted WAPE and the median of MASE across items. Report N items, N periods, % reconstructed actuals, and % excluded zeros. |
| **GS-K-14** | **Probabilistic KPIs** | Pinball loss, Winkler, CRPS (GS-M-10); **coverage** $=\frac{\#\,A_t\in \text{PI}}{n}$ | Quantile/interval forecasts | The empirical coverage of an X% interval should be ≈ X%. Flag it when it differs by more than 5 percentage points [RoT]. |

**Metric selection rule:** the primary metrics are WAPE (portfolio) + MASE (per series) + Bias% + FVA. MAPE and sMAPE are secondary or communication metrics only.

---

## 3. Data Hygiene & Granularity (GS-D)

### GS-D-01 Canonical grain and schema
- **Canonical key:** `item_id × location_id × period_start` (plus `channel` / `customer` where relevant), in base units of measure. Units of measure (UoM) are converted with an auditable conversion table.
- Standard grains: **SKU-Location-Day** for POS/sensing, **SKU-Location-Week** for operational forecasting, and **Product family-Region-Month** for S&OP.
- **Required columns for a demand history row:**
  - `item_id, location_id, period_start, period_grain, demand_qty, demand_type` (POS / orders / shipments)
  - `on_hand_qty, in_stock_pct, promo_flag, price, list_price, event_flags, lifecycle_status, data_quality_flag`

### GS-D-02 Zero vs. missing vs. censored (never conflate)
| State | Meaning | Treatment |
|---|---|---|
| True zero | Listed, available, no demand | Keep as 0 |
| Structural zero | Not ranged, store closed, pre-launch, discontinued | Remove or mask; do not train on it |
| Missing | No record (feed failure) | Impute (GS-D-03) and flag |
| Censored | Sales limited by availability | Reconstruct (GS-M-11) |

### GS-D-03 Missing values [STD/RoT]
- If missingness is informative (for example, holiday closures), model it with dummies rather than imputing it (FPP3 §13.9).
- If missingness is random: impute by model-based interpolation (for example, ARIMA/state-space interpolation) or seasonal-profile interpolation. Never use zero-fill unless the value is a true zero.
- If more than 20% of the history is missing, or there is a gap longer than one seasonal cycle, use only the history after the gap or flag the series [RoT].

### GS-D-04 Outliers [STD]
- **Detection:** robust STL decomposition. Flag remainder values outside $[Q_1-3\,\text{IQR},\ Q_3+3\,\text{IQR}]$, which is the FPP3 §13.9 preferred rule (1.5×IQR is the looser alternative).
- **Treatment:**
  - First **explain** the outlier: promo, event, data error, or bulk/one-off order.
  - Known recurring causes are modeled as regressors, not removed.
  - Data errors and confirmed non-recurring events are replaced via interpolation.
  - Every replacement is logged with its original value. Outliers must never be "cleaned" silently.
- For intermittent series, do not apply STL-based detection. Use size-only checks on non-zero demands, for example the P99 or a 3×IQR rule on non-zero sizes [RoT].

### GS-D-05 Promotional lift decomposition
- **Baseline** $B_t$: demand that would occur without the promotion. Estimate it by regression with promo and price terms, or by forecasting through the promo window with promo periods masked.
- **Gross lift:** $\text{Lift}_t = A_t - B_t$. **Lift %:** $(A_t-B_t)/B_t$.
- **Net incremental volume:**
  $$\text{NetInc} = \sum_{t\in\text{promo}}(A_t-B_t) \;-\; \underbrace{\sum_{t\in\text{post}}(B_t-A_t)}_{\text{post-promo dip / pantry loading}} \;-\; \underbrace{\sum_{t\in\text{pre}}(B_t-A_t)}_{\text{pre-promo dip}} \;-\; \text{Cannibalization} + \text{Halo}$$
- Forecasts for future promos = baseline × modeled lift, using the planned mechanics (depth, display, feature).

### GS-D-06 Cannibalization and halo
- **Cannibalization:** the loss in baseline of substitutable items $j$ (same need-state) during the promotion of item $i$: $\sum_j (B_{j,t}-A_{j,t})^+$.
- **Halo:** the gain in complementary items: $\sum_k (A_{k,t}-B_{k,t})^+$.
- Measure them with cross-price or cross-promo regression terms, or by control-group comparison (test vs. control stores).
- Report the **category-level net** effect. A promo with positive SKU lift but zero or negative category net is flagged.

### GS-D-07 Hierarchy reconciliation [STD]
All coherent methods take the form $\tilde y_h = S G \hat y_h$, where $S$ is the $n\times m$ summing matrix (FPP3 §11.3).

| Method | G / rule | Use when | Caveat |
|---|---|---|---|
| **Bottom-up** | Forecast at the bottom level and sum | Bottom series have strong signal | Noisy at sparse bottoms |
| **Top-down** | Disaggregate the total by proportions $p_j$: average historical proportions $p_j=\frac1T\sum_t y_{j,t}/y_t$, proportions of historical averages, or **forecast proportions** (default) | Very sparse bottoms, low counts | **Always biased** (fails $SGS=S$) |
| **Middle-out** | Forecast a middle level, sum it upward, and apply top-down below it | Strictly hierarchical structures with a reliable middle level | Same top-down bias below the middle |
| **MinT** (GS-D-08) | Optimal combination using all levels | **Default** for production | Needs a residual covariance estimate |

### GS-D-08 MinT (Wickramasuriya, Athanasopoulos & Hyndman 2019) [STD]
$$\tilde{\boldsymbol y}_h=\boldsymbol S(\boldsymbol S'\boldsymbol W_h^{-1}\boldsymbol S)^{-1}\boldsymbol S'\boldsymbol W_h^{-1}\hat{\boldsymbol y}_h$$
This minimizes $\operatorname{tr}(\boldsymbol V_h)$ subject to $SGS=S$. Choose $W_h$ in this priority order:
1. **`mint_shrink`** (shrinkage covariance; default when $m$ is large relative to $T$)
2. `wls_var` (diagonal residual variances)
3. `wls_struct` ($\Lambda=\text{diag}(S\mathbf 1)$; use when residuals are not available, for example for judgmental bases)
4. `ols`

Reconciled forecasts can be negative. Apply a non-negativity fix by clipping and re-reconciling, or by using a constrained solver, and log it.

### GS-D-09 Time-bucket alignment
- Keep a **single fiscal calendar table** (`date → fiscal_week, fiscal_period, fiscal_quarter, fiscal_year, days_in_period, trading_days`). All skills must join to it.
- **Upsampling** daily POS → weekly: sum days within the fiscal week (ISO or retail week-ending). Week-end day is explicit (for example, Saturday).
- **Weekly → calendar month:** apportion by day count. Weeks that span months are split by days in each month, optionally weighted by day-of-week profile, not assigned wholesale [RoT/STD practice].
- **Monthly S&OP → weekly execution:** disaggregate by the weekly profile from history or forecasts (proportions), then reconcile (temporal hierarchies, Athanasopoulos et al. 2017).
- Compare periods of unequal length on a **per-day or per-week rate**, never on raw totals.

### GS-D-10 4-4-5 and 53-week calendars
- In a 4-4-5 (or 4-5-4, 5-4-4) calendar, quarters are 13 weeks and a year is 52 weeks (364 days). A **53rd week** is inserted periodically, roughly every 5–6 years.
- Year-over-year and seasonal-naïve comparisons must align **like weeks** (for example, the retail-calendar "restated" year) and handle week 53 explicitly: drop it, merge it, or model it with a dummy. Moving holidays (Easter, Lunar New Year, Diwali, Eid) require event dummies (FPP3 §13.1).

### GS-D-11 Demand-signal hierarchy
- Prefer the signal closest to the consumer: POS / sell-out, then customer orders (requested date), then shipments / sell-in.
- Shipments are a **constrained** proxy and must not be used as demand when orders are available.
- Track the **sell-in vs. sell-out** gap to detect channel inventory build or burn and bullwhip. A common measure is variance amplification $= \operatorname{Var}(\text{orders})/\operatorname{Var}(\text{demand})$ (Lee, Padmanabhan & Whang 1997).

### GS-D-12 Lifecycle and master data
- New items use analogs or likeness-based curves: shape × scale, with a documented analog and expected range.
- Supersessions or replacements chain history from the old SKU to the new one with a conversion factor.
- Phase-outs switch to TSB or a run-out plan.
- Minimum-history gates for statistical models are set by the planning skill and documented.

### GS-D-13 Lineage
Every transformation (cleaning, reconstruction, imputation, outlier replacement, reconciliation) is logged with **before, after, rule ID, and timestamp**. Raw data are immutable.

---

## 4. Governance & Decision Guardrails (GS-G)

### GS-G-01 Override policy (human-in-the-loop) [EVID + RoT]
**Evidence base:** Fildes, Goodwin, Lawrence & Nikolopoulos (2009) studied more than 60,000 forecasts:
- About 75% of forecasts were adjusted.
- **Small adjustments** often damaged accuracy or wasted effort.
- **Larger adjustments** tended to improve accuracy.
- **Upward (positive) adjustments** were less likely to improve accuracy than downward ones and showed optimism bias.

**Mandatory rules:**
1. Every override requires a **reason code** (for example, promo, customer intel, supply event, new listing, or correction of a data error) and a free-text rationale with an owner and timestamp. Overrides without a reason are rejected.
2. **Materiality threshold [RoT, configurable].** Do not allow overrides whose absolute change is smaller than **max(10% of the statistical forecast, 1 × the forecast's MAD)** at the decision level, unless they correct a data error. These small changes are blocked or routed for justification.
3. **Upward overrides** larger than 20% [RoT] require secondary approval (demand manager) and documented evidence. They are scored separately.
4. Overrides are applied at a **declared level** and disaggregated per GS-D-07. They must never break coherence.

### GS-G-02 FVA-based retention [STD + RoT]
- Track FVA (GS-K-07) by **participant, reason code, direction, and size bucket** each cycle.
- If a touchpoint shows **negative FVA over a rolling 6+ cycles**, with a statistically significant result or a consistent sign, it is suspended for that segment, and the statistical forecast or the prior step is used instead [RoT].
- If the statistical forecast does not beat NF1/NF2 for a segment, the segment is routed to model review. Naïve may be used as the production forecast until it is fixed.

### GS-G-03 Exception management (management by exception)
Default triggers are [RoT, configurable]. Each exception lists cause, impact (units and value), and recommended action.

| Trigger | Default threshold |
|---|---|
| Tracking signal | $\lvert TS\rvert>4$ |
| Bias% at decision level | outside ±10% over the last 3 cycles |
| MASE | > 1.0 (worse than in-sample naïve) on the rolling window |
| FVA of the final forecast vs. naïve | < 0 |
| Forecast change vs. last cycle | > ±25% (within the frozen horizon) |
| Censored history share | > 30% |
| Forward DOS | < safety stock days (shortage) or > max days / shelf life (excess or obsolescence) |
| Prediction interval coverage | off nominal by > 5 percentage points |
| Unreconciled hierarchy delta | > 0.5% at any node |

**Prioritization:** rank exceptions by **value at risk** = |units at risk| × unit margin (or cost), multiplied by an ABC/XYZ weight. Show the top-N only. Do not flood planners.

### GS-G-04 Time fences and freeze
- **Frozen zone** (within lead time): changes only by exception approval.
- **Slushy zone:** changes allowed with reason codes.
- **Liquid zone:** open.
- Forecast versions are snapshotted at each lag for FVA and accuracy (GS-M-01).

### GS-G-05 Constrained vs. unconstrained plans
Keep separate and labeled:
- **Unconstrained demand forecast:** what customers want.
- **Constrained supply plan / sales plan:** what we will ship.
- **Financial plan.**

Accuracy is measured on unconstrained demand. Gaps between the plans are explicit S&OP decisions, not forecast adjustments (ASCM S&OP/IBP practice).

### GS-G-06 Volume-to-value financial reconciliation
- **Value forecast:** $V = \sum_i Q_i \times P_i^{\text{net}}$, where net price = list − trade spend / discounts, in the same currency and the same fiscal calendar (GS-D-09).
- **Reconcile** the consensus demand plan (volume × price) with the financial plan or budget every cycle at the agreed level (for example, brand × region × month). The **gap** = $V_{\text{demand}} - V_{\text{financial}}$ is reported, decomposed per GS-G-07, and resolved in the S&OP management business review. **The volume plan is never silently "plugged" to hit the financial number.**
- **Tolerance [RoT]:** an absolute gap greater than 2% of value at the reconciliation level triggers an escalation.

### GS-G-07 Price-Volume-Mix (PVM) decomposition [STD accounting practice]
For plan (0) vs. forecast or actual (1), with $Q$ = total volume and $s_i$ = mix share:
- **Volume effect** $= (Q_1 - Q_0)\times \bar P_0$, where $\bar P_0 = \sum_i s_{i,0}P_{i,0}$
- **Mix effect** $= Q_1 \times \sum_i (s_{i,1}-s_{i,0})P_{i,0}$
- **Price effect** $= \sum_i Q_{i,1}(P_{i,1}-P_{i,0})$
- Check: Volume + Mix + Price = $V_1 - V_0$ exactly. Verify the residual is 0 in code.

### GS-G-08 Allocation under shortage
When constrained supply is below demand, allocate by a declared rule:
- **Fair-share** (proportional to demand or forecast)
- **Priority tiers** (customer or segment SLA)
- **Margin-optimal** (an LP)

Log the rule and its outcome in OTIF and fill-rate terms. Allocation never changes the unconstrained forecast.

### GS-G-09 Segmentation for governance
Use **ABC** (value, Pareto) × **XYZ** (forecastability, using CV or MASE) × **demand class** (GS-M-03) to set:
- Review frequency
- Service-level targets (higher CSL for A/critical items)
- Override eligibility
- Model families

Cut-offs are [RoT]. A common default is A/B/C = 80/15/5% of value.

### GS-G-10 Standard output contract (all skills)
1. **Executive summary:** 3–6 bullets with the decision, value at risk, and confidence.
2. **KPI table:** the exact metrics from GS-K with lag, level, window, and N.
3. **Exceptions table:** ID, trigger (GS rule), item/location, units, value, recommended action, and owner.
4. **Recommendations:** ranked actions with expected impact.
5. **Assumptions and data-quality log:** % censored/imputed, outliers replaced, excluded zeros, and rules not satisfied.
6. **Reproducibility block:** code or SQL used, parameters, seed, and data snapshot ID.

### GS-G-11 Verification
- Any computed KPI must be reproducible in Python/SQL from the stated inputs.
- Hierarchy sums are checked: $\lvert\text{parent}-\sum\text{children}\rvert \le 10^{-6}$ relative.
- PVM sums are checked as exact.
- Safety stock is spot-checked against a closed-form or simulation estimate.
- When skills produce spreadsheets, formulas must be live, not hard-coded values.

---

## 5. Proposed 25-Skill Architecture

### Domain A: Demand Forecasting (7)
| # | Skill (kebab-case) | Purpose | Enforces |
|---|---|---|---|
| A1 | `nextq-demand-history-cleansing` | Build the canonical demand history: separate zero, missing, and censored values; handle UoM, outliers, and lineage | GS-D-01, D-02, D-03, D-04, D-12, D-13, M-11 (flag only), G-11 |
| A2 | `nextq-demand-pattern-segmentation` | ADI/CV² classification plus ABC/XYZ segmentation to route each series to a model family and policy | GS-M-03, M-04, M-08, G-09, K-04 |
| A3 | `nextq-statistical-baseline-forecasting` | ETS/ARIMA/STL/DHR baseline with naïve benchmarks, AICc and backtest selection, bias-adjusted back-transforms | GS-M-02, M-04, M-05, M-10, M-15, M-16, K-01, K-04, K-05 |
| A4 | `nextq-intermittent-demand-forecasting` | SBA (default), TSB for obsolescence risk, ADIDA/iMAPA, and count distributions for lumpy items | GS-M-03, M-06, M-07, M-08, M-16, K-04, K-14 |
| A5 | `nextq-ml-causal-demand-forecasting` | Global GBM/ML and dynamic-regression models with price, promo, weather, and event drivers under leakage-safe validation | GS-M-09, M-10, M-15, M-16, D-05, K-07, K-13 |
| A6 | `nextq-hierarchical-forecast-reconciliation` | Coherent forecasts with bottom-up, top-down, middle-out, or MinT (shrink), plus temporal hierarchies | GS-D-07, D-08, D-09, M-05 (bias adjustment), G-11 |
| A7 | `nextq-forecast-accuracy-diagnostics` | Rolling-origin scoring by lag with WAPE, MASE, Bias, TS, and probabilistic metrics, plus a metric-misuse guard | GS-M-01, M-16, K-01 to K-07, K-13, K-14, G-03 |

### Domain B: Demand Planning (6)
| # | Skill | Purpose | Enforces |
|---|---|---|---|
| B1 | `nextq-forecast-value-added-analysis` | Stair-step FVA by step, participant, reason code, and override size/direction, with significance testing | GS-K-07, K-05, M-02, G-01, G-02 |
| B2 | `nextq-consensus-demand-review` | Run the S&OP demand review: an unconstrained consensus plan, override intake with materiality and approvals, and time fences | GS-G-01, G-04, G-05, D-07, K-05, G-10 |
| B3 | `nextq-promotion-lift-cannibalization-planning` | Baseline vs. lift, pre/post dips, cannibalization and halo, net incremental volume, and promo forecast build | GS-D-05, D-06, M-09, K-07, G-06 |
| B4 | `nextq-new-product-lifecycle-forecasting` | Analog/likeness launch curves, supersession chaining, and phase-out/run-out (TSB) planning | GS-D-12, M-06, M-10, G-01, G-09 |
| B5 | `nextq-demand-financial-reconciliation` | Volume-to-value conversion, plan vs. budget gap, and PVM bridge at the S&OP level | GS-G-06, G-07, G-05, D-09, D-10, G-11 |
| B6 | `nextq-demand-exception-management` | Prioritized exception queue (TS, bias, MASE>1, FVA<0, swings) ranked by value at risk | GS-G-03, G-09, K-05, K-06, K-04, G-10 |

### Domain C: Demand Sensing (6)
| # | Skill | Purpose | Enforces |
|---|---|---|---|
| C1 | `nextq-short-horizon-demand-sensing` | Use daily POS and orders to re-forecast the 0–N week horizon (N set per lead time), blended with the baseline, with lag-specific FVA versus the baseline | GS-D-11, M-09, M-15, M-16, K-07, G-04 |
| C2 | `nextq-unconstrained-demand-reconstruction` | Detect stockout censoring and rebuild true demand by orders, availability scaling, or EM/Tobit, with substitution netting | GS-M-11, D-02, D-06, D-13, K-09, K-13 |
| C3 | `nextq-pos-sell-through-channel-analysis` | Sell-in vs. sell-out, channel inventory weeks of supply, and bullwhip (variance amplification) detection | GS-D-11, D-09, K-11, G-03 |
| C4 | `nextq-external-driver-signal-integration` | Add weather, events, price elasticity, and macro or search signals as leakage-safe features | GS-M-09, M-16, D-05, D-10, K-07 |
| C5 | `nextq-demand-anomaly-shift-detection` | Real-time detection of level shifts, spikes, and structural breaks (robust STL/IQR, TS), with triage of cause and planner alerts | GS-D-04, K-06, G-03, G-01 |
| C6 | `nextq-time-bucket-calendar-alignment` | Daily ↔ weekly ↔ fiscal month conversion, 4-4-5 / 53-week handling, and temporal disaggregation | GS-D-09, D-10, D-08, G-11 |

### Domain D: Demand-Supply Matching (6)
| # | Skill | Purpose | Enforces |
|---|---|---|---|
| D1 | `nextq-safety-stock-optimization` | Safety stock for CSL or fill-rate targets using forecast-error σ at lead time, lead-time variability, and non-normal/count cases | GS-M-08, M-12, M-13, M-10, K-08, K-09, G-09 |
| D2 | `nextq-replenishment-policy-design` | ROP/(s,Q)/(R,S)/(s,S) and newsvendor quantities, with policy simulation versus targets | GS-M-12, M-13, M-14, M-10, K-08, K-09, K-11 |
| D3 | `nextq-supply-demand-balancing` | Constrained vs. unconstrained gap analysis for S&OP supply review, with scenarios and decision options | GS-G-05, G-04, G-06, D-09, G-10 |
| D4 | `nextq-shortage-allocation-atp` | Allocate constrained supply (fair-share, priority, or margin LP) and available-to-promise, with OTIF and fill-rate impact | GS-G-08, K-09, K-10, G-05, G-11 |
| D5 | `nextq-service-level-performance-analysis` | Measure and root-cause CSL, fill rate, and OTIF (forecast vs. supply vs. execution attribution) | GS-K-08, K-09, K-10, K-13, M-11, G-03 |
| D6 | `nextq-inventory-health-optimization` | Forward DOS, turns, excess and obsolescence risk (TSB for dead stock), and rebalancing recommendations | GS-K-11, K-12, M-06, G-03, G-06, G-09 |

**Coverage check:** every GS principle is enforced by at least one skill, and every skill emits the GS-G-10 output contract.

---

## References

**Professional bodies and frameworks**
- ASCM (APICS), *ASCM Supply Chain Dictionary*, 18th ed. (definitions: fill rate, customer service ratio, OTIF, S&OP). https://learn.ascm.org/sfc/servlet.shepherd/document/download/069R3000006PlDHIA0
- ASCM, SCOR Digital Standard, RL.1.1 Perfect Order Fulfillment. https://scor.ascm.org/performance/reliability/RL.1.1
- CSCMP, *Supply Chain Management Definitions and Glossary of Terms*. https://cscmp.org/CSCMP/Educate/SCM_Definitions_and_Glossary_of_Terms.aspx
- Institute of Business Forecasting (IBF), Glossary: WMAPE. https://ibf.org/knowledge/glossary/weighted-mean-absolute-percentage-error-wmape-299 ; *Forecast Error Metrics*. https://ibf.org/knowledge/posts/forecast-error-metrics-to-assess-performance-39
- Gartner Demand-Driven Value Network (DDVN): definition as quoted (Burkett, 2012; Lord, 2012) in Kinaxis, "Are Demand-Driven Value Networks still a dream?" https://kinaxis.com/en/blog/are-demand-driven-value-network-still-a-dream ; Gartner Supply Chain Top 25 DDVN maturity model, SCMR. https://www.scmr.com/article/2015_supply_chain_top_25_the_art_and_science_of_supply_chain
- CPIM tracking signal (RSFE/MAD, ±4). https://trustedinstitute.com/concept/cpim/demand-planning/forecast-error-tracking/

**Forecasting statistics**
- Hyndman, R.J. & Athanasopoulos, G. *Forecasting: Principles and Practice*, 3rd ed. (OTexts):
  - Ch. 5 (benchmarks, accuracy, MASE, pinball, cross-validation). https://otexts.com/fpp3/accuracy.html
  - Ch. 11 (hierarchical reconciliation, MinT). https://otexts.com/fpp3/reconciliation.html
  - Ch. 13 (weekly/daily data, Croston, combinations, outliers, missing values). https://otexts.com/fpp3/missing-outliers.html
- Hyndman, R.J. & Koehler, A.B. (2006). Another look at measures of forecast accuracy. *IJF* 22(4), 679–688.
- Hyndman, R.J., Koehler, A.B., Snyder, R.D. & Grose, S. (2002). A state space framework for automatic forecasting using exponential smoothing methods. *IJF* 18(3). https://robjhyndman.com/papers/hksg.pdf
- Hyndman, R.J. & Khandakar, Y. (2008). Automatic time series forecasting: the forecast package for R. *JSS* 27(3). https://cloud.r-project.org/web/packages/forecast/vignettes/JSS2008.pdf
- Wickramasuriya, S.L., Athanasopoulos, G. & Hyndman, R.J. (2019). Optimal forecast reconciliation for hierarchical and grouped time series through trace minimization. *JASA* 114(526), 804–819.
- Athanasopoulos, G., Hyndman, R.J., Kourentzes, N. & Petropoulos, F. (2017). Forecasting with temporal hierarchies. *EJOR* 262(1), 60–74.
- Bates, J.M. & Granger, C.W.J. (1969). The combination of forecasts. *ORQ* 20(4); Wang, X., Hyndman, R.J., Li, F. & Kang, Y. (2023). Forecast combinations: an over 50-year review. *IJF* 39(4).
- Makridakis, S., Spiliotis, E. & Assimakopoulos, V. (2022). M5 accuracy competition: Results, findings, and conclusions. *IJF* 38(4).

**Intermittent demand**
- Croston, J.D. (1972). Forecasting and stock control for intermittent demands. *ORQ* 23(3), 289–303.
- Syntetos, A.A. & Boylan, J.E. (2001). On the bias of intermittent demand estimates. *IJPE* 71, 457–466.
- Syntetos, A.A. & Boylan, J.E. (2005). The accuracy of intermittent demand estimates. *IJF* 21(2), 303–314.
- Syntetos, A.A., Boylan, J.E. & Croston, J.D. (2005). On the categorization of demand patterns. *JORS* 56. https://www.tandfonline.com/doi/abs/10.1057/palgrave.jors.2601841
- Kostenko, A.V. & Hyndman, R.J. (2006). A note on the categorization of demand patterns. *JORS*. https://robjhyndman.com/papers/idcat.pdf
- Teunter, R.H., Syntetos, A.A. & Babai, M.Z. (2011). Intermittent demand: Linking forecasting to inventory obsolescence. *EJOR* 214(3), 606–615. https://doi.org/10.1016/j.ejor.2011.05.018
- Kourentzes, N., Petropoulos, F. & Trapero, J.R. (2014). Improving forecasting by estimating time series structural components across multiple frequencies (MAPA). *IJF* 30(2). https://kourentzes.com/forecasting/2014/04/19/improving-forecasting-by-estimating-time-series-structural-components-across-multiple-frequencies ; iMAPA in R `tsintermittent`. https://search.r-project.org/CRAN/refmans/tsintermittent/html/imapa.html
- Shenstone, L. & Hyndman, R.J. (2005). Stochastic models underlying Croston's method. *J. Forecasting* 24(6).
- Svetunkov, I., "Intermittent demand classifications: is that what you need?" (OpenForecast, 2024). https://openforecast.org/2024/07/16/intermittent-demand-classifications-is-that-what-you-need/

**Judgment, FVA, governance**
- Gilliland, M. *Forecast Value Added Analysis: Step-by-Step* (SAS white paper). https://www.sas.com/content/dam/SAS/en_us/doc/whitepaper1/forecast-value-added-analysis-106186.pdf ; Gilliland, M., "FVA Analysis: Why and How," ISF 2019. https://isf.forecasters.org/wp-content/uploads/gravity_forms/2-dd30f7ae09136fa695c552259bdb3f99/2019/07/Gilliland_Michael_ISF2019.pdf ; Gilliland, M. (2010). *The Business Forecasting Deal*. Wiley.
- Fildes, R., Goodwin, P., Lawrence, M. & Nikolopoulos, K. (2009). Effective forecasting and judgmental adjustments. *IJF* 25(1), 3–23. https://doi.org/10.1016/j.ijforecast.2008.11.010 ; practitioner summary, *Foresight* Issue 8. https://forecasters.org/wp-content/uploads/Good-and-Bad-Judgment-in-Forecasting_Issue8.pdf

**Inventory / OR**
- Silver, E.A., Pyke, D.F. & Thomas, D.J. (2017). *Inventory and Production Management in Supply Chains*, 4th ed. CRC Press (safety stock, G(k), fill-rate policies).
- MIT CTL.SC1x, *Inventory Models for Probabilistic Demand* (continuous- and periodic-review lecture notes). https://courses.edx.org/asset-v1:MITx+CTL.SC1x_1+2T2015+type@asset+block/w7l1_ContinuousReviewPolicies_ANNOTATED_v14.pdf
- *Operations research models and methods for safety stock determination: A review* (2020, open access). https://pmc.ncbi.nlm.nih.gov/articles/PMC7550265/
- Lee, H.L., Padmanabhan, V. & Whang, S. (1997). Information distortion in a supply chain: The bullwhip effect. *Management Science* 43(4).

**Censored demand**
- Vulcano, G., van Ryzin, G. & Ratliff, R. (2012). Estimating primary demand for substitutable products from sales transaction data. *Operations Research* 60(2). https://business.columbia.edu/sites/default/files-efs/imce-uploads/CPRM/2009-2-PrimaryDemandEM.pdf
- Tobit Exponential Smoothing for censored demand (arXiv 2407.17920). https://arxiv.org/abs/2407.17920
