---
name: nextq-time-bucket-calendar-alignment
description: >-
  [Next Quarter Supply Chain Suite] Builds and enforces the single fiscal calendar every planning skill joins to:
  generates 4-4-5, 4-5-4 or 5-4-4 retail calendars with a configurable year-end rule and explicit 53-week years,
  verifies contiguity and week-ending days, converts between daily, weekly, fiscal-period and calendar-month buckets
  with total-conserving day-share apportionment (optionally day-of-week weighted), disaggregates monthly plans to
  weeks, maps like weeks for year-over-year comparisons, and reconciles temporal hierarchies. Load when the user asks
  to set up a fiscal or retail calendar, convert weeks to months or months to weeks, handle a 53rd week, align
  retailer and fiscal weeks, compare years like-for-like, or reconcile weekly and monthly forecasts. Inputs: calendar
  rules, bucketed quantities, daily profiles, events. Triggers: "fiscal calendar", "4-4-5", "53-week year", "week to
  month".
metadata:
  author: "Next Quarter"
  organization: "Next Quarter (nextq)"
  suite: "Next Quarter Supply Chain Intelligence"
  version: "1.0.0"
---

# Time-Bucket & Calendar Alignment
*Part of the **Next Quarter (nextq)** Supply Chain Skill Suite — Governed by SUPPLY_CHAIN_GOLD_STANDARD.md.*

## 1. Role & Objective
**Persona:** Planning Data Architect (calendar and bucket owner).
**Decision supported:** Are all plans, actuals and KPIs on the same calendar and comparable buckets? How do we move quantities between grains without creating or losing units?

**Objective:**
- One verified `fiscal_calendar` table.
- Deterministic, total-conserving conversions between grains.
- An explicit 53-week and moving-holiday policy.
- Coherent temporal reconciliation of forecasts made at different grains.

Every other nextq skill that changes buckets calls this skill's rules (e.g. `nextq-demand-financial-reconciliation`, `nextq-pos-sell-through-channel-analysis`, `nextq-short-horizon-demand-sensing`).

## 2. Gold Standard Rules Enforced
| Rule | Inlined standard |
|---|---|
| **GS-D-09** Time-bucket alignment | **Single fiscal calendar table** (`date → fiscal_week, fiscal_period, fiscal_quarter, fiscal_year, days_in_period, trading_days`); all skills join to it. **Daily → weekly:** sum the days within the fiscal week; the week-ending day is explicit (e.g. Saturday). **Weekly → calendar month:** apportion by day count; weeks spanning months are split by days in each month, optionally weighted by a day-of-week profile, never assigned wholesale. **Monthly S&OP → weekly:** disaggregate by a weekly or daily profile (proportions), then reconcile (temporal hierarchies, Athanasopoulos et al. 2017). Compare unequal periods on **per-day or per-week rates**, never raw totals. |
| **GS-D-10** 4-4-5 & 53-week | In 4-4-5 / 4-5-4 / 5-4-4 calendars, quarters are 13 weeks and a year is 52 weeks (364 days). A **53rd week** is inserted periodically (roughly every 5–6 years). Year-over-year and seasonal-naïve comparisons must align **like weeks** (restated year) and handle week 53 explicitly: drop it, merge it, or model it with a dummy. Moving holidays (Easter, Lunar New Year, Diwali, Eid) need event dummies. |
| **GS-D-08** Reconciliation | Coherent forecasts $\tilde y=SG\hat y$. Use WLS with structural scaling $W=\operatorname{diag}(S\mathbf 1)$ when residual covariances are unavailable. Unbiasedness requires $S G S=S$ (asserted). Fuller MinT variants are in `nextq-hierarchical-forecast-reconciliation`. |
| **GS-G-11** Verification | Every conversion conserves totals to $10^{-9}$ relative. Hierarchy sums satisfy $\lvert\text{parent}-\sum\text{children}\rvert\le10^{-6}$ relative. The calendar is contiguous, with no overlaps, every week 7 days, and one week-ending weekday. |
| **GS-G-10 / GS-D-13** | Standard output; every re-bucketing is logged with its method and profile version. |

