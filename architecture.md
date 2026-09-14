# PolyPilot Architecture Reference

## Runtime graph

```text
Market Feeds / Observers
    |
    v
EventBus
    |
    v
Providers (topological order)  ---> feature.Set (derived facts)
    |
    v
SnapshotProvider               ---> Observation (where/what the market is)
    |
    v
Decision { At, Event, Obs, Facts, State }
    |
    v
Strategy.OnUpdate
    |
    | []OrderIntent
    v
Risk.Filter
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

Two things are worth internalizing:

- **Providers run before dispatch, on the event-loop goroutine.** They turn an event into derived facts. Their output (`feature.Set`) is reset before every event and reused.
- **`Observation` carries only place facts; derived values live in `Facts`.** The split is what lets a strategy declare exactly which derived quantities it depends on, and lets the framework tell whether they are ready.

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
- providers (the probability engine among them)
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

When adding a **Provider**, register it and declare its read/write sets:

```go
Providers: []runtime.Provider{
    probability.NewEngine(probability.Options{ /* ... */ }),
    myprovider.New(),
},
```

Do not redesign Engine composition for a strategy that can be expressed through the existing Strategy contract.

## Runtime interfaces

`runtime/types.go` defines the main extension points:

- `Feed`
- `Observer`
- `Provider` — writes derived facts, exposes ports
- `SnapshotProvider` — the single source of place facts (exactly one provider must implement it)
- `Strategy`
- `TickStrategy`
- `ExecutionAwareStrategy`
- `PositionExpiringAwareStrategy`
- `RiskManager`
- `Executor`

Supporting types:

- `Observation`, `Token` — place facts
- `Decision` — the complete per-call input
- `Dependencies`, `PortName`, `BookPort` — capability injection

The `Engine` wires these interfaces together, validating them at startup (`runtime/providers.go`):

- a feature key must have exactly one provider;
- every `DependsOn` key must be provided by someone;
- the provider graph must be acyclic;
- exactly one provider must implement `SnapshotProvider`;
- every port a strategy declares in `Needs()` must be supplied by some provider.

All of these **fail startup** rather than surfacing as a nil pointer or a silently missing value at runtime.

## Observation model

`runtime.Observation` is the strategy-facing view of the current market:

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

func (o Observation) Token(id string) (Token, bool)
```

`Tokens` is a **fixed-size array** with an explicit `TokenCount`; iterate `0..TokenCount-1`. Position convention: `Tokens[0]` is up, `Tokens[1]` is down, matching `clobTokenIds` order — the stop-loss logic depends on it.

There is no `Features`/`Probability`/`GetOrderBook`/`CheckResolved` on the Observation. Derived quantities live behind typed `feature.Key`s; live capabilities live behind ports in `Dependencies`. Timestamps, slugs and end times were deliberately removed — they had no consumers, and a field that looks informative but is dead is the most dangerous kind.

## Decision model

```go
type Decision struct {
    At    time.Time
    Event core.Event
    Obs   Observation
    Facts *feature.Set
    State state.Snapshot
}
```

Contract: `Obs`/`Facts`/`State` are valid **only** for the duration of the callback. `Facts` is owned and reused by the event-loop goroutine. Retaining `*Decision`, `*feature.Set`, or any slice read from it across calls is a bug.

If the snapshot is not ready — no market, open price not yet published, or the book is stale beyond the configured threshold — the engine does **not** build a `Decision` and does **not** call the strategy.

## Feature model

Features are strongly typed keys declared in `feature/keys.go`:

```go
var (
    OpenPrice   = feature.Define[float64]("ext.open_price")
    LatestPrice = feature.Define[float64]("ext.latest_price")
    LatestZ     = feature.Define[float64]("ext.latest_z")
    ZWindows    = feature.Define[[]float64]("ext.z_windows")
)
```

`Define[T]` allocates a dense slot; `Set`/`Get` use that slot rather than a string map. `Get` returns `(T, bool)` — a key that was never written yields `ok == false`, never a silent zero. Naming is `<namespace>.<quantity>`.

## Event types

The engine's dispatch path covers:

- **input events** — `MARKET`, `ORDERBOOK`, `EXTERNAL_PRICE` → providers update facts, then subscribed strategies run
- **`EXECUTION`** → `ExecutionAwareStrategy.OnExecution`
- **`POSITION_EXPIRING`** → `PositionExpiringAwareStrategy.OnPositionExpiring`

