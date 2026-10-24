# Forecast Accuracy vs Inventory Cost for Intermittent Spare-Parts Demand

*Minor Project report

## Abstract

Spare-parts demand is intermittent: most months have no demand at all, and standard accuracy measures can reward forecasts that are poor for stocking. This project asks: **does the forecasting method with the best accuracy also lead to the lowest inventory cost, and does stocking by part criticality reduce cost?** Five forecasting methods (moving average, Croston, SBA, TSB and a global LightGBM model) plus a data-driven selector were compared on two real, public spare-parts datasets (RAF and OIL, 2,000 items each) using rolling-origin evaluation and a month-by-month inventory simulation with each item's own lead time and price. Accuracy and cost rankings agreed only weakly (rank correlation 0.12 on RAF and 0.20 on OIL). The moving average was competitive on accuracy but the most expensive in both datasets. No method was best on both datasets: LightGBM was cheapest on RAF and TSB on OIL. Criticality-aware stocking reduced cost by about 9% under the baseline assumptions on both datasets, but the saving ranged from roughly 0% to 26% across the assumptions tested. The criticality classes and cost parameters are assumptions, not data, and this limits how far the cost conclusions generalise.

## 1. Introduction and research question

Industrial organisations hold spare parts to avoid downtime, but demand for each part is sporadic and hard to forecast. Forecasting research usually judges methods by accuracy, whereas the business cares about inventory cost and service. The two can disagree: a method can be accurate on average (mostly by predicting small numbers in the many zero-demand months) yet leave too little stock when demand does occur.

**Research questions**

1. Does the most accurate forecasting method also give the lowest inventory cost?
2. When methods are compared at the same service level (fill rate), which is cheapest?
3. How much does criticality-aware stocking reduce cost compared with one service level for all parts, and how sensitive is this to assumptions?

**Contribution.** The contribution is a fair, reproducible comparison that links forecast accuracy to inventory cost on real data. It is not a new algorithm. An earlier master's thesis (de Haan, repository listed in the references) already benchmarked several of these methods on accuracy and inventory performance; this project differs by testing the accuracy-to-cost link directly, comparing methods at equal fill rate, and adding criticality-based stocking with a sensitivity analysis.

## 2. Related work

Intermittent demand is conventionally forecast with Croston's method (Croston, 1972), which smooths demand size and demand interval separately. Syntetos and Boylan (2005) showed Croston is biased and proposed a correction (SBA). Teunter, Syntetos and Babai (2011) proposed TSB, which updates the probability of demand every period and therefore handles obsolescence. Demand patterns are classified by the average demand interval (ADI) and the squared coefficient of variation of demand sizes (CV²) using the cut-offs 1.32 and 0.49 (Syntetos, Boylan and Croston, 2005). Hyndman and Koehler (2006) proposed scaled errors such as MASE that remain defined when demand contains zeros. Gradient boosting (Ke et al., 2017) is included as a machine-learning comparison. According to the repository summary, the thesis behind the RAF and OIL data reported that SBA was the most accurate overall, Willemain's bootstrap was best on inventory performance, and machine-learning methods did relatively better on inventory under extreme intermittency. *(Please check all citation details against the original papers before submission.)*

## 3. Data

Both datasets are public and come from the repository listed in the references.

| Dataset | Items used | Months | Zero-demand share | Lead time (median / max) | Price available |
| --- | --- | --- | --- | --- | --- |
| RAF | 2,000 random items of 4,840 eligible | 84 (Jan 1996 – Dec 2002) | 90% | 9 / 33 months | Yes (£) |
| OIL | 2,000 random items of 3,737 eligible | 55 after cleaning (56 raw) | 87% | 1 / 11 months | Yes |

*Eligibility:* at least 4 months with demand in the training period. *Split:* the last 12 months are the test period (time-based, never random); the 12 months before it are used to measure forecast-error spread.

**OIL cleaning.** 0.45% of OIL cells were negative (returns or adjustments) and were set to zero. One month had demand recorded for 28.8% of items versus a median of 5.9%, with total demand about 50 times normal; this is a recording event rather than real demand and fell inside the test window, so it was removed by an automatic rule (a month is removed if far more items than usual show demand). Without this step the OIL results were meaningless (fill rate near 13%).

## 4. Method

**Classification.** Each item is classified on training data only as smooth, erratic, intermittent or lumpy (ADI, CV²).

**Forecasting methods.** Moving average (6 months); Croston and SBA (smoothing 0.1); TSB (0.1 for size and probability); a single global LightGBM model across all items using Tweedie loss, six lags, rolling means, demand frequency, months since last demand and calendar month, trained only on months before the test period; and an *Adaptive* selector that picks, for each demand class, the method with the lowest error on the calibration window. Forecasts are rolling-origin: each month is forecast from data up to the previous month. A leakage test confirmed that changing future data does not change earlier forecasts.

**Accuracy metrics.** MASE and RMSSE (error scaled by a naive forecast, defined with zeros) and scaled bias (positive means over-forecasting), averaged over items and all 12 test months. Paired Wilcoxon tests compare methods item by item.

