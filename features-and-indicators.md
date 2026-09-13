# Features and Indicators

## Existing indicator package

Current indicator implementations include:

- `indicators/zscore.go` — `NewZScore(windowSize)`, `OnTick`, `ZScore(currentPrice, startPrice, remainingSeconds)`, `IsReady`, `Sigma`, `WindowSize`
- `indicators/imbalance.go` — `CalcImBalance(orderBook *sdk.OrderBook, topN int) float64`

Before adding a new calculation, inspect these files and their tests.

## ZScore

PolyPilot's ZScore is designed for short-horizon market behavior and includes time scaling (`remainingSeconds` is an input, not a constant). Do not replace it with a generic library implementation without checking semantic compatibility.

When a strategy consumes ZScore, distinguish:

- current/latest Z
- historical Z window
- direction/sign
- time remaining

Do not assume a Z threshold alone is sufficient for a trading decision.

## Order-book imbalance

`indicators.CalcImBalance` is the existing order-book imbalance helper. It takes a `*sdk.OrderBook` and a depth limit.

Reuse it instead of implementing a competing formula. To obtain a book from inside a strategy, declare `runtime.PortBooks` in `Needs()` and use the injected `deps.Books`.

## Probability features

Features are declared as strongly typed keys in `feature/keys.go`:

```go
var (
    OpenPrice   = Define[float64]("ext.open_price")    // Chainlink open reference
    LatestPrice = Define[float64]("ext.latest_price")  // latest external reference
    LatestZ     = Define[float64]("ext.latest_z")      // latest z-score
    ZWindows    = Define[[]float64]("ext.z_windows")   // recent z samples
)
```

`ZWindows` is a slice. Treat it as a snapshot: do not mutate it, and do not retain it past the current callback.

Read them through the key:

```go
z, ok := feature.LatestZ.Get(d.Facts)
if !ok {
    return nil // not produced yet — never treat as zero
}
```

## Feature namespace discipline

A feature name is `<namespace>.<quantity>` and is declared exactly once, as a package-level `var` in `feature/keys.go`. `Define[T]` allocates a dense slot, so declaring the same logical feature twice produces two unrelated slots that silently do not see each other's writes.

Good:

```go
var (
    OpenPrice   = Define[float64]("ext.open_price")
    LatestZ     = Define[float64]("ext.latest_z")
    MySignal    = Define[float64]("strategy.my_signal")
)
```

Avoid ambiguous names such as:

```text
x
value
signal
tmp
```

## Adding a feature

A feature has two halves: the **key** (what it is) and the **provider** (who produces it). Neither belongs in `Observation`.

1. Declare the key in `feature/keys.go`.
2. Write it from a `Provider.Update(ev, facts)`. `Update` must be pure writing — no event publishing, no order placement.
3. Declare the read/write sets so the engine can validate and order providers:

```go
func (p *Provider) Provides() []feature.AnyKey  { return []feature.AnyKey{feature.MySignal} }
func (p *Provider) DependsOn() []feature.AnyKey { return []feature.AnyKey{feature.LatestZ} }
```

The engine refuses to start if a key has two providers, if a `DependsOn` key has none, or if the graph has a cycle. The topological order also determines `Update` order, so "A reads what B writes" holds within a single event.

4. Register the provider in `main.go` under `Providers:`.
5. Add a focused test covering:
   - initial state (key not yet written → `ok == false`)
   - market reset
   - missing data
   - update ordering relative to its dependencies

Reads are on the event-loop goroutine and the `*feature.Set` is reused per event, so the set itself needs no locking — but any state a provider keeps across events (its own buffers, tickers started in `Init`) does.

## Strategy-specific feature calculations

A simple calculation that is truly local to one strategy may remain in `strategy/`.

A reusable market feature should usually be a key in `feature/` produced by a Provider under `probability/`, or a pure function in `indicators/`.

Do not put generic infrastructure into a Strategy merely because it is convenient.