Other types exist for risk/metrics/reconciliation and are handled inside the runtime, not dispatched to strategies.

There is **no `SIGNAL` event type**. Always inspect `core/constants.go` and `core/event.go` before adding new event-driven behavior.

Caveat on `POSITION_EXPIRING`: the dispatch case is wired, but the producer (`state.StartPositionExpiringLoop` / `RegisterMarketExpiry`) has no production caller yet, so it does not fire today.

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

Strategy code should consume snapshots (`state.Snapshot`) and emit intents rather than reproducing this state machine.

## Risk model

Risk is a hard boundary between strategy and execution.

```go
Filter(orders []OrderIntent, snapshot state.Snapshot, midPrices map[string]float64) ([]OrderIntent, []IntentRejection)
```

Risk filters **per intent**: the approved subset is returned along with the rejections and their reasons. Intent-level rejections do not abort the batch. Batch-level conditions (e.g. daily-loss halt) still reject everything with `RejectBatchAborted`.

The current framework includes protections for:

- maximum daily loss
- maximum exposure per market — per `marketID` (= `conditionId`), resting order notional **plus** held inventory
- maximum slippage — **taker orders only** (`MARKET_FAK`/`MARKET_FOK`), direction-aware: a BUY only counts when priced above mid, a SELL only when below
- maximum open orders — post-batch count, so same-batch cancels free slots
- market cooldown

Exposure in detail, because it decides whether your next order still fits:

- **resting orders**, per order: a BUY counts the `price × size` USDC it locks; a SELL locks tokens and counts its notional `price × remainingSize` — do not read a SELL's `Reserved` as USDC, it is a token count.
- **held inventory**: `Available × midPrices[tokenID]`, i.e. marked to market. `Reserved` tokens are not counted twice — the resting SELL above already carries them.
- **unvaluable inventory is skipped, not guessed**: a token with no mid, or a position the framework could not attribute to a market, contributes nothing.
- **`SPLIT` consumes** `size` USDC of the cap; **`MERGE` releases** it (it burns a token pair and returns `size` USDC). A market pressed against its cap can therefore still be unwound, and a batch that merges before re-quoting nets out rather than double-counting.
- **a `CANCEL` releases it too**, when the cancel travels in the same batch as the replacement. The two caps that measure a *stock* — exposure and open orders — compare the state **after the batch lands**, so cancel-then-re-place in one batch is a replace, not a double count; otherwise a market using more than half its cap could never re-quote. The credit comes from the snapshot only: an `OrderID` that is unknown, stale, or belongs to another market frees nothing, and a repeated cancel frees nothing twice. A cancel that fails leaves its reservation in the ledger, so the next batch counts it again — over-commitment lasts one in-flight batch and self-corrects.
- **balance is the exception: it is measured at the peak.** The executor submits placements *before* cancels, so a BUY still has to be payable from the snapshot's available balance even when the same batch is cancelling a resting BUY that holds that money.

Resting GTC limit orders are not slippage-checked, because "limit buy below mid" is not slippage.

Do not bypass or duplicate these controls without a clear reason.

## Reconciliation

Exchange state is authoritative during reconciliation. The current repository uses periodic reconciliation plus WS-triggered reconciliation.

Strategies should therefore tolerate local state being corrected by reconciliation.

## Concurrency

PolyPilot is concurrent.

Potentially concurrent areas include:

- EventBus subscribers
- provider background loops (`Init`)
- tick-driven strategy evaluation
- probability snapshot reads
- execution events
- reconciliation
- executor queues

All four strategy callbacks — `OnUpdate`, `OnTick`, `OnExecution`, `OnPositionExpiring` — are invoked from the single event-loop goroutine (see the `for/select` in `runtime/engine.go`). Strategy state touched only from those callbacks is therefore serialized and needs no mutex.

Do **not** generalize that into "the framework is single-threaded": providers run their own goroutines (`Provider.Init`), the executor has a queue worker, reconciliation runs independently, and `OnUpdate` blocks the entire engine while it runs (so a blocking call in a callback stalls event processing for every strategy). If a Strategy spawns goroutines or shares state with anything outside the callbacks, protect it.

## Time

All framework time fields are UTC.

Use framework timestamps and `TimeLeftSec` where available.

Never mix local time and UTC implicitly.
