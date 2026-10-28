# Black Week Demand Forecast for Consumer Electronics

**Forecasting Black Week sales from historical data with Prophet.**
Demand Forecasting · Part 2. Part 1: [link to Part 1] 

> **All data is synthetic.** The retailer, suppliers and products are fictional. Product names are generic, with no real brands or models.

## The business question

It is 12 October 2026. A fictional online consumer electronics retailer wants to know: **based on past sales, how much will each product sell during Black Week (23–30 November)?**

**Scope:** 22 established products that were on sale during Black Week 2024 and 2025 and are still on sale in 2026. Products launched more recently don't have enough history for this approach.

## Approach

The only input is daily sales history. Three forecasts are compared:

| Model | How it works |
|---|---|
| **Last year's Black Week** | Black Week 2025 sales, repeated for 2026 |
| **Prophet, seasonality only** | Trend, weekly and yearly seasonality learned from the history |
| **Prophet + Black Week event** | Same, plus Black Week (Monday before Black Friday to Cyber Monday) as a recurring calendar event, so the model learns its day-by-day effect from past years |

**Backtest:** each model forecasts Black Week 2025 using only sales up to 12 October 2025. The forecasts are then compared with what was actually sold.

## Results

**Backtest: Black Week 2025, 22 products**

| Model | Error (WAPE) | Bias | Total forecast vs. actual |
|---|---|---|---|
| Last year's Black Week | 14% | +7% | 3,880 vs. 3,615 |
| Prophet, seasonality only | 37% | −37% | 2,294 vs. 3,615 |
| Prophet + Black Week event | 19% | −6% | 3,398 vs. 3,615 |

**What this shows**

- **The model needs to know when Black Week happens.** Without the event, Prophet treats it as a normal late-November week and misses more than a third of the volume.
- **With the event, Prophet gets close but doesn't beat the simple benchmark.** For stable, established products, last year's Black Week is a strong baseline. Prophet has the smaller total bias (−6% vs. +7%), but last year's figure is more accurate product by product.
- **Always test against a simple benchmark.** A model only earns its place if it beats "same as last year."

**Black Week 2026 forecast (22 products):** 3,179 units with Prophet + Black Week event (80% interval 2,551–3,834), compared with 3,615 units sold in Black Week 2025. Per-product figures are in `black_week_2026_forecast.csv`.

![Backtest error](output/results_simple/chart1_backtest_error.png)
![Product example](output/results_simple/chart2_product_example.png)
![2026 forecast](output/results_simple/chart3_2026_forecast_top10.png)

## Limitations

- **Stockouts.** Some products sold out during past Black Weeks, so sales understate what customers wanted. Both methods inherit that, which is one reason Prophet forecasts below last year.
- **Only two past Black Weeks** to learn the event from, and a single backtest year.
- **No new products.** Products without two years of history are out of scope.
- **Synthetic data.** Demand patterns are assumptions, not market benchmarks.

## How to run

```bash
pip install -r requirements.txt
python generate_dataset.py    # creates the synthetic data in output/data (fixed seed, reproducible)
python forecast_simple.py     # backtest, 2026 forecast and charts in output/results_simple
```

## Repository structure

```
generate_dataset.py       synthetic data generator
forecast_simple.py        backtest, 2026 forecast, charts
output/data/              sales history and product data
output/results_simple/    backtest, forecasts, charts
```

## Data

The forecast uses two files:

| File | Grain | Columns used |
|---|---|---|
| `output/data/sales_daily.csv` | date × product | date, sku, units_sold (plus price, discount, stock and stockout columns for reference) |
| `output/data/products.csv` | product | sku, product_name, category, launch_date, end_of_life_date |

The generator also writes supplier, campaign, inventory and purchase-order files to `output/data/`, for a later inventory planning project. They aren't used here.

## Outputs (`output/results_simple/`)

| File | Content |
|---|---|
| `backtest_2025_by_product.csv` | Black Week 2025 forecast per model vs. actual sales |
| `backtest_2025_summary.csv` | Error (WAPE), bias and totals per model |
| `black_week_2026_forecast.csv` | 2026 forecast per product with 80% interval, last year's sales and the difference |
| `forecast_2026_daily.csv` | Daily forecast to 31 Dec 2026 |
