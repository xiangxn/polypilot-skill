# PolyPilot Strategy Examples

## Minimal event-driven strategy

```go
package strategy

import (
    "context"
    "math"

    "github.com/spf13/viper"
    "github.com/xiangxn/polypilot/core"
    "github.com/xiangxn/polypilot/feature"
    "github.com/xiangxn/polypilot/runtime"
    "github.com/xiangxn/go-polymarket-sdk/orders"
)

type ExampleStrategy struct {
    Bus    *core.EventBus
    config ExampleConfig
}

type ExampleConfig struct {
    TimeLeftSec int64   `mapstructure:"timeleft_sec"`
    MinZ        float64 `mapstructure:"min_z"`
    Size        float64 `mapstructure:"size"`
}

func DefaultExampleConfig() ExampleConfig {
    return ExampleConfig{
        TimeLeftSec: 60,
        MinZ:        2.0,
        Size:        1,
    }
}

func (s *ExampleStrategy) Subscribes() []core.EventType {
    return []core.EventType{core.EventOrderBook}
}

// Needs returns nil when the strategy uses no ports. If it reads
// deps.Books, it must return []runtime.PortName{runtime.PortBooks}.
func (s *ExampleStrategy) Needs() []runtime.PortName { return nil }

func (s *ExampleStrategy) Init(
    bus *core.EventBus,
    ctx context.Context,
    cfg *viper.Viper,
    deps runtime.Dependencies,
) {
    s.Bus = bus

    ec := DefaultExampleConfig()
    if cfg != nil {
        if sub := cfg.Sub("strategies.example"); sub != nil {
            // Unmarshal into a pre-populated struct keeps defaults for
            // keys the user omitted.
            if err := sub.Unmarshal(&ec); err != nil {
                // keep safe defaults
            }
        }
    }
    s.config = ec
}

func (s *ExampleStrategy) OnUpdate(d runtime.Decision) []runtime.OrderIntent {
    o := d.Obs

    if o.TimeLeftSec < s.config.TimeLeftSec {
        return nil
    }

    // Typed feature read. ok=false means "not produced yet" — never treat
    // a missing feature as zero.
    latestZ, ok := feature.LatestZ.Get(d.Facts)
    if !ok || math.Abs(latestZ) < s.config.MinZ {
        return nil
    }

    if o.TokenCount == 0 {
        return nil
    }
    // Tokens[0] is up, Tokens[1] is down (matches clobTokenIds order).
    tok := o.Tokens[0]
    if tok.BidPrice <= 0 {
        return nil
    }

    return []runtime.OrderIntent{{
        Action:   runtime.OrderIntentActionPlace,
        MarketID: o.MarketID,
        TokenID:  tok.ID,
        Price:    tok.BidPrice, // rest at best bid: maker, empty OrderType = GTC
        Side:     orders.BUY,
        Size:     s.config.Size,
    }}
}
```

Important: this is a structural example, not a recommendation to trade this signal.

## Iterating tokens safely

`Observation.Tokens` is a fixed-size array plus a count, not a map:

```go
ins := make([]runtime.OrderIntent, 0, o.TokenCount)
for i := 0; i < o.TokenCount; i++ {
    tok := o.Tokens[i]
    if tok.BidPrice <= 0 {
        return nil // fail closed on an incomplete book
    }
    ins = append(ins, runtime.OrderIntent{
        Action:   runtime.OrderIntentActionPlace,
        MarketID: o.MarketID,
        TokenID:  tok.ID,
        Price:    tok.BidPrice,
        Side:     orders.BUY,
        Size:     s.config.Size,
    })
}
return ins
```

There is no `o.Tokens[tokenID]` map lookup. To find one token by ID use `o.Token(id)`, which returns `(Token, bool)`.

## Safe feature reader

Feature reads already return `(value, ok)`. Wrap only if you need to collapse several keys:

```go
func latestZ(d runtime.Decision) (float64, bool) {
    return feature.LatestZ.Get(d.Facts)
}
```

Then:

```go
z, ok := latestZ(d)
if !ok {
    return nil
}
```

Do not reach into an untyped map — there is none. If a key you need does not exist, declare it in `feature/keys.go` and have a Provider produce it.

## Position-aware sell

