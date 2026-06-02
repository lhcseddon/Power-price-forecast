# German Day-Ahead Power Price Forecasting

A quantitative research project investigating asymmetric price behaviour in the German
day-ahead electricity market. It covers point forecasting, hyperparameter tuning, and
probabilistic forecasting with conformal calibration, literature benchmarks, statistical
testing, and rolling backtests.

---

## Research Question

Day-ahead electricity prices are set the day before delivery based on generation forecasts.
When actual renewable generation deviates from those forecasts, the market has mispriced
supply. This project tests two related hypotheses:

> 1. Do day-ahead renewable generation forecasts and lagged forecast errors predict price
>    asymmetries in particularly negative prices and spikes?
> 2. Can the full conditional distribution of price be forecast in a calibrated way?

---

## Results

**Point forecasting (notebook 04)**

| Regime | N | Benchmark | LASSO | Random Forest | XGBoost |
|---|---|---|---|---|---|
| Normal | 11,868 | 29.66 | 23.24 | 15.09 | **14.28** |
| Negative | 671 | 50.93 | 33.94 | 18.29 | **17.74** |
| Spike | 660 | 72.47 | 56.79 | 49.73 | **48.47** |

*€/MWh MAE. Benchmark = same hour one week ago. Tree models Optuna-tuned.*

**Probabilistic forecasting (notebook 05)**

| Model | Mean pinball ↓ | Median MAE ↓ | Notes |
|---|---|---|---|
| LEAR (literature benchmark) | 6.92 | 21.44 | asinh transform + LASSO |
| **LightGBM quantile** | **5.16** | **15.91** | best at every quantile |
| Distributional DNN | 6.01 | 18.87 | feedforward, multi-quantile pinball loss |

- **Exact CRPS:** 12.31
- **Calibration:** raw 90% prediction interval covered 72.5%; CQR restored coverage to 87.7%
- **Diebold-Mariano:** LightGBM vs LEAR (DM = −10.46, p < 0.0001); LightGBM vs DNN
  (DM = −8.39, p < 0.0001); DNN vs LEAR (DM = −4.44, p < 0.0001)
- **Rolling backtest:** median monthly MAE 15.69 €/MWh (range 7.59–94.92)

---

## Methodology

### Data

All data from [SMARD](https://www.smard.de/en), the German Federal Network Agency's
electricity market platform. Free, public, no registration required. Period 01/01/2020 –
31/12/2024, hourly, 43,843 observations after cleaning.

| File | Description |
|---|---|
| `day_ahead_prices_smard.csv` | Hourly DE/LU day-ahead prices (€/MWh) |
| `generation_actual_smard.csv` | Actual generation by source (MWh) |
| `generation_forecast_smard.csv` | Day-ahead generation forecast by source (MWh) |
| `consumption_actual_smard.csv` | Actual grid load (MWh) |
| `consumption_forecasts_smard.csv` | Forecasted grid load (MWh) |
| `crossborder_smard.csv` | Cross-border physical flows by country (MWh) |

### Feature engineering

All features are constructed from information available at prediction time (D-1). No
contemporaneous actual generation values are used.

- **Forecast-based:** forecasted renewable load ratio, forecasted renewable surplus
- **Lagged forecast errors (t−24h):** wind onshore, wind offshore, solar, load, net export
- **Calendar:** cyclically encoded hour and month, weekend and peak hour indicators
- **Lagged prices:** t−24h, t−48h, t−168h, 7-day rolling mean and standard deviation

### Models

**Point:** naive benchmark, LASSO (TimeSeriesSplit CV), Random Forest, XGBoost (all
tree models Optuna-tuned with 100 and 50 trials respectively).

**Probabilistic:** LightGBM quantile regression (one model per quantile, monotonicity
enforced), LEAR (asinh-transformed + LASSO), distributional DNN (PyTorch, two hidden
layers, simultaneous multi-quantile pinball loss). Conformal calibration via CQR (Romano
et al., 2019).

### Evaluation

Strictly time-based train/test split (July 2023 – December 2024 test period). Point
forecasts evaluated by regime-split MAE across normal, negative and spike price hours.
Probabilistic forecasts evaluated by pinball loss, exact CRPS (properscoring), and
prediction interval coverage. Pairwise model comparisons tested with Diebold-Mariano
(Harvey-Leybourne-Newbold corrected, h=24). Temporal stability assessed via rolling
two-year window with monthly recalibration.

---

## Caveats

**Spike regime.** Forecast errors remain high (XGBoost 48.47 €/MWh MAE on spikes).
Extreme price events are largely driven by gas supply shocks and transmission constraints
not captured by renewable generation features. Adding TTF natural gas prices is the
natural extension.

**Conformal coverage.** CQR coverage (87.7%) falls short of the nominal 90% target.
Electricity prices are non-stationary, which violates the exchangeability assumption
underlying the conformal guarantee. A time-series-aware conformal method (e.g. adaptive
conformal inference, Gibbs & Candès 2021) would address this.

**Rolling backtest outlier.** The maximum monthly MAE of 94.92 €/MWh corresponds to
the 2021–2022 European energy crisis. Price behaviour during that period was driven by
factors outside the model's feature set.

---

## Project Structure

```
Power-price-forecast/
├── data/
│   ├── raw/                   # original SMARD downloads
│   └── processed/             # merged_raw.parquet, features.parquet
├── notebooks/
│   ├── 01_data_load_inspect.ipynb
│   ├── 02_EDA.ipynb
│   ├── 03_feature_engineering.ipynb
│   ├── 04_models.ipynb
│   └── 05_Probabilistic_forecasting.ipynb
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

Run notebooks in order 01 → 05.

---

## Tools

Python · pandas · scikit-learn · XGBoost · LightGBM · PyTorch · Optuna · scipy · matplotlib

---

## References

- Lago, Marcjasz, De Schutter & Weron (2021). *Forecasting day-ahead electricity prices:
  A review of state-of-the-art algorithms, best practices and an open-access benchmark.*
  Applied Energy.
- Nowotarski & Weron (2018). *Recent advances in electricity price forecasting: A review
  of probabilistic forecasting.* Renewable and Sustainable Energy Reviews.
- Romano, Patterson & Candès (2019). *Conformalized Quantile Regression.* NeurIPS.
- Gibbs & Candès (2021). *Adaptive Conformal Inference Under Distribution Shift.* NeurIPS.