# Predicting-Product-Demand-from-Sales-and-Marketing-Campaigns

"""
External Regressors in Demand Forecasting — Campaign Planning Example
Synthetic data only. No fabricated "real" numbers are claimed anywhere;
this is a fictional product used purely to illustrate the mechanism.
"""

import numpy as np
import pandas as pd
from prophet import Prophet
from sklearn.metrics import mean_absolute_error, mean_squared_error

np.random.seed(42)

# ---------------------------------------------------------------
# 1. Build a synthetic daily demand series (2 years)
# ---------------------------------------------------------------
n_days = 730
dates = pd.date_range("2024-01-01", periods=n_days, freq="D")

t = np.arange(n_days)
trend = 200 + 0.12 * t                                   # slow upward trend
weekly_season = 25 * np.sin(2 * np.pi * t / 7)            # weekly pattern
annual_season = 40 * np.sin(2 * np.pi * t / 365.25)       # annual pattern
noise = np.random.normal(0, 10, n_days)

baseline_demand = trend + weekly_season + annual_season + noise

# ---------------------------------------------------------------
# 2. Define planned campaigns (this is the "campaign planning" input)
#    Each campaign has a start day, duration, and discount depth (%).
#    Deeper discount -> larger demand uplift (with diminishing returns).
# ---------------------------------------------------------------
# Deliberately avoided: no campaign falls within 3 weeks of a real shopping
# holiday (Black Friday/Cyber Monday, end-of-year), and no two campaigns
# land in the same calendar week across different years. This keeps the
# regressor's effect cleanly separable from anything the model's own
# yearly seasonality term might otherwise learn.
campaigns = [
    {"start": 60,  "duration": 5,  "discount": 15},   # early March
    {"start": 150, "duration": 3,  "discount": 30},   # late May
    {"start": 240, "duration": 7,  "discount": 20},   # late August
    {"start": 300, "duration": 4,  "discount": 40},   # late October (clear of Black Friday)
    {"start": 400, "duration": 5,  "discount": 15},   # early February
    {"start": 470, "duration": 3,  "discount": 25},   # mid April
    {"start": 560, "duration": 7,  "discount": 35},   # mid July
    {"start": 620, "duration": 4,  "discount": 20},   # early September
    {"start": 685, "duration": 5,  "discount": 30},   # mid November, still 2+ weeks clear of Black Friday
]

is_campaign = np.zeros(n_days)
campaign_intensity = np.zeros(n_days)  # normalized discount depth, 0-1 scale

for c in campaigns:
    s, d, disc = c["start"], c["duration"], c["discount"]
    is_campaign[s:s + d] = 1
    campaign_intensity[s:s + d] = disc / 100.0

# Uplift model: demand lifts roughly proportional to discount depth,
# with diminishing returns (sqrt) and a bit of noise on the effect size.
uplift_factor = 6.0
demand_uplift = uplift_factor * np.sqrt(campaign_intensity) * baseline_demand * is_campaign
uplift_noise = np.random.normal(0, 5, n_days) * is_campaign

actual_demand = baseline_demand + demand_uplift * 0.35 + uplift_noise
actual_demand = np.clip(actual_demand, a_min=10, a_max=None)

df = pd.DataFrame({
    "ds": dates,
    "y": actual_demand,
    "is_campaign": is_campaign,
    "campaign_intensity": campaign_intensity,
})

# ---------------------------------------------------------------
# 3. Train/test split (last 60 days as holdout, includes 1 campaign)
# ---------------------------------------------------------------
holdout_days = 60
train = df.iloc[:-holdout_days].copy()
test = df.iloc[-holdout_days:].copy()

print(f"Train range: {train['ds'].min().date()} -> {train['ds'].max().date()} ({len(train)} days)")
print(f"Test range:  {test['ds'].min().date()} -> {test['ds'].max().date()} ({len(test)} days)")
print(f"Campaign days in test set: {int(test['is_campaign'].sum())}")

# ---------------------------------------------------------------
# 4. Baseline model — no external regressor
# ---------------------------------------------------------------
baseline_model = Prophet(weekly_seasonality=True, yearly_seasonality=True, daily_seasonality=False)
baseline_model.fit(train[["ds", "y"]])

future_baseline = test[["ds"]].copy()
fc_baseline = baseline_model.predict(future_baseline)

# ---------------------------------------------------------------
# 5. Enriched model — with campaign_intensity as external regressor
# ---------------------------------------------------------------
enriched_model = Prophet(weekly_seasonality=True, yearly_seasonality=True, daily_seasonality=False)
enriched_model.add_regressor("campaign_intensity")
enriched_model.fit(train[["ds", "y", "campaign_intensity"]])

future_enriched = test[["ds", "campaign_intensity"]].copy()
fc_enriched = enriched_model.predict(future_enriched)

# ---------------------------------------------------------------
# 6. Compare accuracy: overall and campaign-days-only
# ---------------------------------------------------------------
def metrics(y_true, y_pred):
    mae = mean_absolute_error(y_true, y_pred)
    rmse = np.sqrt(mean_squared_error(y_true, y_pred))
    mape = np.mean(np.abs((y_true - y_pred) / y_true)) * 100
    return mae, rmse, mape

y_true = test["y"].values
camp_mask = test["is_campaign"].values.astype(bool)

mae_b, rmse_b, mape_b = metrics(y_true, fc_baseline["yhat"].values)
mae_e, rmse_e, mape_e = metrics(y_true, fc_enriched["yhat"].values)

mae_b_c, rmse_b_c, mape_b_c = metrics(y_true[camp_mask], fc_baseline["yhat"].values[camp_mask])
mae_e_c, rmse_e_c, mape_e_c = metrics(y_true[camp_mask], fc_enriched["yhat"].values[camp_mask])

print("\n--- Overall holdout accuracy ---")
print(f"Baseline : MAE={mae_b:.2f}  RMSE={rmse_b:.2f}  MAPE={mape_b:.2f}%")
print(f"Enriched : MAE={mae_e:.2f}  RMSE={rmse_e:.2f}  MAPE={mape_e:.2f}%")
print(f"Improvement (MAE): {(1 - mae_e/mae_b)*100:.1f}%")

print("\n--- Campaign-days-only accuracy ---")
print(f"Baseline : MAE={mae_b_c:.2f}  RMSE={rmse_b_c:.2f}  MAPE={mape_b_c:.2f}%")
print(f"Enriched : MAE={mae_e_c:.2f}  RMSE={rmse_e_c:.2f}  MAPE={mape_e_c:.2f}%")
print(f"Improvement (MAE) on campaign days: {(1 - mae_e_c/mae_b_c)*100:.1f}%")

# ---------------------------------------------------------------
# 7. Save outputs for charting / diagram construction
# ---------------------------------------------------------------
out = test[["ds", "y", "is_campaign", "campaign_intensity"]].copy()
out["yhat_baseline"] = fc_baseline["yhat"].values
out["yhat_enriched"] = fc_enriched["yhat"].values
out.to_csv("/home/claude/campaign_regressor_results.csv", index=False)

summary = {
    "mae_baseline_overall": mae_b, "mae_enriched_overall": mae_e,
    "mae_baseline_campaign": mae_b_c, "mae_enriched_campaign": mae_e_c,
    "improvement_overall_pct": (1 - mae_e/mae_b)*100,
    "improvement_campaign_pct": (1 - mae_e_c/mae_b_c)*100,
}
pd.Series(summary).to_csv("/home/claude/campaign_regressor_summary.csv")
print("\nSaved: campaign_regressor_results.csv, campaign_regressor_summary.csv")
