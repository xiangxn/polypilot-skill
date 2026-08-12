# PolyPilot Architecture Reference

## Runtime graph

```text
Market Feeds
    |
    v
EventBus
    |
    v
Probability Engine
    |
    v
Observation
    |
    v
Strategy
    |
    | []OrderIntent
    v
Risk.Check
    |
    v
Executor
    |
    v
Exchange
    |
    v
Execution Events
    |
    v
State / Strategy
```

## Main composition

`main.go` is the canonical composition example.

The current application constructs:

- configuration
- shared Polymarket client
- State
- Risk engine
- Executor
- market feeds
- observer
- probability engine
- strategies
- reconcile loop
- runtime engine

Then calls `engine.Start(ctx)`.

When adding a Strategy, normally the only application composition change is adding it to:

```go
Strategies: []runtime.Strategy{
    &strategy.Strategy{},
    &strategy.MyStrategy{},
},
```

Do not redesign Engine composition for a strategy that can be expressed through the existing Strategy contract.

## Runtime interfaces

`runtime/types.go` defines the main extension points:

- `Feed`
- `Observer`
- `Probability`
- `ProbabilitySnapshotProvider`
- `Strategy`
- `TickStrategy`
- `ExecutionAwareStrategy`
- `MarketResolved`
- `RiskManager`
- `Executor`

The `Engine` wires these interfaces together.

## Observation model

`runtime.Observation` is the strategy-facing market snapshot.

Core fields:

```go
type Observation struct {
    At          int64
    MarketID    string
    Slug        string
    Tokens      map[string]Token
    TokenIds    []string
    Probability float64
    TimeLeftSec int64
    Confidence  float64
    Features    map[string]any
    GetOrderBook  func(tokenId string) *sdk.OrderBook
    FetchPrices   func(obj *gjson.Result) (float64, float64)
    CheckResolved func(slug string) (int, bool)
}
```

The Features map is intentionally extensible. This is useful for research strategies, but requires defensive type handling.

## Event types

The engine's event loop works around:

- MARKET
- ORDERBOOK
- SIGNAL
- EXECUTION

Other event types exist for risk/metrics/reconciliation and should be treated according to their current runtime implementation.

Always inspect `core/constants.go` and `core/event.go` before adding new event-driven behavior.

## State model

State is authoritative for:

- positions
- balances
- orders
- reservations
- fills
- reconciliation

The repository documents a reservation progression:

```text
Provisional
    ->
OrderReservation
    ->
release/finalize
```

Strategy code should consume snapshots and emit intents rather than reproducing this state machine.

## Risk model

Risk is a hard boundary between strategy and execution.

The current framework includes protections for:

- maximum daily loss
- maximum exposure per market
- maximum slippage
- maximum open orders
- market cooldown

Do not bypass or duplicate these controls without a clear reason.

## Reconciliation

Exchange state is authoritative during reconciliation. The current repository uses periodic reconciliation plus WS-triggered reconciliation.

Strategies should therefore tolerate local state being corrected by reconciliation.

## Concurrency

PolyPilot is concurrent.

Potentially concurrent areas include:

- EventBus subscribers
- tick-driven strategy evaluation
- probability snapshot reads
- execution events
- reconciliation
- executor queues

Avoid unsynchronized shared mutable state in a Strategy.

If strategy-local state is accessed from multiple goroutines, protect it appropriately.

## Time

All framework time fields are UTC.

Use framework timestamps and `TimeLeftSec` where available.

Never mix local time and UTC implicitly.
