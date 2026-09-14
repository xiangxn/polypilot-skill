---
name: polypilot
description: Use when developing, modifying, reviewing, or testing trading strategies and strategy-related components for the github.com/xiangxn/polypilot Go framework. Follow PolyPilot's event-driven architecture, the Strategy/Provider/OrderIntent contracts, the Decision + typed-feature model, state/risk/execution boundaries, and testing requirements. Prefer existing framework APIs and patterns over inventing new abstractions.
---

# PolyPilot Agent Skill

You are working in the PolyPilot repository (`github.com/xiangxn/polypilot`), a Go event-driven automation engine for short-duration Polymarket binary markets.

## Primary objective

When a user asks for a new strategy, strategy modification, indicator, signal rule, or strategy bug fix:

1. Inspect the current repository source before coding.
2. Read `main.go` to understand the actual runtime composition.
3. Read `runtime/types.go` for the public contracts.
4. Read `feature/keys.go` for the typed feature keys a strategy may read.
5. Read the closest existing implementation under `strategy/`.
6. Reuse existing `runtime.Decision`, `runtime.OrderIntent`, `state.Snapshot`, indicators, helpers, and SDK types.
7. Keep strategy logic inside `strategy/` unless the requested feature genuinely belongs elsewhere.
8. Never bypass Risk or Executor from a Strategy.
9. Add focused tests for the new behavior.
10. Run `go test -race ./...` when possible.
11. Report exactly what changed, what was tested, and any assumptions.

## Architecture

The normal flow is:

`Feeds -> Providers (topological order) -> Engine dispatch -> Strategy -> Risk.Filter -> Executor -> State`

- **Providers** consume events and write *derived facts* into a `feature.Set`. They run before dispatch, in dependency order.
- **The Engine** then builds a `runtime.Decision` and calls each Strategy whose `Subscribes()` list contains the event type.
- **Strategy** returns `[]runtime.OrderIntent`; **Risk** filters them; **Executor** places them.

A Strategy never sees a raw event as its only input — it sees `Decision{At, Event, Obs, Facts, State}`.

Do not move responsibilities across these boundaries merely to make a strategy easier to implement.

## Strategy contracts

The core interface:

```go
type Strategy interface {
    // Subscribes declares which event types this strategy wants. The engine only
    // calls OnUpdate for those. Must be explicit.
    Subscribes() []core.EventType

    // Needs declares which ports this strategy requires. The engine validates at
    // startup and refuses to run if a declared port has no supplier.
    Needs() []PortName

    Init(bus *core.EventBus, ctx context.Context, cfg *viper.Viper, deps Dependencies)
    OnUpdate(d Decision) []OrderIntent
}
```

Both `Subscribes()` and `Needs()` are **required methods**, not optional. A strategy that omits them does not implement `Strategy`.

Optional capabilities (`TickStrategy` / `ExecutionAwareStrategy` / `PositionExpiringAwareStrategy` are dispatched by interface assertion — they are **not** gated by `Subscribes()`):

```go
type TickStrategy interface {
    OnTick(now time.Time, d Decision) []OrderIntent
}

type ExecutionAwareStrategy interface {
    OnExecution(ev core.ExecutionEvent, d Decision) []OrderIntent
}

type PositionExpiringAwareStrategy interface {
    OnPositionExpiring(ev core.PositionExpiringEvent, snap state.Snapshot) []OrderIntent
}
```

Use `OnUpdate` for event-driven decisions, `OnTick` only when periodic evaluation is actually required, `OnExecution` when execution outcomes must change strategy behavior, and `OnPositionExpiring` for end-of-market cleanup (selling available inventory, cancelling standing orders).

Caveat: the **dispatch** for `POSITION_EXPIRING` is wired, but the **producer** is not started — `state.StartPositionExpiringLoop`/`RegisterMarketExpiry` have no production caller yet. Implementing `OnPositionExpiring` today writes code that will not be invoked until that loop is started in `main.go`.

