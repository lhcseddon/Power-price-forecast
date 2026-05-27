# German Day-Ahead Power Price Forecasting

A quantitative research project investigating whether renewable generation forecast
errors can predict asymmetric price behaviour in the German day-ahead electricity market.

---

## Research Question

When actual wind and solar generation deviates from day-ahead forecasts, the market
has mispriced. Are these forecast errors a systematic predictor of price deviations —
particularly for negative and spike price episodes?

---

## Getting Started

### Prerequisites

- Python 3.10 or higher
- Git

Check your Python version:

```cmd
python --version
```

---

### 1. Clone the repository

```cmd
git clone https://github.com/yourusername/power-price-forecast.git
cd power-price-forecast
```

---

### 2. Create and activate the virtual environment

```cmd
python -m venv venv
venv\Scripts\activate
```

Your terminal prompt should now show `(venv)` at the start.
**You need to run this every time you open a new terminal.**

To deactivate when you're done:

```cmd
deactivate
```

---

### 3. Install dependencies

```cmd
pip install -r requirements.txt
```

---

### 4. Set your ENTSO-E API key

You need a free API key from [transparency.entsoe.eu](https://transparency.entsoe.eu).
Request access by emailing transparency@entsoe.eu with your account email.

Once you have the key, set it as an environment variable:

```cmd
set ENTSOE_API_KEY=your_token_here
```

To set it permanently, search for **"Edit environment variables for your account"**
in the Windows Start menu, click **New**, and add:
- Variable name: `ENTSOE_API_KEY`
- Variable value: your token

Verify it is set:

```cmd
python -c "import os; print(os.environ.get('ENTSOE_API_KEY'))"
```

---

### 5. Launch Jupyter

```cmd
jupyter lab
```

This opens in your browser. Run the notebooks in order:

| Notebook | Description |
|---|---|
| `01_data_fetch.ipynb` | Fetch prices, generation, load and forecasts from ENTSO-E |
| `02_eda.ipynb` | Explore price distributions and identify asymmetries |
| `03_features.ipynb` | Engineer features from causal hypotheses |
| `04_models.ipynb` | Train and evaluate LASSO and XGBoost models |

---

## Project Structure

```
power-price-forecast/
├── data/
│   ├── raw/               # raw data from ENTSO-E (gitignored)
│   └── processed/         # feature-engineered data (gitignored)
├── notebooks/
│   ├── 01_data_fetch.ipynb
│   ├── 02_eda.ipynb
│   ├── 03_features.ipynb
│   └── 04_models.ipynb
├── src/
│   ├── fetch.py           # data fetching functions
│   ├── features.py        # feature engineering
│   └── models.py          # model training and evaluation
├── results/
│   └── figures/           # saved plots
├── .gitignore
├── requirements.txt
└── README.md
```

---

## Methodology

1. **EDA** — confirm asymmetric price distribution and identify the driving mechanism
2. **Feature engineering** — build features from causal hypotheses, not automated selection
3. **Modelling** — LASSO for interpretability, XGBoost for performance
4. **Evaluation** — regime-split evaluation across normal, negative and spike price hours

---

## Data Source

[ENTSO-E Transparency Platform](https://transparency.entsoe.eu) — free, public data on
European power generation, load and prices.

---

## Results

*To be updated once modelling is complete.*

| Model | MAE (€/MWh) | Spike MAE | Negative MAE |
|---|---|---|---|
| Naive benchmark | — | — | — |
| LASSO | — | — | — |
| XGBoost | — | — | — |

---

## Tools

Python · pandas · scikit-learn · XGBoost · entsoe-py · matplotlib · JupyterLab