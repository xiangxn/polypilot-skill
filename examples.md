# PolyPilot Strategy Examples

## Minimal event-driven strategy

```go
package strategy

import (
    "context"

    "github.com/spf13/viper"
    "github.com/xiangxn/polypilot/core"
    "github.com/xiangxn/polypilot/runtime"
    "github.com/xiangxn/polypilot/state"
    "github.com/xiangxn/go-polymarket-sdk/orders"
)

type ExampleStrategy struct {
    Bus *core.EventBus
}

func (s *ExampleStrategy) Init(
    bus *core.EventBus,
    ctx context.Context,
    cfg *viper.Viper,
) {
    s.Bus = bus
}

func (s *ExampleStrategy) OnUpdate(
    e core.Event,
    o runtime.Observation,
    snap state.Snapshot,
) []runtime.OrderIntent {
    if e.Type != core.EventOrderBook {
        return nil
    }

    if o.TimeLeftSec < 60 {
        return nil
    }

    latestZ, ok := o.Features["latestZ"].(float64)
    if !ok {
        return nil
    }

    if latestZ < 2.0 {
        return nil
    }

    tokenID := ""
    for id := range o.Tokens {
        tokenID = id
        break
    }
    if tokenID == "" {
        return nil
    }

    return []runtime.OrderIntent{{
        Action:   runtime.OrderIntentActionPlace,
        MarketID: o.MarketID,
        TokenID:  tokenID,
        Price:    o.Tokens[tokenID].BidPrice,
        Side:     orders.BUY,
        Size:     1,
    }}
}
```

Important: this is a structural example, not a recommendation to trade this signal.

## Safe feature reader

Prefer helpers when a Strategy consumes many dynamic features:

```go
func floatFeature(o runtime.Observation, key string) (float64, bool) {
    v, ok := o.Features[key].(float64)
    return v, ok
}
```

Then:

```go
latestZ, ok := floatFeature(o, "latestZ")
if !ok {
    return nil
}
```

## Position-aware sell

```go
pos, ok := snap.Position.Tokens[tokenID]
if !ok || pos.Available <= 0 {
    return nil
}

return []runtime.OrderIntent{{
    Action:   runtime.OrderIntentActionPlace,
    MarketID: o.MarketID,
    TokenID:  tokenID,
    Price:    price,
    Side:     orders.SELL,
    Size:     pos.Available,
}}
```

## Cancel an existing order

```go
return []runtime.OrderIntent{{
    Action:  runtime.OrderIntentActionCancel,
    OrderID: orderID,
}}
```

## Execution-aware cleanup

A strategy can implement:

```go
func (s *ExampleStrategy) OnExecution(
    ev core.ExecutionEvent,
    o runtime.Observation,
    snap state.Snapshot,
) []runtime.OrderIntent {
    if ev.Status != core.ExecutionStatusRejected {
        return nil
    }

    // Build only the cleanup intents required by this strategy.
    return nil
}
```

Always inspect the current `core.ExecutionStatus` and `core.ExecutionReason` definitions before matching statuses.

## Configuration pattern

```go
type ExampleConfig struct {
    Threshold float64 `mapstructure:"threshold"`
    Size      float64 `mapstructure:"size"`
}

func DefaultExampleConfig() ExampleConfig {
    return ExampleConfig{
        Threshold: 2.0,
        Size:      1,
    }
}
```

Load it from a dedicated Viper subsection and retain safe defaults if configuration is absent or invalid.
