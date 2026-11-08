 
# Taking Sales Forecasting Further: Black Friday Demand Planning with Prophet

In my previous projects, I explored how to build and evaluate sales forecasts using Python and Prophet. I started with [Building My First Sales Forecast in Python with Prophet](https://github.com/Ibarca/Building-My-First-Sales-Forecast-in-Python-with-Prophet/blob/7d96a4d40f1ad34093ce58e52e6a35fa6e6f112e/README.md), where I explained how Prophet uses historical trends and seasonal patterns to predict future sales. I then took it a step further with [Forecast Validation with Prophet: Backtesting Using One Train-Test Split](https://github.com/Ibarca/Forecast-Validation-with-Prophet-Backtesting-Using-One-Train-Test-Split/blob/main/README.md), focusing on how to evaluate forecast accuracy using historical data.

While these exercises helped to understand both forecasting and model evaluation, I was still relying on Prophet's built-in seasonalities without considering additional factors that could influence demand. With Black Friday approaching, I thought it would be a good opportunity to build on what I've learned and explore **how to incorporate promotional events and discount information into Prophet**, this time applying the model to a practical e-commerce demand-planning problem. Let's go!



<p align="center">
  <img src="https://raw.githubusercontent.com/Ibarca/Predicting-Product-Demand-from-Sales-and-Marketing-Campaigns/cdd54471b803b965e2f9e94b4feee944d6729975/Images/black_week_optimized.jpg"
       alt="Black Week"
       width="850">
</p>



## 1. What Am I Adding This Time?

In e-commerce, demand is not only influenced by historical trends and seasonality. Promotional activities, discounts and other commercial factors can also be associated with changes in purchasing behaviour.

This time, I'll introduce two additional elements into Prophet:

- **Promotional events:** Identifying Black Week and other promotional periods so the model can estimate demand patterns associated with these events, particularly since Black Friday falls on different dates each year.
- **Discount percentage:** Including the discount applied to each product as an external regressor, allowing the model to consider historical relationships between discount levels and sales.

The objective is to explore whether this additional information can make the forecasts more useful for demand planning.


<p align="center">
  <img src="https://github.com/Ibarca/Predicting-Product-Demand-from-Sales-and-Marketing-Campaigns/blob/bddf56d92f9a2fa3d48e49df9402d607894a587c/Images/Image%2008.11.26%20at%2017.01.jpeg"
       alt="Adding regressors to Prophet for demand forecasting"
       width="850">
</p>


### What Are Regressors?

Regressors are additional variables that help a forecasting model capture relationships that may not be fully explained by historical trends and seasonality. In e-commerce, demand can be influenced by promotional activities, pricing decisions, marketing campaigns and external factors.

#### Different Types of Regressors

Depending on the information they represent, regressors can take different forms:

| Type | Description | E-commerce examples |
|---|---|---|
| **Binary** | Indicates whether a condition is present (1) or absent (0) | Black Friday, promotional events |
| **Continuous numerical** | Represents a measurable quantity | Discount percentage, product price, marketing spend, temperature |
| **Discrete numerical** | Represents countable quantities | Number of active campaigns, advertising impressions |
| **Categorical** | Represents distinct categories without a natural order | Campaign type, marketing channel |
| **Ordinal** | Represents categories with a meaningful order | Promotion intensity: low, medium, high |

In Prophet, external regressors are added using `add_regressor()`. Since Prophet requires numerical inputs, categorical variables must first be appropriately encoded, for example using binary indicator columns.

For this project, I'll use two external regressors:

| Regressor | Type | Values |
|---|---|---|
| **Promotional event** | Binary | `0` = No event, `1` = Event |
| **Discount percentage** | Continuous numerical | `0–100%` |

Both variables will be included alongside Prophet's trend and seasonality components.

#### How Can Regressors Help Forecast Demand?

Imagine a product that sold **100 units during Black Week last year with a 10% discount**. This year, the retailer plans to offer a **30% discount**.

Should we expect the same demand?

Not necessarily. Historical sales patterns alone might not fully capture how demand varies under different promotional conditions. By incorporating discount percentage as a regressor, Prophet can estimate the historical relationship between discount levels and sales, provided the training data contains enough relevant variation to learn from.

> [!IMPORTANT]
> **Prediction is not the same as causation.**
>
> Prophet might predict higher sales under a 30% discount than under a 10% discount. However, this doesn't prove that increasing the discount would cause the predicted increase in sales. Other factors, such as promotional events, marketing activities or seasonality, may also explain the historical relationship.

#### Important Considerations

There are two important points to keep in mind when using regressors in Prophet:

- **Future values must be available:** Prophet requires regressor values for both historical observations and the forecast period. Planned discounts and promotional calendars are usually known in advance, while future website traffic or competitor prices may need to be estimated separately.
- **More regressors don't necessarily improve accuracy:** Irrelevant variables can introduce noise and increase the risk of overfitting, while highly correlated regressors can make their individual effects difficult to interpret.

For this exercise, I'll evaluate whether incorporating promotional events and discount information improves forecast accuracy through historical backtesting. My objective is to **forecast demand under planned promotional conditions**, rather than measure the causal impact of discounts or identify an optimal discount strategy.

### What Prophet Can and Cannot Tell Us

Before moving forward, I think it's important to understand what Prophet can actually tell us when we introduce promotional events and discount percentages as regressors.

| Question | Can Prophet do it? | Explanation |
|---|---|---|
| Forecast product demand for Black Week 2026? | ✅ Yes | Uses historical sales patterns and the planned values of the regressors to estimate future sales. |
| Account for promotional events? | ✅ Yes | A binary regressor allows Prophet to learn historical sales patterns associated with promotional periods. |
| Incorporate different discount percentages? | ✅ Yes | A numerical regressor allows Prophet to capture historical relationships between discounts and sales. |
| Estimate demand under a planned 30% discount? | ✅ Yes, with caution | Prophet can generate a conditional forecast using that discount, provided the model has learned a useful relationship from historical data. |
| Determine whether regressors improve forecast accuracy? | ✅ With backtesting | Comparing models on historical holdout periods helps evaluate whether the additional variables improve predictions. |
| Calculate how many extra units a 30% discount will cause compared with 10%? | ❌ Not reliably | Prophet can produce different predictions for each discount, but the difference is not necessarily a causal effect. |
| Identify the optimal discount to maximize profit? | ❌ Not on its own | This requires reliable estimates of price response, margins, costs and an optimization approach. |
| Recover demand lost during stockouts? | ❌ Not directly | Prophet forecasts from the available data but doesn't automatically reconstruct unobserved demand. |
| Guarantee accurate forecasts for future Black Weeks? | ❌ No | Forecast accuracy depends on data quality, historical patterns and how representative past events are of future conditions. |

The distinction I want to keep in mind throughout this project is that **forecasting demand under a planned discount is not the same as measuring the causal impact of that discount**.

For example, Prophet might predict 100 units with a 10% discount and 140 units with a 30% discount. However, this doesn't prove that increasing the discount would generate 40 additional sales. The historical relationship could also reflect other factors, such as marketing activity, seasonality or the timing of promotions.

For this exercise, I'll focus on using promotional information to improve demand forecasting and support inventory planning, rather than attempting to optimize discount strategies.




## 2. The Objective

For this exercise, I'll use synthetic data from a fictional consumer-electronics retailer to forecast product-level demand for **Black Week 2026 (23–30 November)**.

The goal is to estimate expected demand using historical sales, promotional events and planned discounts, and explore how these forecasts could support inventory planning.

Before generating the 2026 forecast, I also want to answer one question: **Does incorporating promotional information improve forecast accuracy compared with using historical sales patterns alone?**

To explore this, I'll backtest the approach against Black Week 2025, comparing a basic Prophet model with versions that include promotional events and discount information. This will help me understand whether the additional variables provide useful predictive information, although one historical backtest won't be enough to establish that the same results will hold in future years.

Rather than revisiting forecasting fundamentals or evaluation methods, which I covered in my previous projects, I'll focus on introducing these additional variables, evaluating their contribution and translating the final forecast into practical demand-planning insights.

*All data used in this project is synthetic. The retailer and products are fictional.*

## 3. The Dataset

For this exercise, I'm using synthetic data from a fictional consumer-electronics retailer, covering daily sales between January 2023 and October 2026, together with product information and a marketing calendar.

The original catalogue contains 40 products, but not all of them are suitable for this exercise. Some were launched recently, while others have been discontinued or replaced by newer generations.

To keep the analysis focused, I've selected **21 established products** with sales history across Black Week 2023, 2024 and 2025 that remain active for Black Week 2026.

The dataset includes three main files:

| Dataset | Information |
|---|---|
| `sales_daily.csv` | Daily sales, discounts and stockout information |
| `products.csv` | Product catalogue and lifecycle dates |
| `marketing_calendar.csv` | Historical and planned promotional events |

### Handling Stockouts

One important consideration when forecasting demand is that **observed sales don't always represent actual demand**.

If a product runs out of stock, sales may fall even though customers still want to purchase it. Training a model on these observations without accounting for stock availability could lead to underestimating demand.

Fortunately, the dataset includes a `stockout_flag` identifying affected observations. I'll exclude these days from model training rather than treating them as normal demand observations.

This doesn't recover the demand that was lost during stockouts, but it avoids teaching the model that constrained sales necessarily reflect lower customer interest. I'll keep the original observations for historical reporting and exclude stockout-affected days from the main accuracy calculation.

## 4. Preparing the Promotional Variables

The next step is to prepare the additional information that Prophet will use alongside historical sales.

### Promotional Events

Black Week takes place every year, but its dates change. Rather than expecting Prophet's yearly seasonality to capture this moving event automatically, I'll explicitly identify the historical and planned promotional periods.

For this exercise, I'll use Prophet's holiday functionality to represent Black Week dates. This allows the model to estimate an event effect separately from its regular seasonal patterns.

However, there are only three historical Black Weeks in the dataset, so the amount of information available to estimate this effect is limited.

### Discount Percentage

The second variable is the discount percentage applied to each product.

I'll add this as an external regressor using Prophet's `add_regressor()` functionality, allowing the model to estimate how sales have historically varied with discount levels.

For the final forecast, I'll provide the planned discounts for Black Week 2026. This is important because Prophet needs future values for external regressors when generating predictions.

One challenge is that discounts and promotional events often occur together. For example, a retailer may offer its largest discounts during Black Week, when demand is already unusually high. This makes it difficult to distinguish the contribution of the discount from the effect of the event itself.

For this reason, I'll interpret the discount regressor as a predictive variable rather than a measure of how much additional demand a particular discount will cause.

## 5. Building and Evaluating the Forecast

I'll train separate Prophet models for each of the 21 selected products, using daily sales history and the additional promotional information.

Before generating the Black Week 2026 forecast, I'll evaluate the approach using a historical backtest. The idea is to recreate a forecasting exercise for Black Week 2025 using only the sales history available before that event.

I'll compare three approaches:

| Approach | Variables |
|---|---|
| Baseline Prophet | Historical sales, trend and seasonality |
| Prophet + Events | Baseline plus promotional event indicators |
| Prophet + Events + Discounts | Promotional events plus discount percentage |

The comparison will help me understand whether the additional variables improve predictive performance. However, the main objective remains building a useful demand forecast, not identifying a universally superior model.

### Evaluating Accuracy

I'll evaluate the forecasts using **Weighted Absolute Percentage Error (WAPE)**, calculated as the sum of absolute forecast errors divided by the sum of observed sales.

I'll calculate WAPE using non-stockout observations to avoid evaluating the model against sales that may have been constrained by inventory availability.

For the discount-based backtest, I'll also distinguish between discounts that were known at the forecast cutoff and realized discounts. If only realized discounts are available, the evaluation will be conditional on those discounts and won't fully represent a real historical planning exercise.

The results will help me decide how much confidence to place in the additional variables before forecasting 2026.

**[Insert model comparison chart and backtesting results]**

## 6. Forecasting Black Week 2026

After evaluating the models, I'll use the available sales history through October 2026 and the planned promotional information to forecast demand for Black Week 2026.

The forecast will focus on expected units sold for each product, allowing me to compare anticipated demand with historical Black Week performance.

I'll look at three main questions:

- How many units are expected to sell across the selected products?
- Which products show the largest expected demand changes compared with 2025?
- Which products deserve closer attention when preparing inventory?

**[Insert total Black Week 2026 forecast]**

**[Insert product-level forecast comparison]**

The forecasts will represent estimates based on historical patterns and the planned promotional conditions, not guaranteed sales outcomes.

## 7. Translating the Forecast into Demand Planning

The most interesting part of this exercise is understanding how the forecasts could support commercial decisions.

For example, if a product is expected to experience significantly higher demand than in the previous year, a retailer may want to review whether sufficient inventory will be available. Similarly, products with previous Black Week stockouts may deserve additional attention.

However, forecasting demand is only one part of inventory planning. To calculate actual replenishment requirements, we would also need current stock levels, incoming deliveries, supplier lead times and potentially safety-stock targets.

Since this exercise focuses on demand forecasting, I'll use the results to identify potential planning priorities rather than calculate exact purchase orders.

**[Insert product-level planning recommendations based on actual results]**

## 8. Limitations

There are several limitations worth keeping in mind when interpreting the results.

First, the dataset is synthetic, so the findings don't represent the performance of a real retailer.

Second, the model has only three historical Black Weeks from which to learn promotional effects. This makes it difficult to draw strong conclusions about the relationship between discounts and demand.

Third, excluding stockout-affected observations avoids treating constrained sales as normal demand, but it doesn't estimate the sales that were lost.

Finally, the backtest focuses on Black Week 2025. Although this is useful for understanding the model's historical performance, additional evaluation periods would be needed to assess how reliably the approach performs across different years.

## 9. Key Takeaways

*To be completed after running the analysis.*

With this project, I want to move beyond forecasting based only on trends and seasonality and explore how additional commercial information can be incorporated into Prophet.

The main learning objective is to understand how promotional events and discount information affect the forecasting process, whether they provide useful predictive information and how the resulting forecasts can support demand planning.

Ultimately, I'm interested in connecting the technical side of forecasting with the commercial decisions that retailers need to make before major promotional events.

---

*All data used in this project is synthetic. The retailer and products are fictional.*