```go
pos, ok := d.State.Position.Tokens[tokenID]
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

Use `pos.Available`, not `pos.Reserved` — reserved size is already committed to standing orders.

## Cancel an existing order

Cancels need only `OrderID`, but the order has to actually exist in the authoritative snapshot. The repository helper does that lookup for you:

```go
for _, orderID := range BuildCancelIntent(tokenID, d.State.Orders) {
    ins = append(ins, runtime.OrderIntent{
        Action:  runtime.OrderIntentActionCancel,
        OrderID: orderID,
    })
}
```

To **replace** a quote, return the cancel and the new PLACE in the same batch. Exposure and open-order caps are measured after the batch lands, so the cancelled order's notional is not counted twice — re-quoting does not need headroom for two quotes at once. Two caveats: the cancel must name an order the snapshot actually holds (a stale or invented `OrderID` frees nothing), and it frees exposure/slots but **not** balance — placements reach the exchange before cancels, so a BUY must be payable from the available balance as reported.

## Using a port

Declare the port, then use it. The engine guarantees a declared port is non-nil (it refuses to start otherwise):

```go
func (s *MyStrategy) Needs() []runtime.PortName {
    return []runtime.PortName{runtime.PortBooks}
}

func (s *MyStrategy) Init(bus *core.EventBus, ctx context.Context, cfg *viper.Viper, deps runtime.Dependencies) {
    s.books = deps.Books // guaranteed non-nil because Needs() declared it
}

func (s *MyStrategy) exitPrice(tokenID string) (float64, bool) {
    book, ok := s.books.Book(tokenID) // ok=false also covers "book too stale"
    if !ok {
        return 0, false
    }
    // SDK ordering: Bids ascending, so Bids[len-1] is the best bid.
    if len(book.Bids) == 0 {
        return 0, false
    }
    return book.Bids[len(book.Bids)-1].Price, true
}
```

Do not call `deps.Books` for anything you did not declare in `Needs()`, and do not read a port that is not in `runtime.Dependencies`.

## Execution-aware cleanup

```go
func (s *ExampleStrategy) OnExecution(
    ev core.ExecutionEvent,
    d runtime.Decision,
) []runtime.OrderIntent {
    if ev.Status != core.ExecutionStatusRejected {
        return nil
    }

    // Build only the cleanup intents required by this strategy.
    return nil
}
```

`OnExecution` is dispatched by interface assertion, not by `Subscribes()`. Always inspect the current `core.ExecutionStatus` and execution reason constants before matching statuses.

## Position-expiring cleanup

```go
func (s *MyStrategy) OnPositionExpiring(
    ev core.PositionExpiringEvent,
    snap state.Snapshot,
) []runtime.OrderIntent {
    var ins []runtime.OrderIntent
    for _, tokenID := range ev.TokenIDs {
        if avail := ev.Available[tokenID]; avail > 0 {
            ins = append(ins, runtime.OrderIntent{
                MarketID: ev.MarketID,
                TokenID:  tokenID,
                Price:    s.config.ExitPrice,
                Side:     orders.SELL,
                Size:     avail,
            })
        }
        for _, orderID := range BuildCancelIntent(tokenID, snap.Orders) {
            ins = append(ins, runtime.OrderIntent{
                Action:  runtime.OrderIntentActionCancel,
                OrderID: orderID,
            })
        }
    }
    return ins
}
```

This signature takes `state.Snapshot` directly rather than a `Decision` — the path is deliberately not gated on market-data readiness. Note that the producer (`state.StartPositionExpiringLoop`) is not started in `main.go` yet, so this hook does not fire in production as of this writing.

## Defining a new feature

A new derived value is a new typed key plus a Provider that writes it — never a new `Observation` field.

```go
// feature/keys.go — declare once, package level.
var MySignal = Define[float64]("my_signal")
```

```go
// myprovider/provider.go
type Provider struct{ /* ... */ }

func (p *Provider) Name() string { return "myprovider" }

func (p *Provider) Provides() []feature.AnyKey { return []feature.AnyKey{feature.MySignal} }

// Declare dependencies so the engine topologically orders providers. Declaring a
// key nobody provides is a startup error, and so is a cycle.
func (p *Provider) DependsOn() []feature.AnyKey { return []feature.AnyKey{feature.LatestZ} }

func (p *Provider) Ports() map[runtime.PortName]any { return nil }

func (p *Provider) Init(ctx context.Context) { /* start background work, honour ctx */ }

// Update must be pure writing: no event publishing, no order placement.
func (p *Provider) Update(ev core.Event, facts *feature.Set) {
    z, ok := feature.LatestZ.Get(facts)
    if !ok {
        return
    }
    feature.MySignal.Set(facts, z*2)
}
```

Register it in `main.go` under `Providers: []runtime.Provider{...}`.

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

Load it from a dedicated Viper subsection and retain safe defaults if configuration is absent or invalid. Because `Init` unmarshals into `DefaultExampleConfig()` rather than a zero struct, omitted keys keep their defaults — but that also means a typo'd key is silently ignored and the default is used. Never rely on a zero value as a meaningful trading parameter.