There is **no `MarketResolved` hook.** Settlement is handled by the state layer, not by strategies.

## Decision

`runtime.Decision` is the complete input to one decision:

```go
type Decision struct {
    At    time.Time
    Event core.Event      // the event that triggered this call
    Obs   Observation     // where/what the market is
    Facts *feature.Set    // derived values, produced by Providers
    State state.Snapshot  // positions, balances, open orders
}
```

**Lifetime contract:** `Obs`/`Facts`/`State` are valid only for the duration of the `OnUpdate`/`OnTick`/`OnExecution` call. `Facts` is owned and reused by the event-loop goroutine and is `Reset` before every event — never retain `*Decision`, `*feature.Set`, or any slice obtained from them across calls.

## Observation and membership

```go
const MaxTokens = 2

type Token struct {
    ID       string
    AskPrice float64
    BidPrice float64
}

type Observation struct {
    MarketID    string
    TimeLeftSec int64
    TokenCount  int
    Tokens      [MaxTokens]Token
}
```

`Tokens` is a **fixed-size array**, not a map. Iterate `for i := 0; i < o.TokenCount; i++`, or use the lookup helper:

```go
tok, ok := o.Token(tokenID)
```

Position convention: `Tokens[0]` is **up**, `Tokens[1]` is **down**, matching `clobTokenIds` order. This convention carries the stop-loss direction logic — do not reorder or re-derive it.

Do **not** read raw SDK order books from a Strategy to price against the market. Declare `runtime.PortBooks` in `Needs()` and use the injected `deps.Books` (`runtime.BookPort`) when you genuinely need the full book; the top-of-book prices you normally need are already in `Tokens[i].AskPrice` (best ask) and `Tokens[i].BidPrice` (best bid).

```go
type BookPort interface {
    // The returned *sdk.OrderBook must be treated as read-only after return.
    // ok=false means unavailable, including "stale".
    Book(tokenID string) (*sdk.OrderBook, bool)
}
```

Note the SDK's ordering convention if you read a full book: `Asks` is descending (`Asks[len-1]` is best ask), `Bids` is ascending (`Bids[len-1]` is best bid).

## Features

Features are **strongly typed keys**, declared once in `feature/keys.go`:

```go
var (
    OpenPrice   = feature.Define[float64]("ext.open_price")
    LatestPrice = feature.Define[float64]("ext.latest_price")
    LatestZ     = feature.Define[float64]("ext.latest_z")
    ZWindows    = feature.Define[[]float64]("ext.z_windows")
)
```

Read them through the key, which returns `(value, ok)`:

```go
latestZ, ok := feature.LatestZ.Get(d.Facts)
if !ok {
    return nil // fail closed: missing is not zero
}
```

Rules:

- **Never** treat a missing feature as zero. `ok == false` means the provider has not produced it yet; fail closed.
- **Never** mutate a slice read from `Facts` (e.g. `ZWindows`). Treat it as a snapshot.
- **Never** add a feature by stuffing a new field into `Observation`. If you need a new derived value, define a key in `feature/` and have a Provider produce it (see below).
- `feature.MustGet` panics on a missing key. It is for replay/dev paths only — never in a trading path.

## Providers

A Provider is the only way derived facts enter the system. Its `Update` must be **pure writing** — publish no events, place no orders.

```go
type Provider interface {
    Name() string
    Provides() []feature.AnyKey  // keys this provider writes
    DependsOn() []feature.AnyKey // keys this provider reads
    Ports() map[PortName]any     // capability ports it exposes
    Update(ev core.Event, facts *feature.Set)
    Init(ctx context.Context)
}
```

At startup the engine validates all providers and **refuses to start** on:

