# Black Week Demand Planning: Campaign-Aware Forecasting with Prophet

**Demand Forecasting · Part 2.** Part 1: [link to Part 1]

> **All data is synthetic.** The retailer, suppliers and products are fictional. No real company data is used.

## The business question

It is 12 October 2026. A mid-sized online home & kitchen retailer must commit stock for Black Week (23–30 November) before it knows what customers will buy. **How much will it really sell?**

Two things make this hard:

1. **Stockouts hide demand.** 17 of 40 products sold out during Black Week 2024, and 18 in 2025. On those days, sales show available stock, not demand.
2. **Few events to learn from.** Each product has seen only two Black Weeks.

## Approach

| Step | What | Why |
|---|---|---|
| 1 | **Stockout correction**: sold-out days set to missing | A sold-out day says demand was *at least* the sales, so it should not be treated as low demand |
| 2 | **Pooled Black Week profile** learned across products that did not sell out | Single products have too few clean event days; this borrows strength across the catalog |
| 3 | **Prophet with regressors**: Black Week profile + planned discount depth (`-log(1 - discount)`), Christmas holiday, multiplicative mode | Both regressors are known in advance from the campaign plan, so there is no leakage |
| 4 | **Backtest**: plan as of 12 Oct 2025, forecast Black Week 2025 | Tests the method on an event it has not seen |
| 5 | **Scenario ranges** from backtest errors (P10/P90 of actual ÷ forecast on products that did not sell out) | Prophet's own intervals ignore uncertainty in the event effect |

Four models are compared: last year + growth (common planner shortcut), Prophet baseline (seasonality only, as in Part 1), Prophet + regressors on raw sales, and Prophet + regressors with stockout correction.

## Results

**Backtest: Black Week 2025, total across 40 products**

| Model | Bias vs. true demand | WAPE vs. true demand |
|---|---|---|
| Last year + growth | −33% | 49% |
| Prophet baseline | −63% | 64% |
| Prophet + regressors (raw sales) | −42% | 48% |
| **Prophet + regressors (stockouts fixed)** | **−10%** | **26%** |

The largest improvement comes from correcting the censored history, not from adding regressors alone.

**Scenarios.** Prophet's P10–P90 interval covered only 22% of backtest outcomes. Backtest-based multipliers (low ×0.79, high ×1.74) covered 72%. The range is asymmetric, so the risk is mostly to the upside.

**Black Week 2026 forecast (all 40 products):** low 12.4k · expected 15.8k · high 27.4k units. Last year + growth would plan 12.7k.

![Backtest bias](output/results/chart1_backtest_bias.png)
![Weighted blanket backtest](output/results/chart2_hero_sku_backtest.png)
![2026 scenarios](output/results/chart3_2026_scenarios_top10.png)

## Limitations

- Synthetic data: price sensitivities and event effects are assumptions, not market benchmarks.
- Only two past Black Weeks, and the scenario multipliers come from a single backtest year.
- Comparison with *true* demand is only possible because the data is synthetic. With real data, validation is limited to products that never sold out (the script reports both).

## How to run

```bash
pip install -r requirements.txt
python generate_dataset.py   # creates output/data and output/hidden (fixed seed, reproducible)
python demand_planning.py    # backtest, 2026 forecast, scenarios and charts in output/results
python sanity_check.py       # optional: chart of observed sales vs. hidden true demand
```

## Repository structure

```
generate_dataset.py      synthetic data generator
demand_planning.py       censoring fix, models, backtest, scenarios, charts
sanity_check.py          censoring sanity chart
output/data/             data the planner can use
output/hidden/           ground truth, for evaluation only
output/results/          backtest, forecasts, scenarios, charts
```

## Data dictionary

**Setup:** 40 products in 4 categories (Small Appliances, Cookware, Tableware & Gifts, Home Textiles). Daily sales 1 Jan 2024 – 11 Oct 2026. Six suppliers: 3 EU (lead time 14–24 days) and 3 Asia (63–84 days). Black Friday 2026 is 27 Nov.

### `output/data/`: what the planner can use

| File | Grain | Columns |
|---|---|---|
| `sales_daily.csv` | date × product | units_sold, list_price_eur, discount_pct, selling_price_eur, campaign_id, stock_end_of_day, stockout_flag, revenue_eur |
| `products.csv` | product | product_name, category, supplier_id, list_price_eur, unit_cost_eur, moq_units, case_pack_units |
| `suppliers.csv` | supplier | region, country, lead_time_mean_days, lead_time_sd_days |
| `campaigns.csv` | campaign | campaign_name, campaign_type, start_date, end_date, status (completed / planned) |
| `campaign_discounts.csv` | campaign × product | discount_pct, including planned Black Week 2026 discounts |
| `inventory_snapshot.csv` | product | on_hand_units as of 12 Oct 2026 |
| `open_purchase_orders.csv` | order | sku, supplier_id, order_date, qty_units, confirmed_eta |
| `purchase_orders_history.csv` | order | received orders with actual_arrival and actual_lead_time_days |
| `cost_params.csv` | product | unit cost, holding_cost_rate_annual, clearance_discount_pct, stockout_goodwill_penalty_pct |

### `output/hidden/`: ground truth (never used for forecasting)

| File | Use |
|---|---|
| `actuals_2026.csv` | True daily demand 12 Oct – 31 Dec 2026, for a later forecast-vs-actual review |
| `true_demand_history.csv` | Unconstrained historical demand, to evaluate the stockout correction |
| `ground_truth_params.csv` | Demand archetype, price elasticity and event strength per year |
| `open_po_actual_arrivals.csv` | Actual arrival dates of open orders (some later than confirmed) |

### `output/results/`

| File | Content |
|---|---|
| `backtest_2025_black_week_by_sku.csv` | Black Week 2025 forecast per model vs. observed and true demand |
| `backtest_2025_summary.csv` | Bias and WAPE per model |
| `black_week_2026_scenarios.csv` | Low / expected / high per product, plus the comparison models |
| `forecast_2026_daily.csv` | Daily forecast to 31 Dec 2026 |
| `scenario_calibration.csv` | Scenario multipliers and interval coverage |
v")