**Calendar conventions [practice, configurable]:**
- **Year-end rule:** fiscal year ends on the **weekday nearest** an anchor date (default Saturday nearest 31 January, a common US retail convention) or the **last** such weekday on or before it.
- **Naming:** fiscal year = the calendar year in which it starts.
- **53rd week:** appended to the last period by default (period 12 becomes 5 or 6 weeks), and Q4 becomes 14 weeks.
- **Like-for-like:** if the prior year has 53 weeks, `SHIFT` (default) compares current week $w$ with prior week $w+1$ (a restated prior year that excludes its first week); `DROP` compares $w$ with $w$ and ignores week 53. A current-year week 53 has no comparator and is reported separately or modeled with a dummy.
- Weeks nest into fiscal periods, but **not** into calendar months. Temporal hierarchies therefore use fiscal periods; calendar months are reached only by day-share apportionment.

## 3. Required Inputs & Data Schema
| Table | Columns | Notes |
|---|---|---|
| `calendar_rules` | `anchor_month, anchor_day, weekday (Mon=0..Sun=6), rule ∈ {nearest, last}, pattern ∈ {445, 454, 544}, week53_period` | one row per company or region |
| `fiscal_calendar` (output) | `date, fiscal_year, fiscal_week, fiscal_period, fiscal_quarter, week_ending, is_week53, days_in_period, trading_days, holiday_flags` | single source of truth |
| `bucketed_input` | `key columns…, fiscal_year, fiscal_week` (or `date`, or `month`), `qty` | quantities to convert |
| `daily_profile` | `key, date, weight` (DOW × seasonality, mean 1) | for weighted apportionment and disaggregation |
| `event_calendar` | `event_id, date, scope` | moving holidays per year |
| `trading_calendar` | `location_id, date, open_flag` | closures → trading days |

## 4. Execution Workflow
1. **Generate** the calendar for the required years from `calendar_rules`.
   - Assert each year has 364 or 371 days (52 or 53 weeks).
   - The period pattern must sum to 52, plus the week 53 placement.
2. **Verify** (GS-G-11): contiguous, unique dates, 7-day weeks, a single week-ending weekday, and quarters of 13 weeks (14 for a 53-week Q4). Join holidays and trading days, and compute `days_in_period` and `trading_days`.
3. **Classify input grains.** A run must declare its source and target grain. Mixed grains in one input → stop (`MIXED_GRAIN`).
4. **Convert:**

   | From → to | Method |
   |---|---|
   | Daily → fiscal week / period / quarter | Sum by calendar keys (exact) |
   | Weekly → fiscal period | Exact mapping (weeks nest) |
   | Weekly → calendar month | Split each week's quantity across its days by DOW weights (equal if none), then sum by month |
   | Monthly → weekly | Spread the month total over its days by the daily profile, then sum by fiscal week |
   | Retailer week (other week-end day) → fiscal week | Spread to days (DOW-weighted), then re-sum |

   Every conversion asserts conservation.
5. **Rates for comparison:** per-day = qty / days (or trading days), and per-week = 7 × per-day. Use rates whenever comparing periods of unequal length (4- vs. 5-week periods, 53-week years).
6. **Year-over-year:** build the like-week map with the declared policy. Moving holidays are compared on event-relative weeks (event week −1, 0, +1) and flagged when they change periods across years.
7. **Temporal reconciliation:**
   - When forecasts exist at several nested fiscal grains (weekly, period, quarter), build $S$ for the quarter pattern and reconcile with WLS-structural.
   - Assert coherence and $S G S=S$.
   - Calendar-month views are derived **after** reconciliation by day-share apportionment.
8. **Log** the method, profile version and conservation residuals. **Output** (Section 6).