**Inventory simulation.** For each item and method, a monthly order-up-to policy is simulated over the test year with the item's own lead time L. The target level is S = forecast × L + z × σ × √L, where σ is the forecast-error spread on the calibration window. Unmet demand is lost and penalised. The system starts in steady state. Holding cost is 20% of unit price per year; shortage cost per unit is a multiple of unit price by class: Critical 1.0, Essential 0.3, Desirable 0.1 (classes randomly assigned 20% / 30% / 50%). **These costs and classes are assumptions.** Two policies are compared: *flat* (z = 1.65 for every item) and *criticality-aware* (z from the newsvendor ratio z = Φ⁻¹(s / (s + h)), so critical items get more safety stock).

**Equal-service comparison.** Every method is simulated over a grid of z values to draw its cost-versus-fill-rate curve, and costs are interpolated at the same fill rates.

## 5. Results

### 5.1 Demand classes

All sampled items were intermittent or lumpy: RAF 1,248 intermittent and 752 lumpy; OIL 1,694 intermittent and 306 lumpy. There were no smooth or erratic items, so class-based routing has little to choose between.

### 5.2 Forecast accuracy (test period, lower is better except bias near 0)

| RAF | RMSSE | MASE | Bias |
| --- | --- | --- | --- |
| TSB | **0.602** | 0.946 | −0.029 |
| LightGBM (= Adaptive) | 0.612 | 0.967 | 0.064 |
| Moving average | 0.615 | **0.942** | −0.059 |
| SBA | 0.626 | 1.069 | 0.246 |
| Croston | 0.632 | 1.098 | 0.310 |

| OIL | RMSSE | MASE | Bias |
| --- | --- | --- | --- |
| Adaptive | **0.502** | 0.798 | 0.223 |
| TSB | 0.504 | 0.819 | 0.272 |
| Moving average | 0.512 | **0.751** | 0.119 |
| LightGBM | 0.539 | 0.878 | 0.363 |
| SBA | 0.573 | 1.001 | 0.606 |
| Croston | 0.582 | 1.034 | 0.674 |

On both datasets the differences between TSB, the moving average and LightGBM are small, while Croston and SBA over-forecast clearly (bias 0.25 to 0.67). Paired tests give very small p-values because 2,000 items are compared, so statistical significance should be read together with the size of the differences.

### 5.3 Inventory cost (flat policy, total over 12 test months)

| RAF | Total cost | Fill rate |
| --- | --- | --- |
| LightGBM (= Adaptive) | **261,811** | 63.9% |
| SBA | 272,269 | 64.2% |
| Croston | 275,172 | 65.1% |
| TSB | 278,042 | 58.7% |
| Moving average | 299,438 | 53.9% |

| OIL | Total cost | Fill rate |
| --- | --- | --- |
| TSB | **254,222** | 67.8% |
| SBA | 254,470 | 67.1% |
| Adaptive | 255,497 | 67.1% |
| Croston | 256,142 | 67.9% |
| LightGBM | 258,238 | 66.3% |
| Moving average | 265,636 | 65.7% |

On RAF, LightGBM is 4% cheaper than SBA and 14% cheaper than the moving average. On OIL the four best methods are within 1.6% of each other and only the moving average is clearly worse (about 4.5% above TSB). In an item-by-item paired test, LightGBM was cheaper than SBA on RAF (p < 0.001) but not on OIL (p = 0.30).

### 5.4 Does accuracy predict cost?

The rank correlation between RMSSE rank and cost rank was **0.12 on RAF and 0.20 on OIL** (six methods, so weak evidence either way). The clearest disagreement is the moving average: it ranks 2nd to 4th on accuracy (best MASE on both datasets) but is the most expensive method on both. On RAF, TSB has the best RMSSE but only the fourth-lowest cost.

### 5.5 Cost at the same service level

Total cost interpolated at the same fill rate (lower is better):

| RAF | Fill 40.1% | Fill 56.1% | Fill 72.0% |
| --- | --- | --- | --- |
| LightGBM | **168,442** | **213,579** | 347,648 |
| Croston | 174,642 | 216,918 | **346,288** |
| SBA | 174,929 | 219,878 | 353,044 |
| TSB | 188,474 | 259,326 | 462,000 |
| Moving average | 225,943 | 320,830 | 604,333 |

| OIL | Fill 38.1% | Fill 59.9% | Fill 81.7% |
| --- | --- | --- | --- |
| TSB | **257,262** | **245,099** | 366,268 |
| SBA | 258,145 | 249,160 | 349,975 |
| Croston | 258,538 | 249,445 | **344,309** |
| LightGBM | 259,225 | 251,887 | 355,495 |
| Moving average | 261,674 | 257,085 | 424,315 |

At equal service the moving average is clearly the most expensive on both datasets and TSB becomes expensive at higher service on RAF. The best method depends on the dataset and the service level: LightGBM leads at low and medium service on RAF, TSB at low and medium service on OIL, and Croston at the highest service level on both. Differences among the leading methods are small (typically 1 to 4%).

