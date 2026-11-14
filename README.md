# btc-cent-forecast-01 — Four-Hour BTC Price Forecaster

One entry for the make.fun competition `btc-cent-forecast-01`: forecast the
BTC price four hours ahead. Scoring is mean absolute error in cents across
12 checkpoints over 48 hours; the entry must beat the house persistence
baseline (repeat the last close).

## Method

The entry is deliberately conservative. At a four-hour horizon, minute
closes carry almost no reliable signal beyond the slow drift already in
the recent tape, and full-strength trend extrapolation over-shoots.

For each case, `main.py`:

1. Takes the most recent 720 one-minute closes (12 hours).
2. Fits a straight line to their natural logarithms by ordinary least
   squares.
3. Extrapolates the fitted slope only **one quarter** of the way across
   the 240-minute horizon, anchored at the last observed close.
4. Clamps the result to [0.5x, 2x] of the last close and rounds to cents.

Degenerate input never crashes the run: a tape shorter than 720 closes,
or one containing a non-positive / non-finite price, falls back to
forecasting the last close itself.

## Why this, and not something fancier

Three variants were backtested offline on the six practice cases (see
`research/score_variants.py`). The log-trend method above won on
robustness: it is the smallest honest tilt away from persistence, so it
survives regime changes (a downtrend practice week is not a promise
about the live week) instead of over-fitting to it.

Measured practice MAE (cents, lower is better; baseline 44,521.52):

| Variant | MAE (cents) | vs baseline |
|---|---|---|
| **A — log-trend, quarter slope (shipped)** | **43,212.80** | −2.94% |
| B — 50/50 blend of A with mean reversion | 44,990.30 | +1.05% (worse) |
| C — EMA momentum on log prices | 41,371.47 | −7.07% |
| Persistence (repeat last close) | 44,521.64 | — |

Decision: ship variant A. Variant C beats A on the practice set by
only 4.26%, far short of the 10% margin the decision rule requires to
override A — the six practice cases come from a single downtrend week,
so a small practice edge is noise, and A was chosen for robustness
across regimes. Variant B does not even beat the baseline and was
rejected outright.

Per-case forecasts (variant A) and errors, in cents:

| case | forecast | actual | error |
|---|---|---|---|
| p00 | 74,763.32 | 74,447.80 | 31,552 |
| p01 | 74,385.97 | 73,680.84 | 70,513 |
| p02 | 73,618.23 | 73,210.50 | 40,773 |
| p03 | 73,070.49 | 73,777.35 | 70,686 |
| p04 | 73,694.53 | 73,545.88 | 14,865 |
| p05 | 73,555.56 | 73,864.44 | 30,888 |

## Usage

```bash
MAKEFUN_INPUT_DIR=/path/to/input MAKEFUN_OUTPUT_DIR=/path/to/output python3 main.py
```

Defaults are `/input/` and `/output/`. One `<case>.out` file is written
per `<case>.in` file, each containing exactly:

```json
{"forecasts": [{"case": "p00", "price": 62100.25}]}
```

Pure Python 3 standard library, no network access, no randomness,
deterministic — identical inputs produce byte-identical outputs, and it
runs in well under a second per case.

## Files

- `main.py` — the entrypoint.
- `research/score_variants.py` — offline backtest harness for the three
  candidate variants on the practice cases (not part of the entry).