## 5. Verification Snippets
```python
import numpy as np, pandas as pd

def fiscal_year_end(year, anchor_month=1, anchor_day=31, weekday=5, rule="nearest"):
    """FY end = the `weekday` (Mon=0 .. Sat=5) nearest to (or last on/before) the anchor date in calendar `year`.
    Default = Saturday nearest 31 Jan (common US retail 4-5-4 convention)."""
    a = pd.Timestamp(year=year, month=anchor_month, day=anchor_day)
    back = (a.weekday() - weekday) % 7                      # days back to previous `weekday`
    if rule == "last":
        return a - pd.Timedelta(days=back)
    return a - pd.Timedelta(days=back) if back <= 3 else a + pd.Timedelta(days=7 - back)

def build_calendar(fy, pattern=(4, 5, 4), week53_period=12, **kw):
    """Daily calendar for fiscal year `fy` (ending in calendar year fy+1 by default anchor). 52 or 53 weeks;
    the 53rd week is appended to `week53_period` (default last period). Quarters = 13 weeks (14 in a 53-week Q4)."""
    start = fiscal_year_end(fy, **kw) + pd.Timedelta(days=1)
    end = fiscal_year_end(fy + 1, **kw)
    days = pd.date_range(start, end, freq="D"); n_weeks = len(days) // 7
    assert len(days) % 7 == 0 and n_weeks in (52, 53), (fy, len(days))
    weeks_per_period = list(pattern) * 4
    if n_weeks == 53: weeks_per_period[week53_period - 1] += 1
    period_of_week = np.repeat(np.arange(1, 13), weeks_per_period)
    wk = np.repeat(np.arange(1, n_weeks + 1), 7)
    cal = pd.DataFrame({"date": days, "fiscal_year": fy, "fiscal_week": wk,
                        "fiscal_period": period_of_week[wk - 1], "week_ending": days[6::7].repeat(7)})
    cal["fiscal_quarter"] = (cal.fiscal_period - 1) // 3 + 1
    cal["is_week53"] = cal.fiscal_week.eq(53)
    return cal

def verify_calendar(cals):
    """GS-D-09 single calendar: contiguous, no overlaps, each date once, 7 days per week, week ends on one weekday."""
    c = pd.concat(cals, ignore_index=True).sort_values("date")
    assert c.date.is_unique, "OVERLAP"
    assert (c.date.diff().dropna() == pd.Timedelta(days=1)).all(), "GAP"
    g = c.groupby(["fiscal_year", "fiscal_week"]).date
    assert (g.count() == 7).all(), "PARTIAL_WEEK"
    assert c.week_ending.dt.weekday.nunique() == 1, "MIXED_WEEK_END"
    return True

def week_to_calendar_month(weekly, cal, dow_weights=None):
    """Weekly qty -> calendar months by day share (optionally DOW-weighted, weights index Mon=0..Sun=6). Conserves totals."""
    w = np.ones(7) if dow_weights is None else np.asarray(dow_weights, float)
    d = cal.merge(weekly, on=["fiscal_year", "fiscal_week"])
    d["wt"] = w[d.date.dt.weekday]
    d["share"] = d.wt / d.groupby(["fiscal_year", "fiscal_week"]).wt.transform("sum")
    d["qty_day"] = d.qty * d.share
    out = d.groupby(d.date.dt.to_period("M")).qty_day.sum()
    assert abs(out.sum() - weekly.qty.sum()) <= 1e-9 * max(abs(weekly.qty.sum()), 1)
    return out

def month_to_weeks(month_qty, cal_days, daily_profile):
    """Disaggregate a calendar-month total to fiscal weeks by a daily profile (e.g. DOW x seasonality); conserves total."""
    p = np.asarray(daily_profile, float); assert len(p) == len(cal_days) and p.sum() > 0
    d = cal_days.assign(qty=month_qty * p / p.sum())
    return d.groupby(["fiscal_year", "fiscal_week"]).qty.sum()

def like_week_map(n_weeks_cur, n_weeks_prev, policy="SHIFT"):
    """GS-D-10 year-over-year like-week mapping. Equal lengths: w -> w. Prior year has 53 weeks:
    SHIFT maps w -> w+1 (restated prior year, its week 1 excluded); DROP maps w -> w and ignores week 53.
    Current year has 53 weeks: week 53 has no comparator (DUMMY / reported separately)."""
    m = {}
    for w in range(1, n_weeks_cur + 1):
        if n_weeks_prev == 53 and policy == "SHIFT": m[w] = w + 1
        elif w <= n_weeks_prev: m[w] = w
        else: m[w] = None
    return m

def temporal_S(pattern):
    """Summing matrix weeks -> periods -> quarter for one quarter (e.g. 4-5-4): rows = [quarter, periods..., weeks...]."""
    nw = sum(pattern); rows = [np.ones(nw)]; start = 0
    for k in pattern:
        r = np.zeros(nw); r[start:start + k] = 1; rows.append(r); start += k
    return np.vstack(rows + [np.eye(nw)])

def reconcile_wls_struct(S, yhat):
    """Temporal reconciliation (Athanasopoulos et al. 2017) with structural scaling W = diag(S 1): coherent, unbiased."""
    W_inv = np.diag(1 / S.sum(axis=1))
    G = np.linalg.solve(S.T @ W_inv @ S, S.T @ W_inv)
    y = S @ G @ yhat
    assert np.allclose(S @ G @ S, S)                       # unbiasedness (SGS = S)
    return y
```
**Test vector** (executed). Defaults: Saturday nearest 31 January, 4-5-4 pattern.
- **Fiscal year ends:** FY2022 → 28 Jan 2023, FY2023 → 3 Feb 2024, FY2024 → 1 Feb 2025.
  - **FY2023 has 53 weeks** (29 Jan 2023 – 3 Feb 2024), consistent with the 53-week US retail fiscal 2023. FY2024 has 52 weeks.
  - FY2023 weeks per period = [4,5,4, 4,5,4, 4,5,4, 4,5,**5**].
