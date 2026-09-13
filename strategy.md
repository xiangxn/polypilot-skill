# PolyPilot Strategy Development Guide

## 1. Start from the existing strategy

Before creating a new strategy, inspect:

- `strategy/strategy.go`
- `strategy/utils.go`
- `strategy/queue_map.go`
- `strategy/*_test.go`

The existing strategy demonstrates the expected lifecycle, configuration loading, market caching, event handling, OrderIntent construction, position checks, cancellation, and execution-aware behavior.

Do not create a second framework inside the strategy package.

## 2. Recommended structure

For a new strategy:

```go
type MyStrategy struct {
    Bus     *core.EventBus
    books   runtime.BookPort
    config  MyStrategyConfig

    // strategy-local state only
    markets *QueueMap[market.SlugMarket]
}

type MyStrategyConfig struct {
    // int64, not float64: Observation.TimeLeftSec is int64 (seconds).
    TimeLeftSec int64   `mapstructure:"timeleft_sec"`
    Threshold   float64 `mapstructure:"threshold"`
    Size        float64 `mapstructure:"size"`
}

func DefaultMyStrategyConfig() MyStrategyConfig {
    return MyStrategyConfig{
        TimeLeftSec: 60,
        Threshold:   1.0,
        Size:        5,
    }
}

func (s *MyStrategy) Subscribes() []core.EventType {
    return []core.EventType{core.EventOrderBook}
}

func (s *MyStrategy) Needs() []runtime.PortName {
    return []runtime.PortName{runtime.PortBooks}
}

func (s *MyStrategy) Init(
    bus *core.EventBus,
    ctx context.Context,
    cfg *viper.Viper,
    deps runtime.Dependencies,
) {
    s.Bus = bus
    s.books = deps.Books // declared in Needs(), so guaranteed non-nil

    mc := DefaultMyStrategyConfig()
    if cfg != nil {
        if sub := cfg.Sub("strategies.my_strategy"); sub != nil {
            _ = sub.Unmarshal(&mc) // keep defaults on error
        }
    }
    s.config = mc
}
```

Keep strategy state small and explicit. Do not declare a port in `Needs()` that you never use, and never dereference a `Dependencies` field you did not declare — undeclared ports may be nil.

## 3. Entry logic

A strategy should be readable as:

```text
event
  -> market eligibility
  -> data readiness
  -> signal condition
  -> inventory/risk-aware decision
  -> OrderIntent
```

Do not mix unrelated responsibilities into one large conditional.

Example (`OnUpdate` receives a `Decision`; unpack `o := d.Obs` first):

```go
func (s *MyStrategy) OnUpdate(d runtime.Decision) []runtime.OrderIntent {
    o := d.Obs

    if o.TimeLeftSec < s.config.TimeLeftSec {
        return nil
    }

    z, ok := feature.LatestZ.Get(d.Facts)
    if !ok || math.Abs(z) < s.config.Threshold {
        return nil
    }

    tok := o.Tokens[0] // Tokens[0] is up, Tokens[1] is down
    return []runtime.OrderIntent{{
        Action:   runtime.OrderIntentActionPlace,
        MarketID: o.MarketID,
        TokenID:  tok.ID,
        Price:    tok.BidPrice,
        Side:     orders.BUY,
        Size:     s.config.Size,
    }}
}
```

Note that you do not need a "data readiness" guard for market facts: if the snapshot is not ready the engine does not call you at all. Readiness guards are only needed for *features* your own logic requires.

## 4. Reading Features safely

Features are strongly typed keys. A read returns `(value, ok)`:

```go
z, ok := feature.LatestZ.Get(d.Facts)
if !ok {
    return nil // fail closed: "not produced yet" is not zero
}
```

Incorrect:

```go
z := feature.MustGet(d.Facts) // panics — dev/replay paths only
```

If a feature is optional, missing, stale, or malformed, fail closed. Never substitute a default for a missing feature in a trading path.

A slice read from `Facts` (e.g. `ZWindows`) is a snapshot — do not mutate it, and do not retain it past the call.

If the key you need does not exist, declare it in `feature/keys.go` and have a Provider write it. Do not add fields to `Observation` and do not fall back to an untyped map — there is none.

## 5. Reading token information

`Observation.Tokens` is a fixed-size array (`[MaxTokens]Token`, `MaxTokens == 2`) with an explicit `TokenCount`:

