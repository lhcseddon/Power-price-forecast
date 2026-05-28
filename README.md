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

| Regime | N | Benchmark | LASSO | Random Forest | XGBoost | Best improvement |
|---|---|---|---|---|---|---|
| Normal | 11,868 | 29.66 | 23.24 | 16.02 | **14.78** | −50% |
| Negative | 671 | 50.93 | 33.94 | 19.77 | **17.62** | −65% |
| Spike | 660 | 72.47 | 56.79 | 53.18 | **47.97** | −34% |

*All values in €/MWh MAE. Test period: July 2023 – December 2024.*

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
No actual generation values at time t are used — only day-ahead forecasts and
lagged actuals.

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

Three models are trained and compared against a naive benchmark:

- **Benchmark** — same hour one week ago (`price_lag_168h`)
- **LASSO** — with standardisation and 5-fold time-series cross-validation,
  selected for interpretability. Coefficients show which features survive
  regularisation and in which direction.
- **Random Forest** — 500 trees, max depth 10
- **XGBoost** — 500 estimators, learning rate 0.05, early stopping on validation set

### Evaluation

The test set covers July 2023 – December 2024 (18 months, ~13,000 hours).
Train/test split is strictly time-based — no random shuffling.

Evaluation is split into three price regimes:
- **Normal** — all hours outside the other two regimes
- **Negative** — hours where actual price < 0 €/MWh
- **Spike** — hours in the top 5th percentile of actual prices

Overall MAE is insufficient for this problem. A model that predicts average prices
well but misses negative and spike hours is useless for trading decisions. The
regime-split table is the core result.

---

## Caveats

**Spike regime:** Absolute errors remain high even for the best model (47.97 €/MWh).
Extreme price spikes are often driven by gas supply shocks, cold snaps, or
transmission constraints — events not captured by renewable forecast features alone.
Adding TTF natural gas prices as a feature is the natural next step.

**Data snooping:** An earlier version of this model used contemporaneous actual
generation values as features, which constitutes look-ahead bias. All features in
the current version are strictly constructed from information available at prediction
time. Results are slightly better after fixing this, which suggests the day-ahead
forecast itself — not the realised error — is the primary price signal.

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

Python · pandas · scikit-learn · XGBoost · matplotlib · JupyterLab · VSCode

---

## Next Steps

1. **Add TTF gas prices** — gas sets the marginal price during high-demand hours.
   Expected to improve spike regime performance significantly.
2. **Intraday vs day-ahead spread** — same data, sharper question about whether
   day-ahead forecast errors predict the intraday correction.