- **Verification:** FY2022–FY2024 pass the contiguity and week checks.
- **Weekly → calendar month:** FY2024 week 4 (Sun 25 Feb – Sat 2 Mar 2024), 70 units.
  - Equal days: Feb **50** / Mar **20**.
  - With weekend weight 2: Feb **46.667** / Mar **23.333**.
  - Totals conserved. This is the same day-share rule used by `nextq-demand-financial-reconciliation`.
- **Monthly → weekly:** March 2024, 310 units, flat daily profile → weeks 4–9 = 20 / 70 / 70 / 70 / 70 / 10 (sum 310).
- **Like weeks:** prior year with 53 weeks, `SHIFT` → week 1 ↔ prior week 2, week 52 ↔ prior week 53. Current week 53 under `DROP` → no comparator.
- **Temporal reconciliation** (4-5-4 quarter): base quarter 1,400, periods 420 / 560 / 430, 13 weeks × 100 (sum 1,300).
  - WLS-structural gives quarter **1,370**, periods 414.6 / 535.8 / 419.6, weeks summing to 1,370.
  - Coherent, and $S G S=S$ holds.

```sql
-- Calendar integrity checks (each query must return zero rows)
SELECT date FROM fiscal_calendar GROUP BY date HAVING COUNT(*) > 1;                                   -- overlaps
SELECT fiscal_year, fiscal_week FROM fiscal_calendar GROUP BY 1,2 HAVING COUNT(*) <> 7;               -- partial weeks
SELECT d.date FROM generate_series((SELECT MIN(date) FROM fiscal_calendar),
                                   (SELECT MAX(date) FROM fiscal_calendar), INTERVAL '1 day') AS d(date)
LEFT JOIN fiscal_calendar c ON c.date = d.date WHERE c.date IS NULL;                                  -- gaps
SELECT fiscal_year FROM fiscal_calendar GROUP BY 1 HAVING COUNT(*) NOT IN (364, 371);                 -- year length
SELECT DISTINCT EXTRACT(DOW FROM week_ending) FROM fiscal_calendar
HAVING COUNT(DISTINCT EXTRACT(DOW FROM week_ending)) > 1;                                             -- mixed week-end
```