```go
for i := 0; i < o.TokenCount; i++ {
    tok := o.Tokens[i]
    ...
}
```

Position convention is guaranteed: `Tokens[0]` is up, `Tokens[1]` is down, matching `clobTokenIds` order. Use it for direction-dependent logic rather than re-deriving direction from prices.

For a single token by ID: `tok, ok := o.Token(id)`.

Top-of-book prices are already here: `tok.AskPrice` is the best ask, `tok.BidPrice` is the best bid. You only need `runtime.PortBooks` when you want the full depth.

## 6. Order cancellation

Cancellation is an OrderIntent:

```go
runtime.OrderIntent{
    Action:  runtime.OrderIntentActionCancel,
    OrderID: orderID,
}
```

Use current `state.Snapshot.Orders` to determine which live orders exist. The repository helper does the lookup and returns matching order IDs:

```go
for _, orderID := range BuildCancelIntent(tokenID, d.State.Orders) {
    ...
}
```

Do not assume an order remains open just because the strategy created it.

## 7. Position-aware exits

Use:

```go
pos, ok := d.State.Position.Tokens[tokenID]
if !ok || pos.Available <= 0 {
    return nil
}
```

Use `Available`, not `Reserved`.

For market-style exits, use the framework's existing order-book helpers (`CalculateMarketPrice` in `strategy/utils.go`) rather than inventing price traversal.

## 8. Execution-aware strategy

Implement `OnExecution` only when execution status changes the strategy's desired action. It is dispatched by interface assertion, not by `Subscribes()`:

```go
func (s *MyStrategy) OnExecution(ev core.ExecutionEvent, d runtime.Decision) []runtime.OrderIntent
```

Typical flow:

```text
PLACE intent
  -> exchange response
  -> execution event
      -> ACCEPTED / FILLED / REJECTED / ...
          -> strategy reaction
```

Never equate ACCEPTED with FILLED.

## 9. TickStrategy

Use `TickStrategy` when a strategy must periodically re-evaluate even without a new MARKET/ORDERBOOK event.

```go
func (s *MyStrategy) OnTick(now time.Time, d runtime.Decision) []runtime.OrderIntent
```

Tick-driven behavior is independent from data events. If using it, design deduplication carefully — the tick fires repeatedly and there is no event to tell "this is new".

Do not add a ticker just to make event-driven logic easier.

## 10. Position-expiring handling

For end-of-market cleanup (sell what is available, cancel what is standing), implement `PositionExpiringAwareStrategy`:

```go
func (s *MyStrategy) OnPositionExpiring(ev core.PositionExpiringEvent, snap state.Snapshot) []runtime.OrderIntent
```

This signature takes `state.Snapshot` directly, not a `Decision` — the path deliberately does not require market data to be ready, because cleanup is exactly what you still need when the book has gone stale.

Keep cleanup separate from ordinary entry/exit logic.

Note: the dispatch case is wired, but the producer (`state.StartPositionExpiringLoop` / `RegisterMarketExpiry`) is not started in `main.go` yet, so this hook does not fire in production as of this writing.

## 11. Strategy configuration

Use a dedicated namespace:

```yaml
strategies:
  my_strategy:
    timeleft_sec: 60
    threshold: 1.5
    size: 5
```

Never silently interpret a zero value as a valid trading parameter unless zero is intentionally supported.

Two Viper behaviours worth knowing:

- Unmarshalling into a struct pre-populated with `DefaultMyStrategyConfig()` **preserves defaults** for keys absent from the config file (it does not zero them).
- Unknown keys are **silently ignored** — a typo'd key falls back to the default with no warning. Verify a new key actually took effect rather than assuming.

## 12. Testing strategy decisions

Build `runtime.Decision` values directly in tests. Avoid live feeds.

Test the decision table:

| Scenario | Expected |
|---|---|
| unsubscribed event type | no intents |
| insufficient time | no intents |
| missing feature (`ok == false`) | no intents |
| signal false | no intents |
| valid entry | PLACE intent |
| no position on exit | no intents |
| valid position exit | SELL intent |
| `TokenCount < 2` | no intents |
| execution rejection requiring cleanup | CANCEL/exit intents |

For a feature-heavy strategy, seed facts with a fresh `feature.NewSet()` and the same `Set` calls a Provider would make.

Run `go test -race ./...`. Mutation-check important guards: delete the guarded line and confirm the test actually fails — a test that stays green is decoration.
