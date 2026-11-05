# Autoregressive Models for High-Frequency Financial Data

MSc Data Science and Analytics - Dissertation, University of Leeds (MATH5872M, 2026)
Author: Deepika Sivakumar

This project models the time between trades in three very different markets: a currency pair (GBP/USD), a cryptocurrency (BTCUSDT) and a US stock (NVDA). Trades do not arrive at a steady rate. They bunch together in bursts and then go quiet, and that pattern carries information about liquidity and market activity. The dissertation asks how well Autoregressive Conditional Duration (ACD) models capture this behaviour at tick level, and what goes wrong when the data is recorded at millisecond, microsecond and nanosecond precision.

---

## At a glance

| | |
|---|---|
| **Markets** | GBP/USD (FX), BTCUSDT (crypto), NVDA (US equity) |
| **Data** | 7.3M trade durations over a 14-day training window |
| **Resolution** | Millisecond, microsecond and nanosecond |
| **Models** | Poisson baselines, EACD, WACD, Log-ACD (two forms)|
| **Language** | R (R Markdown) |
| **Key packages** | `ACDm`, `data.table`, `archive` |

---

## Why this is interesting

ACD models were introduced by Engle and Russell (1998) for trade data recorded to the second. Modern feeds are far finer than that, and the finer resolution exposes problems the original framework never had to deal with:

- **Near-simultaneous trades.** Exchanges can stamp many fills from a single order with the same timestamp. In one NVDA event, 8,095 fills shared one nanosecond timestamp. Left untreated, these produce huge numbers of near-zero durations.
- **Distributional breakdown.** Weibull, Burr and generalised gamma densities can diverge near zero. When nearly half the durations are sub-millisecond, the likelihood becomes unreliable without any obvious error message.
- **Persistence at the boundary.** Estimated persistence sits close to, and sometimes above, one. This is the regime studied by recent work on integrated ACD processes (Cavaliere et al.), where standard inference needs care.

Comparing three asset classes side by side shows how much of the modelling outcome depends on how each market records its data.

---

## Data

| Asset | Source | Feed type | Resolution | Training durations |
|---|---|---|---|---|
| GBP/USD | Dukascopy | Quote revisions | Millisecond | 906,431 |
| BTCUSDT | Binance aggTrades | Pre-aggregated trades | Microsecond | 4,535,411 |
| NVDA | Databento | Exchange trades | Nanosecond | 1,856,576 |

The raw data is not included in this repository because of vendor licence terms. The data sources above are publicly accessible, and the code documents exactly how each file is read and processed.

---

## Method

The workflow follows the same order as the dissertation: data selection, pre-processing, model selection and fitting, then results.

### 1. Pre-processing

- **Duration construction.** Separate reading paths for nanosecond integer timestamps (NVDA) and POSIXct timestamps (FX and crypto), so no precision is lost.
- **Consolidation of simultaneous trades.** Fills sharing a timestamp are merged into one event, following the principle in Engle and Russell (1998) with VWAP aggregation from Huth and Abergel (2011). This was not applied to GBP/USD, which is a quote feed, or reapplied to Binance data, which is already aggregated upstream.
- **Diurnal adjustment.** Trading activity follows a daily rhythm (busy opens and closes, quieter middays). This was removed using a smoothed 5-minute profile, fitted on training data only so that nothing leaks into the test day.

### 2. Model ladder

Models were fitted in order of increasing flexibility, each one motivated by a shortcoming in the previous:

1. **Homogeneous and inhomogeneous Poisson.** Baselines that assume no clustering. Rejected on all assets, which confirms there is dependence to model.
2. **EACD.** Exponential innovations. Retained as the cross-asset baseline because quasi-maximum likelihood keeps it consistent for the conditional mean even when the distribution is wrong, and it cannot be corrupted by near-zero durations.
3. **WACD.** Weibull innovations. Fits were corrupted on BTCUSDT and NVDA by sub-millisecond mass (19.1% and 48.9% of durations).
4. **Log-ACD (LACD1, LACD2).** Log specifications that avoid positivity constraints on parameters.

### 3. Diagnostics

- **Ljung-Box statistics** are read as effect sizes alongside ACF magnitudes, since p-values carry little information at hundreds of thousands of observations.
- **Order selection** is driven by residual squared autocorrelation, with AIC and BIC as supporting evidence.

### 4. Validation

- **Simulation study.** Synthetic durations are generated from the fitted model and compared with real test data on clustering (ACF) and on marginal shape (quantiles). Moment-based comparisons are avoided because the mean may not exist when persistence is at or above one.

---

## Main findings

**Data recording drives the results as much as the model does.** Persistence came out above one for BTCUSDT and NVDA and just below one for GBP/USD. The dissertation shows that this is linked to how near-simultaneous prints are handled, not to sample size. Rebuilding the NVDA events from the exchange's own event flags moves persistence back toward one.

---

## Limitations and further work

- Standard errors near the unit persistence boundary are not reliable and are flagged rather than reported at face value.
- The linear and log persistence measures are on different scales, so the log model is not presented as resolving the linear result.

---

## Repository structure

```
.
├── code_D.Rmd        # Full analysis: data loading, pre-processing, model fitting, diagnostics, simulation
└── README.md
```

## Running the code

1. Install R (4.x) and the packages:
   ```r
   install.packages(c("ACDm", "data.table", "archive"))
   ```
2. Download the raw data from the sources listed above and update the file paths at the top of `code_D.Rmd`.
3. Knit `code_D.Rmd`, or run it chunk by chunk. Model fitting on the full BTCUSDT sample takes a while, so running chunks individually is recommended.

---

## References

- Engle, R. F. and Russell, J. R. (1998). Autoregressive conditional duration: a new model for irregularly spaced transaction data. *Econometrica*.
- Cavaliere, G., Mikosch, T., Rahbek, A. and Vilandt, F. Work on inference in integrated and near-integrated ACD models, including *Journal of Econometrics* (2024) and arXiv:2505.06190.
- Grammig, J. and Maurer, K.-O. (2000). Non-monotonic hazard functions and the autoregressive conditional duration model. *Econometrics Journal*.
- Lunde, A. (1999). A generalized gamma autoregressive conditional duration model. Working paper.
- Huth, N. and Abergel, F. (2011). The times change: multivariate subordination, empirical facts. Working paper.
- Fernandes, M. and Grammig, J. (2005). Nonparametric specification tests for conditional duration models. *Journal of Econometrics*.
- Pacurar, M. (2008). Autoregressive conditional duration models in finance: a survey of the theoretical and empirical literature. *Journal of Economic Surveys*.

---

## Acknowledgements

Supervised by Dr Xiling Zhang, School of Mathematics, University of Leeds.
