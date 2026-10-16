# Architecture

Coursework repository for *Introduction to Financial Engineering* (the assignment
handout is `docs/ife_pa2.pdf`) plus a set of time-series / portfolio exercises.
There is no application and no test suite: the repository is a small library of
three model classes, one batch script that drives the portfolio backtest, and
five notebooks that explore the same models interactively.

## Structure

```
.
├── src/                 # importable model code (flat modules, no package)
│   ├── arima.py         # class ARIMA  — stationarity tests, ARMA order search, ARMA+GARCH forecast
│   ├── markowitz.py     # class Optimization — covariance assembly, QP weights, backtest P&L
│   ├── black_scholes_merton.py  # class option — binomial tree + closed-form BSM
│   └── run_backtest.py  # entry point: data -> selection -> forecast -> weights -> metrics
├── notebooks/           # 01..05, exploratory versions of the same material
├── data/                # inputs (KRX monthly closes, KOSPI, AirPassengers, TAF exercise sheets)
├── results/             # CSVs written by src/run_backtest.py
└── docs/                # assignment handout and two reference papers on Markowitz solvers
```

### Entry points

| Entry point | What it does |
|---|---|
| `cd src && python run_backtest.py` | The whole walk-forward portfolio backtest. Writes the five CSVs in `results/`. |
| `cd src && python arima.py` | `__main__` demo: ARMA(1,0,1) fit and one-step-ahead error study on four stocks from the KRX panel. |
| `cd src && python black_scholes_merton.py` | `__main__` demo: the four option cases from the assignment handout. |
| `notebooks/01..05` | Notebook equivalents; `01` imports `src/arima.py`, the rest are self-contained. |

`run_backtest.py` has no functions at all — it is 110 lines of module-level code
split into four blocks guarded by `if True:` / `if Security_Selection:` /
`if portfolio:` / `if portfolio_analysis:` flags. Those flags are how the author
re-ran one stage at a time; the stages communicate through DataFrames held in
module scope and through the CSVs in `results/`.

## Dependencies

Internal (fan-in):

```
run_backtest.py ──imports──> arima.py      (also imported by notebooks/01)
                └─imports──> markowitz.py
black_scholes_merton.py  (standalone, imported by nothing)
```

`arima.py` has the highest fan-in and is the de-facto interface of the repo. It
is imported with `from arima import *`, so it also supplies `os`, `np`, `pd` and
`plt` to its callers — `run_backtest.py` and `notebooks/01` rely on that and do
not import pandas themselves. Renaming or narrowing those re-exports breaks both
callers silently.

External: numpy, pandas, scipy, matplotlib, statsmodels (`SARIMAX`, `adfuller`,
`kpss`, ACF/PACF plots), `arch` (GARCH), `cvxopt` (`qp` for the Markowitz
problem), xlrd/openpyxl for the Excel inputs. `notebooks/04` additionally tries
`rpy2` + R's `rugarch` to cross-check the GARCH estimates; the stored outputs
show `rpy2` was not installed when the notebook was last run, so those cells
carry `ModuleNotFoundError` / `NameError` tracebacks.

## Conventions

- Flat modules under `src/`, no package and no `__init__.py`. Notebooks reach the
  modules with `sys.path.insert(0, '../src')`.
- Paths are relative to the *script's own directory*: `src/run_backtest.py`
  reads `../data/...` and writes `../results/...`, so it must be run from `src/`.
  Notebooks read `../data/...` and must be run from `notebooks/`.
- One class per module, state held on the instance and mutated in place
  (`Optimization.cov`, `.mean`, `.sol`, `.profit` are filled by successive calls).
- Results are passed between stages as CSVs with the dates as the index and the
  664 stock codes as columns; missing names are left as NaN rather than dropped.
- Comments inside the notebooks are mostly Korean; the module code is English.

## Impact

What a change ripples into:

- **`arima.py`** — highest fan-in. `forecasting_ARMA_GARCH` is the only method
  the backtest calls; the other ten methods are diagnostics used from notebooks.
  Changing the `(forecast, variance)` return shape breaks `run_backtest.py` and
  `markowitz.py` (which reads `var_df` as variances and square-roots them).
- **`markowitz.py`** — `guess_test_cov` rebuilds the covariance as
  `outer(std, std) * corr`, where `std` comes from the GARCH output and `corr`
  from the realised returns of the same window. It is the join point between the
  two forecasting models; a change in either feeds straight into the QP. Note
  that `ARIMA.forecasting_ARMA_GARCH` returns `conditional_volatility`, a
  standard deviation, which `run_backtest.py` stores in `var_df` and
  `guess_test_cov` then takes the square root of again. The column is named
  "var" but holds a volatility. This has been left as-is — changing it would
  change every number in `results/`.
- **`run_backtest.py`** — changing the window (120 months), the universe size
  (`n = 20`) or the minimum return (`r_min = 0.01` in
  `Optimization.optimalize_portfolio`) invalidates every CSV in `results/`.
  `n` is passed separately from the number of columns actually selected, so the
  two must be kept in sync by hand.
- **`results/*.csv`** — committed outputs of the last run. Nothing reads them
  back except `evaluation_metric.csv`'s own producer, so they are a record, not
  an input.

### Complexity ranking

There are no tests, so a real CRAP score cannot be computed. The table below is
the upper bound `CRAP_max = cc² + cc`, i.e. the score each function would have at
zero coverage, from `lizard` (cc = cyclomatic complexity, nloc = lines of code).

| Function | File | cc | nloc | CRAP_max |
|---|---|---:|---:|---:|
| `option.calculate_price` | `black_scholes_merton.py` | 8 | 18 | 72 |
| `ARIMA.evaluate_ARIMA` | `arima.py` | 7 | 26 | 56 |
| `ARIMA.adf_test` | `arima.py` | 4 | 38 | 20 |
| `ARIMA.kpss_test` | `arima.py` | 4 | 37 | 20 |
| `Optimization.calculate_return` | `markowitz.py` | 4 | 30 | 20 |

Whole repo: 559 nloc, 24 functions, average cc 2.5, nothing over the default
`lizard` thresholds. The riskiest code is therefore not any single function but
`run_backtest.py`, which `lizard` scores as zero functions because all 110 lines
are top-level.
