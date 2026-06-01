# German Day-Ahead Power Price Forecasting

A quantitative research project on the German day-ahead electricity market. It starts from a
causal hypothesis about renewable-driven price asymmetries, builds and tunes point-forecasting
models, and then extends to research-grade **probabilistic forecasting** with proper
distributional evaluation, literature benchmarks, statistical testing, and rolling backtests.

---

## Research Question

Day-ahead electricity prices are set the day before delivery based on generation forecasts.
When actual renewable generation deviates from those forecasts, the market has mispriced
supply. This project asks:

> Can day-ahead renewable generation forecasts and lagged forecast errors predict price
> asymmetries — and can the full distribution of price be forecast in a calibrated, defensible
> way?

---

## Headline Results

**Point forecasting (notebook 04).** Across a regime-split evaluation, XGBoost beats a naive
benchmark in every regime, with the largest gain in the negative price regime (−65% MAE).

| Regime | N | Benchmark | LASSO | Random Forest | XGBoost |
|---|---|---|---|---|---|
| Normal | 11,868 | 29.66 | 23.24 | 15.09 | **14.28** |
| Negative | 671 | 50.93 | 33.94 | 18.29 | **17.74** |
| Spike | 660 | 72.47 | 56.79 | 49.73 | **48.47** |

*€/MWh MAE. Tree models Optuna-tuned.*

**Probabilistic forecasting (notebook 05).** LightGBM quantile regression is the best
distributional model, significantly outperforming both the LEAR literature benchmark and a
distributional DNN.

| Model | Mean pinball ↓ | Median MAE ↓ | Notes |
|---|---|---|---|
| LEAR (literature benchmark) | 6.916 | 21.44 | asinh transform + LASSO |
| **LightGBM quantile** | **5.16** | **15.91** | best at every quantile |
| Distributional DNN | 6.01 | 18.87 | competitive but behind on tabular data |

- **Exact CRPS:** 12.31
- **Calibration:** raw 90% prediction interval covered only 72.5% — overconfident. Conformalized
  Quantile Regression (CQR) restored coverage to 87.7%.
- **Statistical significance (Diebold-Mariano):** LightGBM beats LEAR (DM = −10.46, p < 0.0001)
  and the DNN (DM = −8.39, p < 0.0001); DNN beats LEAR (DM = −4.44, p < 0.0001).
- **Stability (rolling backtest):** median monthly MAE 15.69 €/MWh (range 7.59–94.92); the
  single large outlier is the 2021–2022 energy crisis.

---

## What Makes This More Than a Tutorial

- **Leakage-free feature design.** Every feature is constructed from information available at
  prediction time (day-ahead forecasts and lagged actuals). An earlier version that used
  contemporaneous actual generation was identified as look-ahead bias and corrected.
- **Distributional forecasting done correctly.** Evaluated with pinball loss, CRPS and
  calibration — not MAE applied to intervals.
- **Conformal prediction.** Diagnosed overconfident intervals and repaired them with CQR,
  including an honest treatment of why the conformal guarantee is only approximate on
  non-stationary price data.
- **Literature benchmarks.** Implements LEAR (Lago et al.) and a distributional DNN, and
  explains why gradient boosting wins on tabular data.
- **Statistical testing.** Every model comparison is validated with a Diebold-Mariano test
  (Harvey-Leybourne-Newbold corrected), not eyeballed.
- **Rolling-window backtesting.** Performance evaluated as a distribution over time, isolating
  the exact market regime where the model breaks down.

---

## Methodology

### Data

All data from [SMARD](https://www.smard.de/en), the German Federal Network Agency's electricity
market platform. Free, public, no registration. Period 01/01/2020 – 31/12/2024, hourly,
43,843 observations.

| File | Description |
|---|---|
| `day_ahead_prices_smard.csv` | Hourly DE/LU day-ahead prices (€/MWh) |
| `generation_actual_smard.csv` | Actual generation by source (MWh) |
| `generation_forecast_smard.csv` | Day-ahead generation forecast by source (MWh) |
| `consumption_actual_smard.csv` | Actual grid load (MWh) |
| `consumption_forecasts_smard.csv` | Forecasted grid load (MWh) |
| `crossborder_smard.csv` | Cross-border physical flows by country (MWh) |

### Features

Forecast-based (available at prediction time): forecasted renewable load ratio and renewable
surplus. Lagged forecast errors (yesterday's, available at prediction time): wind onshore,
wind offshore, solar and load forecast errors, plus net export. Calendar: cyclically encoded
hour and month, weekend and peak indicators. Lagged prices: t-24h, t-48h, t-168h, and 7-day
rolling mean and standard deviation.

### Models

Point: naive benchmark, LASSO, Random Forest, XGBoost (Optuna-tuned).
Probabilistic: LightGBM quantile regression, LEAR, distributional DNN (PyTorch, multi-quantile
pinball loss). Conformal calibration via CQR.

### Evaluation

Strictly time-based train/test split (no shuffling). Point forecasts evaluated by regime-split
MAE; probabilistic forecasts by pinball loss, CRPS and interval coverage. Model comparisons
tested with Diebold-Mariano. Stability assessed by rolling-window monthly recalibration.

---

## Caveats

- **Spike regime.** Absolute errors stay high for the hardest prices; these are driven by gas
  shocks and grid constraints outside the current feature set. Adding TTF gas prices is the
  natural next step.
- **Conformal coverage.** CQR improves but does not perfectly reach nominal coverage because
  electricity prices are non-stationary and violate the exchangeability assumption. A
  time-series-aware conformal method would close the gap.

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

- Lago, Marcjasz, De Schutter & Weron (2021). *Forecasting day-ahead electricity prices: A
  review of state-of-the-art algorithms, best practices and an open-access benchmark.*
  Applied Energy.
- Nowotarski & Weron (2018). *Recent advances in electricity price forecasting: A review of
  probabilistic forecasting.* Renewable and Sustainable Energy Reviews.
- Romano, Patterson & Candès (2019). *Conformalized Quantile Regression.* NeurIPS.
