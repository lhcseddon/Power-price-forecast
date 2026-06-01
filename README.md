# German Day-Ahead Power Price Forecasting

A quantitative research project investigating whether renewable generation forecasts
and lagged forecast errors can predict asymmetric price behaviour in the German
day-ahead electricity market.

---

## Research Question

Day-ahead electricity prices are set the day before delivery based on generation
forecasts. When actual renewable generation deviates from those forecasts, the market
has mispriced supply. This project asks:

> Can day-ahead renewable generation forecasts and lagged forecast errors predict
> price asymmetries — particularly negative prices and price spikes?

---

## Key Finding

All three models beat the naive benchmark across every price regime. The largest
improvement is in the **negative price regime**, where XGBoost reduces MAE by 65%
over the benchmark. This confirms that renewable generation forecasts carry
systematic information about downside price risk that a simple lag model ignores.

Hyperparameter tuning via Optuna improved XGBoost MAE from 16.58 to 16.17 €/MWh
and Random Forest from 18.07 to 16.98 €/MWh — modest gains suggesting the
feature set is the primary driver of performance rather than model configuration.

| Regime | N | Benchmark | LASSO | Random Forest | XGBoost | Best improvement |
|---|---|---|---|---|---|---|
| Normal | 11,868 | 29.66 | 23.24 | 15.09 | **14.28** | −52% |
| Negative | 671 | 50.93 | 33.94 | 18.29 | **17.74** | −65% |
| Spike | 660 | 72.47 | 56.79 | 49.73 | **48.47** | −33% |

*All values in €/MWh MAE. Tree models use Optuna-tuned hyperparameters.
Test period: July 2023 – December 2024 (~13,000 hours).*

---

## LASSO Coefficients

LASSO provides an interpretable read of which features matter and in which direction.
All non-zero coefficients are directionally consistent with the underlying mechanism:

| Feature | Coefficient | Interpretation |
|---|---|---|
| `price_lag_24h` | +69.2 | Strong autocorrelation — yesterday's price is the dominant signal |
| `ren_load_ratio_fc` | −28.3 | Higher forecasted renewable share → lower prices ✅ |
| `price_lag_168h` | +16.9 | Same hour last week carries information |
| `price_roll_7d_mean` | +14.0 | Recent price level anchors expectations |
| `net_export_lag24` | +11.9 | High net exports signal tight neighbouring markets |
| `price_roll_7d_std` | +8.6 | Recent volatility raises expected price level |
| `is_weekend` | −7.1 | Lower industrial load on weekends → lower prices ✅ |
| `wind_off_error_lag24` | −1.8 | More offshore wind than expected → lower prices ✅ |
| `wind_on_error_lag24` | −1.0 | More onshore wind than expected → lower prices ✅ |

---

## Methodology

### Data

All data sourced from [SMARD](https://www.smard.de/en) — the German Federal Network
Agency's electricity market data platform. Free, public, no registration required.

| File | Description |
|---|---|
| `day_ahead_prices_smard.csv` | Hourly DE/LU day-ahead prices (€/MWh) |
| `generation_actual_smard.csv` | Actual generation by source (MWh) |
| `generation_forecast_smard.csv` | Day-ahead generation forecast by source (MWh) |
| `consumption_actual_smard.csv` | Actual grid load (MWh) |
| `consumption_forecasts_smard.csv` | Forecasted grid load (MWh) |
| `crossborder_smard.csv` | Cross-border physical flows by country (MWh) |

Period: 01/01/2020 – 31/12/2024, hourly resolution, 43,843 observations.

### Feature Engineering

Features are constructed to be strictly available at prediction time (D-1).
No actual generation values at time t are used — only day-ahead forecasts
and lagged actuals.

**Forecast-based features** (available at prediction time):
- Forecasted renewable load ratio — wind + solar forecast as share of load forecast
- Forecasted renewable surplus — excess renewable forecast over load forecast

**Lagged forecast error features** (yesterday's errors, available at prediction time):
- Wind onshore, wind offshore and solar forecast errors lagged 24h
- Load forecast error lagged 24h
- Net export lagged 24h

**Calendar features:**
- Hour of day and month encoded cyclically (sin/cos)
- Weekend and peak hour indicators

**Lagged price features:**
- Prices at t-24h, t-48h, t-168h
- 7-day rolling mean and standard deviation (computed on lagged prices)

### Models

| Model | MAE (€/MWh) | Notes |
|---|---|---|
| Benchmark | 32.88 | Same hour one week ago |
| LASSO | 25.46 | Standardised, TimeSeriesSplit CV |
| Random Forest | 16.98 | 581 trees, Optuna-tuned |
| XGBoost | 16.17 | 480 estimators, Optuna-tuned |

**Optuna tuning:** 100 trials for XGBoost, 50 for Random Forest, using TPE sampler.
Best XGBoost params: `n_estimators=480, max_depth=6, learning_rate=0.021,
subsample=0.61, colsample_bytree=0.89, min_child_weight=1, gamma=0.96`

### Evaluation

The test set covers July 2023 – December 2024 (18 months, ~13,000 hours).
Train/test split is strictly time-based — no random shuffling.

Evaluation is split into three price regimes:
- **Normal** — all hours outside the other two regimes
- **Negative** — hours where actual price < 0 €/MWh (671 hours, 5.1% of test set)
- **Spike** — hours in the top 5th percentile of actual prices

Overall MAE is insufficient for this problem. A model that predicts average prices
well but misses negative and spike hours is useless for trading decisions. The
regime-split table is the core result.

---

## Caveats

**Spike regime:** Absolute errors remain high even for the best model (48.47 €/MWh).
Extreme price spikes are often driven by gas supply shocks, cold snaps, or
transmission constraints — events not captured by renewable forecast features alone.
Adding TTF natural gas prices is the natural next step.

**Data snooping:** An earlier version of this model used contemporaneous actual
generation values as features, which constitutes look-ahead bias. All features in
the current version are strictly constructed from information available at prediction
time. Results improved after fixing this, suggesting the day-ahead forecast itself
— not the realised error — is the primary price signal.

---

## Project Structure

```
Power-price-forecast/
├── data/
│   ├── raw/                   # original SMARD downloads, never modified
│   └── processed/
│       ├── merged_raw.parquet # cleaned and merged, all columns renamed
│       └── features.parquet   # final feature set used for modelling
├── notebooks/
│   ├── 01_load_and_inspect.ipynb
│   ├── 02_eda.ipynb
│   ├── 03_features.ipynb
│   └── 04_models.ipynb
├── results/
│   └── figures/
├── .gitignore
├── requirements.txt
└── README.md
```

---

## Setup

```cmd
git clone https://github.com/lhcseddon/power-price-forecast.git
cd power-price-forecast
python -m venv venv
venv\Scripts\activate
pip install -r requirements.txt
```

Run notebooks in order: 01 → 02 → 03 → 04.

---

## Tools

Python · pandas · scikit-learn · XGBoost · Optuna · matplotlib · JupyterLab · VSCode

---

## Next Steps

1. **Add TTF gas prices** — gas sets the marginal price during high-demand hours.
   Expected to improve spike regime performance significantly.
2. **LSTM** — capture temporal dependencies beyond lagged features using a
   sequence model.
3. **Intraday vs day-ahead spread** — same data, sharper question about whether
   day-ahead forecast errors predict the intraday correction.