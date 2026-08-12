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
    config  MyStrategyConfig

    // strategy-local state only
    markets *QueueMap[market.SlugMarket]
}

type MyStrategyConfig struct {
    TimeLeftSec float64 `mapstructure:"timeleft_sec"`
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
```

Keep strategy state small and explicit.

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

Example:

```go
if o.TimeLeftSec < s.config.TimeLeftSec {
    return nil
}

signal, ok := readSignal(o)
if !ok {
    return nil
}

if !signal.Entry {
    return nil
}

return []runtime.OrderIntent{{
    Action:   runtime.OrderIntentActionPlace,
    MarketID: o.MarketID,
    TokenID:  tokenID,
    Price:    price,
    Side:     orders.BUY,
    Size:     s.config.Size,
}}
```

## 4. Reading Features safely

Correct:

```go
latestZ, ok := o.Features["latestZ"].(float64)
if !ok {
    return nil
}
```

Incorrect:

```go
latestZ := o.Features["latestZ"].(float64)
```

If a feature is optional, missing, stale, or malformed, fail closed.

## 5. Reading token information

`Observation.Tokens` maps token ID to `runtime.Token`.

Do not assume token ordering unless the framework/source explicitly guarantees it. If the strategy needs UP/DOWN semantics, use the market/token metadata available from the current framework rather than relying on an accidental map order.

## 6. Order cancellation

Cancellation is an OrderIntent:

```go
runtime.OrderIntent{
    Action:  runtime.OrderIntentActionCancel,
    OrderID: orderID,
}
```

Use current `state.Snapshot.Orders` to determine which live orders exist.

Do not assume an order remains open just because the strategy created it.

## 7. Position-aware exits

Use:

```go
pos, ok := stateSnap.Position.Tokens[tokenID]
if !ok || pos.Available <= 0 {
    return nil
}
```

For market-style exits, use the framework's existing order-book helpers where available rather than inventing price traversal.

## 8. Execution-aware strategy

Implement `OnExecution` only when execution status changes the strategy's desired action.

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

The framework documentation explicitly notes that tick-driven behavior is independent from data events. If using it, design deduplication carefully.

Do not add a ticker just to make event-driven logic easier.

## 10. MarketResolved

If a strategy needs explicit settlement behavior, implement `MarketResolved`.

Keep settlement logic separate from ordinary entry/exit logic.

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

## 12. Testing strategy decisions

Build observations/events directly in tests. Avoid live feeds.

Test the decision table:

| Scenario | Expected |
|---|---|
| wrong event type | no intents |
| insufficient time | no intents |
| missing feature | no intents |
| signal false | no intents |
| valid entry | PLACE intent |
| no position on exit | no intents |
| valid position exit | SELL intent |
| execution rejection requiring cleanup | CANCEL/exit intents |
