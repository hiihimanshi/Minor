# Does the Most Accurate Forecast Give the Lowest Inventory Cost? (Spare-Parts Demand)

A reproducible comparison of intermittent-demand forecasting methods on **real spare-parts data**, measuring
both forecast accuracy and the **inventory cost** each forecast leads to (month-by-month simulation with each
item's own lead time and price).

## What it does
1. Loads real data (RAF: 5,000 items, 84 months, lead time + price; OIL: 14,500 materials, 56 months, lead time + price).
2. Classifies items by ADI / CV² (smooth, erratic, intermittent, lumpy) using training data only.
3. Forecasts with Moving Average, Croston, SBA, TSB, a global LightGBM (Tweedie loss) and a data-driven Adaptive selector.
4. Evaluates with rolling-origin MASE, RMSSE and bias on a time-based 12-month test split.
5. Simulates inventory (order-up-to policy, lost sales) under a flat and a criticality-aware stocking policy.
6. Compares accuracy against cost, including cost at the **same fill rate**, and runs a sensitivity analysis.

## Run it
Open `Minor_Project_final.ipynb` in Google Colab, then Runtime > Run all.
Change only the Settings cell (`DATASET = 'RAF'`, `'OIL'` or `'CARPARTS'`; `SAMPLE_N = None` uses every item).

## Results (2,000-item random samples, seed 42)
See `results_and_figures.zip` (CSV tables, figures, executed notebooks) and the written report.

## Data (not redistributed here)
Downloaded at run time from https://github.com/danieldehaan96/spdf (RAF, OIL). Please cite that repository and the
thesis it accompanies. Car Parts (optional): Zenodo record 4656021 (CC-BY 4.0).

## Honest limitations
- RAF is 1996-2002 data and OIL's units are undocumented; neither is guaranteed to represent modern machine parts.
- Holding rate, shortage-cost multipliers and criticality classes are **assumptions** (not in the data); the sensitivity analysis shows how much conclusions depend on them.
- One chronological train/test split per dataset; results are on 2,000-item samples.
- OIL preprocessing: negative demand (returns) set to 0; one system-wide anomaly month removed automatically.