- the same feature key provided by two providers;
- a `DependsOn` key that nobody provides;
- a dependency cycle;
- zero or more than one provider implementing `SnapshotProvider` (the engine's single market-fact source).

The returned order determines `Update` call order, so "provider A reads a key provider B writes" holds within a single event.

## OrderIntent rules

`runtime.OrderIntent` supports:

- `PLACE`
- `CANCEL`
- `SPLIT`
- `MERGE`

For PLACE, populate at minimum:

- `MarketID`
- `TokenID`
- `Price`
- `Side`
- `Size`

`OrderType` is optional and defaults to GTC at the framework boundary. **An empty `OrderType` means a GTC limit order, i.e. you are the maker.**

For CANCEL, provide `OrderID`.

For SPLIT/MERGE, provide `Size` and the required `Tokens`.

A Strategy must return intents. It must not call the Executor directly and must not directly mutate State.

## State and inventory

Use the supplied `state.Snapshot` for current orders and positions:

```go
pos, ok := d.State.Position.Tokens[tokenID] // state.TokenPosition{Available, Reserved, MarketID, ...}
openOrders := d.State.Orders                // orderID -> state.OrderReservation
```

`TokenPosition.MarketID` is the `conditionId` the token belongs to — use it to group inventory by market. The framework fills it in from fills, startup restore, and reconciliation; it can still be empty for a position it could not attribute, so do not branch on it being set.

Do not build a second authoritative order/position ledger inside a Strategy.

If a Strategy needs transient deduplication or intent state, keep that state explicitly and make its lifecycle clear. Do not confuse strategy-local state with the framework's authoritative State.

Remember that PolyPilot maintains provisional/order reservations and reconciles with the exchange. Do not manually emulate that accounting in strategy code.

## Execution behavior

Strategy execution failures can arrive asynchronously. If the strategy implements `ExecutionAwareStrategy`, handle only the execution outcomes relevant to the strategy.

Do not assume a submitted order is filled. Do not treat PLACE intent as a fill, and do not equate `ACCEPTED` with `FILLED`.

Do not use local intent creation as proof of position change; use the provided snapshot/execution events.

## Risk boundary

The runtime sends strategy intents through `RiskManager.Filter` before the Executor:

```go
Filter(orders []OrderIntent, snapshot state.Snapshot, midPrices map[string]float64) ([]OrderIntent, []IntentRejection)
```

It returns the approved subset and the per-intent rejections. Intent-level rejections do **not** abort the batch — one malformed CANCEL no longer takes down the rest. (Batch-level conditions such as a daily-loss halt still reject everything, tagging those rejections `risk.RejectBatchAborted` / `"BATCH_ABORTED"`.)

Do not duplicate global risk rules in every Strategy unless the rule is specifically part of the strategy's alpha logic.

Framework-level risk includes daily loss, market exposure, slippage, open orders, and market cooldown. Note that **slippage applies only to taker orders** (`MARKET_FAK`/`MARKET_FOK`) and is direction-aware; resting GTC limit orders are not slippage-checked. Strategy code should not bypass these controls.

Market exposure is per market (`conditionId`) and counts both resting order notional and the inventory you already hold (marked to mid) — so a position you are carrying consumes headroom for new orders, while `MERGE` gives headroom back. A `CANCEL` sent in the same batch as its replacement gives headroom back too: exposure and open-order count are both measured **after** the batch lands, so re-quoting is a one-batch operation rather than something that needs the cap headroom of two quotes. Balance is the exception — it is measured **before** the batch's cancels land, so a BUY must be payable from the reported available balance. See `architecture.md` → Risk model for the exact formula.

## Market/event handling

Use the event type explicitly and handle only what you subscribed to:

```go
switch d.Event.Type {
case core.EventMarket:
    ...
case core.EventOrderBook:
    ...
}
```

Return `nil` when no action is required.

Do not assume every event contains the same payload type. Validate event data before type assertions.

Current event types (see `core/constants.go`):

```text
MARKET, MARKET_RESOLVED, ORDERBOOK, EXTERNAL_PRICE, ORDER, EXECUTION,
RISK, STRATEGY, SYSTEM, METRICS, POSITION_EXPIRING, RECONCILE
```

The engine's dispatch path covers `MARKET`, `ORDERBOOK`, `EXTERNAL_PRICE` (input events), `EXECUTION`, and `POSITION_EXPIRING`. There is no `SIGNAL` event type.

## Time and market lifecycle

All framework time fields are UTC.

If the snapshot is not ready — no market yet, open price not yet published, or book stale — **the engine does not call the strategy at all**. You do not need to guard for readiness, but you must not assume you are called on every event.

For short-duration markets:

- check `d.Obs.TimeLeftSec` before entering;
- avoid acting on expired/near-expiry markets unless the strategy explicitly requires it;
- never infer market lifecycle solely from local wall-clock state when framework fields already provide it.

Do not access future market information.

## Indicators

Existing indicators include:

- `indicators.ZScore`
- `indicators.CalcImBalance`

Read their implementation before using them. Do not reimplement an existing indicator inside a Strategy.

If a requested indicator is genuinely missing, first decide whether it is reusable infrastructure. If yes, add it under `indicators/` with tests; otherwise keep a simple strategy-specific calculation local.

## Configuration

Follow the existing Viper pattern. Note the `Init` signature — it takes `deps`:

```go
func (s *Strategy) Init(bus *core.EventBus, ctx context.Context, cfg *viper.Viper, deps runtime.Dependencies) {
    s.Bus = bus
    s.books = deps.Books // only for ports you declared in Needs()

    sc := DefaultStrategyConfig()
    if cfg != nil {
        if sub := cfg.Sub("strategies.your_strategy"); sub != nil {
            if err := sub.Unmarshal(&sc); err != nil {
                // keep safe defaults
            }
        }
    }
    s.config = sc
}
```

Use safe defaults. A missing configuration must not silently produce dangerous zero-value trading parameters.

Note on Viper semantics: `Unmarshal` into a **pre-populated** struct does not zero fields absent from the config file, so starting from `DefaultXConfig()` preserves defaults for keys the user omitted. Unknown keys are silently ignored — there is no warning for a typo'd key.

## Testing

For every new Strategy:

- test entry conditions;
- test non-entry conditions;
- test exit/stop conditions;
- test missing/malformed features (a key that was never `Set` must yield `ok == false`, not a zero);
- test insufficient time remaining;
- test position-dependent behavior;
- test duplicate/no-op behavior where applicable;
- test generated OrderIntent fields.

Prefer deterministic unit tests over live Polymarket calls. Build `runtime.Decision` values directly in tests.

Run:

```bash
go test -race ./...
```

At minimum, run the relevant package tests.

## Safety checklist before finishing

Verify:

- [ ] `Subscribes()` lists exactly the event types the strategy acts on.
- [ ] `Needs()` lists every port the strategy dereferences in `Init`.
- [ ] No direct Executor call from Strategy.
- [ ] No direct State mutation from Strategy.
- [ ] Feature reads use `Key.Get` and handle `ok == false` (no silent zero).
- [ ] Nothing from `Decision`/`Facts` is retained across calls.
- [ ] No future data.
- [ ] No accidental order duplication.
- [ ] PLACE/CANCEL/SPLIT/MERGE intent fields are valid.
- [ ] Position checks use `state.Snapshot`.
- [ ] No blocking or long-running call inside a callback — all four callbacks run on the engine's event-loop goroutine, so blocking stalls every other strategy too.
- [ ] Time-left conditions are explicit.
- [ ] Configuration has safe defaults.
- [ ] New behavior has tests.
- [ ] `go test -race` passes or failures are explicitly reported.

## Source-of-truth rule

The repository source is authoritative. If this Skill conflicts with the current source, inspect the source and follow the current API. Do not invent an API based on this document.

For implementation examples, prefer:
- `main.go`
- `runtime/types.go`
- `runtime/providers.go`
- `feature/keys.go`, `feature/feature.go`
- `strategy/strategy.go`
- `strategy/*_test.go`
- `probability/features.go`
- `indicators/*`
- `risk/*`
