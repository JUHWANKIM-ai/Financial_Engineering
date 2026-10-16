# Financial Engineering (coursework, 2023–2024)

[![Python](https://img.shields.io/badge/python-3.11-3776AB?logo=python&logoColor=white)](requirements.txt)
[![statsmodels](https://img.shields.io/badge/statsmodels-ARIMA%20%2F%20SARIMAX-4051B5)](requirements.txt)
[![arch](https://img.shields.io/badge/arch-GARCH(1%2C1)-6E4C9B)](requirements.txt)
[![CVXOPT](https://img.shields.io/badge/CVXOPT-quadratic%20programming-0B7285)](requirements.txt)
[![Data](https://img.shields.io/badge/data-KRX%20monthly%20closes%201999--2020-D9480F)](data/)

Assignments and exercises from two courses: *Introduction to Financial Engineering*
(the handout is kept in [`docs/ife_pa2.pdf`](docs/ife_pa2.pdf)) and a time-series
analysis course. Two things are implemented.

**Derivatives pricing.** An `option` class that builds the binomial tree of the
underlying, prices European and American calls and puts by backward induction,
and prices the European cases again with the closed-form Black–Scholes–Merton
formula so the two can be compared. The four parameter sets it runs on are the
ones in the handout.

**A walk-forward equity portfolio ("Fund-K").** From a panel of 664 Korean listed
stocks with monthly closes (1999-12 to 2020-12), each month the 20 stocks with
the highest Sharpe ratio over the trailing 120 months are selected; an ARMA model
forecasts each one's next-month return and a GARCH(1,1) fitted on its residuals
supplies the volatility (its last conditional volatility, carried forward one
month); the covariance is rebuilt from those volatilities and
the realised correlations; and a long-only minimum-variance quadratic program
with a 1% monthly return floor gives the weights. The book is compared against
the KOSPI index. The notebooks around it work through the underlying statistics:
stationarity tests, ARMA order selection by AIC/SBC and Ljung-Box, out-of-sample
forecast comparison (Granger–Newbold, Diebold–Mariano), and GARCH models of
weekly oil prices and of the US bond–T-bill spread.

## Pipeline

```
data/krx_monthly_close.xlsx          data/kospi_monthly.csv
  664 KRX stocks, monthly close        KOSPI monthly close
  1999-12 … 2020-12 (253 month ends)   2000-02 … 2020-12
        │                                      │
        ▼  pct change                          ▼  pct change
  monthly returns (252 months)           benchmark returns
        │
        ▼  rolling 120-month window, stepped one month at a time
  Security selection   top 20 by mean/std (Sharpe) over the window
        │
        ├─► ARMA(1,0,1)  per stock ──────────► results/return_df.csv  (mean forecast)
        │     statsmodels SARIMAX, 1 step ahead
        └─► GARCH(1,1) on the ARMA residuals ─► results/var_df.csv    (conditional volatility)
                                                   arch.arch_model
        │
        ▼  Covariance = outer(sigma, sigma) * corr(window returns)
  Markowitz QP (CVXOPT)   min wᵀΣw   s.t.  μᵀw ≥ 1%,  Σw = 1,  w ≥ 0
        │
        ▼
  results/sol_df.csv (weights) ─► results/profit_df.csv (monthly log P&L, Fund-K vs KOSPI)
        │
        ▼
  results/evaluation_metric.csv   cumulative / annualized return and vol, MDD, Sharpe, Sortino
```

Derivatives are a separate, self-contained path:

```
S0, K, T, r, sigma, N ─► binomial tree ─► backward induction ─► European / American price
                      └► Black–Scholes–Merton closed form ───► European price (cross-check)
```

## Results

The portfolio backtest as it was last run, read from
[`results/evaluation_metric.csv`](results/evaluation_metric.csv) (re-derived
here from `results/profit_df.csv` and matching to four decimals).

| | Fund-K | KOSPI |
|---|---:|---:|
| Cumulative return (sum of monthly log returns) | 4.1670 | 1.2438 |
| Annualized return | 21.95% | 6.15% |
| Annualized volatility | 18.96% | 24.06% |
| Downside volatility | 9.84% | 17.29% |
| Worst month (log return) | −19.47% | −26.31% |
| Best month (log return) | 17.13% | 20.25% |
| Maximum drawdown | −32.14% | −48.52% |
| Sharpe ratio | 1.1576 | 0.2557 |
| Sortino ratio | 2.2314 | 0.3557 |

Read these with three caveats, all of them properties of the script that produced
the file:

1. **The window includes the training period.** The metrics cover all 252 months
   (2000-01 to 2020-12). The first 120 months hold the in-sample weights fitted on
   the training window; only 2010-01 to 2020-12 (132 months) is walk-forward.
2. **No risk-free rate.** Sharpe and Sortino are the annualized return divided by
   the annualized (downside) volatility, with the risk-free rate taken as zero.
3. **The Calmar row of the CSV is wrong and is omitted above.** The script indexes
   the Sharpe row instead of the drawdown row, so the value it stores is the
   annualized volatility. The code is left as it is; the row is simply not reported.

The option prices are not stored in the repository; the table below is what
`python src/black_scholes_merton.py` prints, with the two parameter sets of the
handout and a ten-step tree.

| Case | Binomial (N = 10) | Black–Scholes–Merton |
|---|---:|---:|
| European call, S₀ = 70, K = 60, T = 10, r = 0.05, σ = 0.2 | 36.0580 | 36.0200 |
| American call, same parameters | 36.0580 | — |
| European put, S₀ = 100, K = 95, T = 5, r = 0.04, σ = 0.1 | 1.2848 | 1.2854 |
| American put, same parameters | 2.1805 | — |

The American call matches the European one, as it should for a non-dividend-paying
underlying; the American put is worth more, which is where early exercise bites.

## Repository layout

| Path | Contents |
|---|---|
| `src/arima.py` | `ARIMA` class: ADF / KPSS tests, ACF-PACF plots, ARMA order search scored by SSE / AIC / SBC / Ljung-Box, one-step-ahead forecast error study, and `forecasting_ARMA_GARCH` (SARIMAX mean + GARCH(1,1) volatility) used by the backtest |
| `src/markowitz.py` | `Optimization` class: covariance from forecast volatilities and realised correlations, the long-only CVXOPT quadratic program, and the P&L / plot against the benchmark |
| `src/black_scholes_merton.py` | `option` class: binomial tree, European and American backward induction, closed-form Black–Scholes–Merton |
| `src/run_backtest.py` | Entry point for the portfolio study; writes every file in `results/` |
| `notebooks/01_arima_air_passengers.ipynb` | Seasonal ARMA on the AirPassengers series through `src/arima.py` |
| `notebooks/02_arma_interest_rate_spread.ipynb` | ARMA model selection on the US 5-year bond minus T-bill spread; expanding-window forecasts compared with Granger–Newbold and Diebold–Mariano tests |
| `notebooks/03_seasonal_arima_air_passengers.ipynb` | The same exercise on AirPassengers: differencing, multiplicative seasonal ARMA, forecast comparison |
| `notebooks/04_garch_oil_and_spread.ipynb` | McLeod-Li / ARCH-LM tests and GARCH(1,1) on weekly oil prices, on the rate spread and on NYSE returns; the last cells try to cross-check against R's `rugarch` through `rpy2` |
| `notebooks/05_portfolio_backtest_prototype.ipynb` | Earlier prototype of the portfolio backtest (selection by cumulative return, Sharpe, Sortino or at random) |
| `data/krx_monthly_close.xlsx` | 664 Korean listed stocks, monthly close, 1999-12-28 to 2020-12-30 |
| `data/kospi_monthly.csv` | KOSPI monthly close, 2000-02 to 2020-12, the benchmark |
| `data/air_passengers.csv` | The AirPassengers series used by notebooks 01 and 03 |
| `data/taf2_quarterly_us_rates.xls`, `data/taf3_weekly_oil.xls`, `data/taf3_daily_nyse.xlsx` | Course exercise sheets: quarterly US 5-year bond and T-bill rates, weekly oil spot prices, daily NYSE returns |
| `results/` | Output of the last backtest run: forecast means and volatilities, weights, monthly P&L, metrics |
| `docs/` | The assignment handout and two reference papers on solving the Markowitz problem |
| `ARCHITECTURE.md` | Module map, dependency direction, conventions and a complexity ranking |

## Get Started

**1. Install.**

```bash
python -m venv .venv && . .venv/Scripts/activate   # Windows; use .venv/bin/activate elsewhere
pip install -r requirements.txt
```

Python 3.11. Versions are not pinned — the repository never carried a lock file,
so the file lists the packages that are actually imported and nothing more.

**2. Data.** Everything the code reads is in `data/` and is committed. The one
exception is `notebooks/05_portfolio_backtest_prototype.ipynb`, which opens a
`kospi.xlsx` that is not in the repository; its last cells cannot be rerun as
they stand. Nothing else needs to be downloaded.

**3. Run.** Each script resolves its paths relative to its own directory, so run
them from the directory they live in.

```bash
cd src
python run_backtest.py            # the whole portfolio study; overwrites ../results/*.csv
python black_scholes_merton.py    # the four option cases of the handout
python arima.py                   # ARMA(1,0,1) demo and forecast error study on four stocks
```

`run_backtest.py` refits an ARMA and a GARCH model for 20 stocks in each of the
132 walk-forward months, so it is slow. Its four stages are guarded by the
`Security_Selection`, `portfolio` and `portfolio_analysis` flags near the top of
each block; set one to `False` to skip it and reuse the CSVs already in
`results/`.

```bash
cd notebooks
jupyter lab                       # notebooks read ../data and add ../src to sys.path
```
