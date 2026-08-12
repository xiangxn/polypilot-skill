# Features and Indicators

## Existing indicator package

Current indicator implementations include:

- `indicators/zscore.go`
- `indicators/imbalance.go`

Before adding a new calculation, inspect these files and their tests.

## ZScore

PolyPilot's ZScore is designed for short-horizon market behavior and includes time scaling. Do not replace it with a generic library implementation without checking semantic compatibility.

When a strategy consumes ZScore, distinguish:

- current/latest Z
- historical Z window
- direction/sign
- time remaining

Do not assume a Z threshold alone is sufficient for a trading decision.

## Order-book imbalance

`indicators.CalcImBalance` is the existing order-book imbalance helper.

Reuse it instead of implementing a competing formula.

## Probability features

The current probability engine exposes features including:

```text
latestZ
zWindows
openPrice
latestPrice
endTime
diffPrice
```

`zWindows` is a historical slice. Treat it as a snapshot; do not mutate it.

## Feature namespace discipline

Use stable, descriptive feature names.

Good:

```text
latestZ
openPrice
latestPrice
diffPrice
myFeature
```

Avoid ambiguous names such as:

```text
x
value
signal
tmp
```

If adding a feature to the probability layer, add a focused test for:

- initial state
- market reset
- missing data
- update ordering
- concurrent access if applicable

## Strategy-specific feature calculations

A simple calculation that is truly local to one strategy may remain in `strategy/`.

A reusable market feature should usually belong in `probability/` or `indicators/`.

Do not put generic infrastructure into a Strategy merely because it is convenient.
