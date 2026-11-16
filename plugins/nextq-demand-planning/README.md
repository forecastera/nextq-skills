# Next Quarter Demand Planning & Forecasting

25 skills that turn Claude into a rigorous demand planning analyst for supply chain, S&OP / IBP, inventory and FP&A teams. Every skill follows the **Next Quarter Supply Chain Gold Standard** ([SUPPLY_CHAIN_GOLD_STANDARD.md](SUPPLY_CHAIN_GOLD_STANDARD.md)). That means the formulas are sourced, forecast errors are measured at a stated lag, results are benchmarked against a naïve forecast instead of generic industry numbers, and every output reports any rule it could not satisfy.

Claude loads a skill automatically when your request matches it. For example: "set safety stock for these SKUs at a 98% fill rate", "why did forecast accuracy drop last month?", or "reconcile the consensus demand plan to the budget".

## Install

In Claude Code:
```
/plugin install nextq-demand-planning --marketplace forecastera/nextq-skills
```
On claude.ai (Team / Enterprise), your organization Owner adds it under **Organization settings > Plugins & skills**.

## Skills

**Data foundation**
| Skill | What it does |
|---|---|
| `nextq-demand-history-cleansing` | Builds the canonical, auditable demand history every other skill consumes |
| `nextq-unconstrained-demand-reconstruction` | Rebuilds true demand where sales were truncated by stockouts, allocations or short-ships |
| `nextq-time-bucket-calendar-alignment` | Builds and enforces one fiscal calendar (4-4-5, 4-5-4, 5-4-4) for all planning |
| `nextq-demand-pattern-segmentation` | Classifies series as smooth / erratic / intermittent / lumpy, plus ABC/XYZ and lifecycle |
| `nextq-demand-anomaly-shift-detection` | Detects and triages spikes, dips and level shifts as data arrives |

**Forecasting**
| Skill | What it does |
|---|---|
| `nextq-statistical-baseline-forecasting` | Statistical baseline with prediction intervals (ETS, ARIMA, naïve benchmarks) |
| `nextq-intermittent-demand-forecasting` | Slow-moving, lumpy and spare-parts demand (SBA, TSB, Croston) |
| `nextq-ml-causal-demand-forecasting` | Leakage-safe ML / causal forecasts (gradient boosting, dynamic regression) |
| `nextq-new-product-lifecycle-forecasting` | New launches via analogs, plus phase-in / phase-out |
| `nextq-promotion-lift-cannibalization-planning` | Promo lift, pre/post dips, cannibalization and halo |
| `nextq-external-driver-signal-integration` | Screens and encodes weather, holiday, price and macro drivers |
| `nextq-hierarchical-forecast-reconciliation` | Coherent forecasts across product, location, channel and time hierarchies |
| `nextq-short-horizon-demand-sensing` | Re-forecasts the next few weeks from POS, open orders and fresh signals |
| `nextq-pos-sell-through-channel-analysis` | Sell-in vs sell-out, channel inventory and weeks of supply |

**Performance & governance**
| Skill | What it does |
|---|---|
| `nextq-forecast-accuracy-diagnostics` | Lag-aware WAPE, MASE, bias, tracking signal and interval coverage |
| `nextq-forecast-value-added-analysis` | FVA stair-step: which process steps improve or worsen the forecast |
| `nextq-demand-exception-management` | Management-by-exception triggers and planner worklists |
| `nextq-consensus-demand-review` | Runs the monthly S&OP / IBP demand review and governs overrides |
| `nextq-demand-financial-reconciliation` | Reconciles the consensus demand plan with the financial plan or budget |

**Inventory & supply matching**
| Skill | What it does |
|---|---|
| `nextq-safety-stock-optimization` | Safety stock and reorder points for service-level or fill-rate targets |
| `nextq-replenishment-policy-design` | (s,Q), (R,S), (s,S) policies, EOQ, pack and MOQ constraints |
| `nextq-inventory-health-optimization` | Days of supply, turns, excess and obsolescence per SKU-location |
| `nextq-service-level-performance-analysis` | Fill rate, OTIF and cycle service level with root causes |
| `nextq-supply-demand-balancing` | S&OP supply review: rough-cut capacity and projected inventory |
| `nextq-shortage-allocation-atp` | Fair-share, priority and margin-optimal allocation, plus available-to-promise |

## What this plugin runs and sends

This plugin contains **instructions only**: Markdown skill files and one reference document. It ships no scripts, hooks, MCP servers or binaries. It does not contact any external service and does not send your data anywhere. When a skill needs a calculation, Claude runs it in its own code-execution environment on the data you provide in the conversation.

## Important

The skills support planning decisions; they do not make them. Review outputs, especially inventory targets, allocations and financial reconciliations, before you act on them. Formulas marked **[RoT]** are configurable rules of thumb, not industry benchmarks, so calibrate them to your business.

## Support

Next Quarter: https://nextq.ai
