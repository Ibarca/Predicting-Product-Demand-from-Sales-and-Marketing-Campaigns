# Black Week Demand Planning

A practical demand-planning case study exploring how historical sales, promotional events and planned discounts can help estimate product-level demand for Black Week.

**Business question:** How many units of each product should an online consumer-electronics retailer expect to sell during Black Week 2026 (23–30 November)?

> All data in this project is synthetic. The retailer and products are fictional.

## 1. The Challenge

Black Week creates demand patterns that are difficult to estimate from historical sales alone.

Two factors can distort the picture:

- **Stockouts:** Zero sales may mean that a product was unavailable, not that customers did not want it.
- **Changing discounts:** The same product may have a different discount from one year to the next, affecting demand.

A useful forecast therefore needs to account for the event itself and the promotion planned for the forecast period.

## 2. The Approach

The notebook builds forecasts for products with sales history across all three previous Black Weeks and compares a series of increasingly informed models.

1. Explore daily sales, product information and the marketing calendar.
2. Remove sold-out days from model training so unavailable products do not teach the model that demand was zero.
3. Use Prophet to model historical sales patterns.
4. Add Black Week and other promotions as event effects.
5. Add the planned discount as an explanatory variable (regressor).
6. Backtest the approaches by pretending the forecast was made on 12 October in 2024 and 2025.
7. Compare forecasts with actual sales using weighted absolute percentage error (WAPE).
8. Combine the model forecast with the simple "same as last year" benchmark.

The forecast for 2026 uses sales data through **12 October 2026** and the discounts in the 2026 promotion plan.

## 3. Results

The backtest highlights that knowing when Black Week takes place matters more than adding every available variable. Discount information also helps, particularly when the planned discount differs from previous years. Other promotion events add little in this particular test.

| Method | 2024 WAPE | 2025 WAPE |
|---|---:|---:|
| Same as last year | 22% | 11% |
| Prophet with event, promotion and discount information | 15% | 12% |
| Average of both approaches | 18% | 8% |

The combined approach is used for the final planning forecast. It balances the familiar year-on-year benchmark with a model that can account for event and discount information.

## 4. Black Week 2026 Forecast

For the 21 established products included in the analysis:

| Planning measure | Units |
|---|---:|
| Forecast demand | **3,793** |
| Low planning estimate | 2,713 |
| High planning estimate | 4,511 |
| Black Week 2025 sales for the same products | 3,510 |

The notebook also produces:

- A product-level forecast and planning range.
- A comparison with the previous year's sales and discounts.
- Notes highlighting products that sold out during the previous Black Week or have a changed planned discount.
- An indicative daily demand profile to help think about operational workload.

The low and high estimates are historical error-based planning ranges, not guaranteed outcomes.

## 5. Data

The project uses four input files:

| File | Description |
|---|---|
| `sales_daily.csv` | Daily units sold, discount and stockout flag by product |
| `products.csv` | Product catalogue |
| `marketing_calendar.csv` | Promotional events from 2023 to 2026 |
| `promotion_plan_2026.csv` | Planned discounts for the remainder of 2026 |

## 6. Tools & Technologies

- Python
- pandas and NumPy
- Matplotlib
- Prophet
- Jupyter Notebook

## 7. Limitations

- **Synthetic data:** The results illustrate a method; they are not evidence of performance for a real retailer.
- **Limited history:** The analysis has at most three previous Black Weeks, with only two backtesting years.
- **Stockout treatment:** Sold-out days are excluded from model training, but lost demand is not explicitly estimated.
- **Discount assumptions:** The model learns discount effects from historical observations. Discounts outside the historical range are uncertain.
- **Product scope:** Newer products without sufficient Black Week history require a different forecasting approach.
- **Promotional coverage:** The marketing calendar is shop-wide and may not mean every product participates in every promotion.

## 8. Key Takeaway

Demand planning is not just about extrapolating last year's sales. It requires understanding what makes the forecast period different: event timing, promotional mechanics and product availability.

The goal is to turn that context into a forecast that supports better inventory and operational decisions.

---

*This is an independent portfolio project using synthetic data. It is intended as a practical demonstration of demand-planning and commercial-analytics methods.*