### 5.6 Criticality-aware stocking and sensitivity

Under baseline assumptions, criticality-aware stocking reduced total cost by **9.4% on RAF** (LightGBM) and **9.0% on OIL** (TSB), mainly by cutting shortage cost for critical items while holding fewer units of low-priority items. The saving depends strongly on the assumptions (saving in % of flat-policy cost, for lead times scaled by 0.5, 1 and 1.5):

| Shortage cost × | RAF | OIL |
| --- | --- | --- |
| 0.5 | 21.5 / 24.2 / 25.6 | 15.0 / 17.7 / 19.8 |
| 1.0 | 8.3 / 9.4 / 9.6 | 8.0 / 9.0 / 9.2 |
| 2.0 | 1.9 / 0.3 / −1.2 | 9.9 / 8.7 / 5.7 |

On RAF the saving disappears when shortage costs are doubled. Because the cost parameters and class labels are assumed, this result shows the *mechanism and its sensitivity*, not a guaranteed saving.

## 6. Discussion

1. **Accuracy is not a reliable guide to cost.** The moving average illustrates this on both datasets. A plausible explanation, which this project did not test, is that scaled errors averaged over all months are dominated by the many zero-demand months, whereas stocking cost is driven by the size of demand when it occurs and by the shortfall in forecasts at those times.
2. **There is no universal winner.** LightGBM was cheapest on RAF but not on OIL, so a method should be validated on the organisation's own data. Simple methods (SBA, TSB, Croston) stayed competitive, which agrees with the general finding that complex models rarely beat simple ones decisively on intermittent data.
3. **Method choice matters less than policy design.** Differences among the leading methods were mostly 1 to 4%, while criticality-aware stocking changed cost by about 9% at baseline. The dominant uncertainty is the cost and criticality assumptions.
4. **The Adaptive selector did not help consistently.** On RAF it selected LightGBM for both classes and so duplicated it; on OIL it achieved the best RMSSE but was only third on cost.

## 7. Limitations

- Criticality classes and the shortage and holding cost parameters are assumptions; no real downtime-cost data exist in the datasets.
- RAF covers 1996 to 2002 and appears to be general stores items; OIL's units are not documented. Neither is guaranteed to represent modern machine spare parts.
- A single chronological train/test split per dataset and 2,000-item samples were used; repeated rolling windows or full-dataset runs would give more robust estimates.
- Smoothing constants and LightGBM settings were fixed, not tuned. Safety stock assumes independent forecast errors and uses 12 months of errors, which is noisy for sparse items.
- Rank correlations are based on six methods and should be treated as indicative.
- No machine-condition or failure data were used, so the link between predictive maintenance and spares demand is not tested.

## 8. Conclusion and future work

On two real spare-parts datasets, forecast accuracy was only weakly related to inventory cost, the moving average was consistently costly despite competitive accuracy, no forecasting method won on both datasets, and criticality-based stocking gave a saving of about 9% under baseline assumptions that was sensitive to cost assumptions. Practitioners should therefore evaluate forecasting methods by simulated cost and service on their own data rather than by accuracy alone. Future work: (1) real downtime and criticality data, (2) prediction intervals (for example conformal prediction) instead of a single error spread, (3) failure or remaining-useful-life information as a demand driver, (4) repeated rolling windows and tuned parameters, (5) additional methods such as bootstrap-based forecasting.

## References

- Croston, J. D. (1972). Forecasting and stock control for intermittent demands. *Operational Research Quarterly*, 23(3), 289–303.
- de Haan, D. Spare part demand forecasting datasets and benchmark (master's thesis repository). https://github.com/danieldehaan96/spdf
- Hyndman, R. J., and Koehler, A. B. (2006). Another look at measures of forecast accuracy. *International Journal of Forecasting*, 22(4), 679–688.
- Ke, G. et al. (2017). LightGBM: a highly efficient gradient boosting decision tree. *Advances in Neural Information Processing Systems 30*.
- Syntetos, A. A., and Boylan, J. E. (2005). The accuracy of intermittent demand estimates. *International Journal of Forecasting*, 21(2), 303–314.
- Syntetos, A. A., Boylan, J. E., and Croston, J. D. (2005). On the categorization of demand patterns. *Journal of the Operational Research Society*, 56(5), 495–503.
- Teunter, R. H., Syntetos, A. A., and Babai, M. Z. (2011). Intermittent demand: linking forecasting to inventory obsolescence. *European Journal of Operational Research*, 214(3), 606–615.

## Appendix: reproducibility

Code: `Minor_Project_final.ipynb` (Google Colab). Seed 42; test period 12 months; calibration window 12 months; minimum 4 demand months in training; sample 2,000 items; lead time = each item's own value (rounded, at least 1 month); holding 20% of price per year; shortage multiples 1.0 / 0.3 / 0.1 of price; flat z = 1.65. Result tables (CSV), figures and executed notebooks are in `results_and_figures.zip`. Figures to insert: `fig3_accuracy_vs_cost_*.png` (accuracy vs cost) and `fig4_cost_vs_fillrate_*.png` (cost vs service level).