## 6. Standardized Output Format
```markdown
# Calendar & Bucket Alignment — <scope> — run CAL_<run_id>

## Executive Summary
- Calendar <rule> generated for FY<a>–FY<b>: <n53> 53-week year(s) (FY<x>, week 53 in period <p>); integrity checks PASS/FAIL.
- <N> conversions run (<types>); max conservation residual <r> (≤1e-9 ✔).
- YoY policy <SHIFT/DROP>; moving holidays shifting periods: <list>.
- Temporal reconciliation: <n> quarters reconciled, max adjustment <x>% at <level>.

## Metadata (mandatory)
| Level | Source grain → target grain | Time window | Calendar rule | Week-ending day | 53-week policy | Apportionment weights | Weighting | Calendar version |
|---|---|---|---|---|---|---|---|---|
| <keys> | WEEK → CAL MONTH (etc.) | <start–end> | Sat nearest 31-Jan, 4-5-4 | Saturday | append to P12; YoY SHIFT | DOW <version> / equal | units | <id> |

## KPI Table
| Check | Result | Rule |
|---|---|---|
| Contiguity / overlaps / partial weeks | | GS-D-09 |
| Year lengths (364/371) | | GS-D-10 |
| Conservation residual (max) | | GS-G-11 |
| Temporal coherence | | GS-D-08 |

## Fiscal Year Summary
| FY | start | end | weeks | 53rd-week period | Q4 weeks | holidays moved vs LY |
|---|---|---|---|---|---|---|

## Conversion Log
| run | source grain | target grain | method | weights version | total in | total out | residual |
|---|---|---|---|---|---|---|---|

## Exceptions
| ID | Trigger (rule) | object | metric | value | action | owner |
|---|---|---|---|---|---|---|
| | Calendar integrity failure (GS-D-09) | | | | fix calendar table | Data architect |
| | MIXED_GRAIN input | | | | split run | Data owner |
| | Week 53 unhandled in YoY (GS-D-10) | | | | set policy | FP&A / planning |
| | Moving holiday crossing period (GS-D-10) | | | | add dummy / restate | Demand planner |

## Recommendations
1. …

## Data-Quality Log
| object | issue (missing profile → equal weights, retailer week-end mismatch, closed days) | treatment |
|---|---|---|

## Reproducibility
Calendar rules, profile versions, policies, code §5.
```

## 7. Edge Cases
- **Different calendars per region or entity** (e.g. a statutory calendar-month entity alongside retail 4-5-4): keep one calendar table per `calendar_id`. Cross-entity roll-ups go via daily grain, never by week labels.
- **ISO weeks vs. retail weeks:** ISO weeks start Monday, and ISO year 53s fall in different years than the retail 53-week years. Map through dates only.
- **Lead times in days with weekly buckets:** convert with $L_{\text{weeks}}=L_{\text{days}}/7$ (fractional) and keep the fraction (consumed by `nextq-safety-stock-optimization`). Do not round to whole weeks.
- **Store closures and holidays:** per-day rates use trading days for store-level comparisons and calendar days for e-commerce.
- **Partial current period:** compare month-to-date on rates or matched days only. A partial period is never compared to a full one.
- **Changing the fiscal year-end:** create a new `calendar_id` with an explicit transition year (possibly not 52 or 53 weeks), excluded from YoY.
- **Monthly → weekly without a profile:** use equal daily weights, flagged `FLAT_PROFILE`. Never use equal weekly shares, which would ignore 4- vs. 5-week periods and part-month weeks.
