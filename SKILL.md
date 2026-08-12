---
name: polypilot
description: Use when developing, modifying, reviewing, or testing trading strategies and strategy-related components for the github.com/xiangxn/polypilot Go framework. Follow PolyPilot's event-driven architecture, Strategy/OrderIntent contracts, Observation/Features model, state/risk/execution boundaries, and testing requirements. Prefer existing framework APIs and patterns over inventing new abstractions.
---

# PolyPilot Agent Skill

You are working in the PolyPilot repository (`github.com/xiangxn/polypilot`), a Go event-driven automation engine for short-duration Polymarket binary markets.

## Primary objective

When a user asks for a new strategy, strategy modification, indicator, signal rule, or strategy bug fix:

1. Inspect the current repository source before coding.
2. Read `main.go` to understand the actual runtime composition.
3. Read `runtime/types.go` for the public contracts.
4. Read the closest existing implementation under `strategy/`.
5. Reuse existing `runtime.Observation`, `runtime.OrderIntent`, `state.Snapshot`, indicators, helpers, and SDK types.
6. Keep strategy logic inside `strategy/` unless the requested feature genuinely belongs elsewhere.
7. Never bypass Risk or Executor from a Strategy.
8. Add focused tests for the new behavior.
9. Run `go test -race ./...` when possible.
10. Report exactly what changed, what was tested, and any assumptions.

## Architecture

The normal flow is:

`Feed -> Probability -> Strategy -> Risk -> Executor -> State`

The EventBus connects the components. The runtime engine handles MARKET, ORDERBOOK, SIGNAL, and EXECUTION events. Strategy code produces `[]runtime.OrderIntent`; Risk validates intents before execution.

Do not move responsibilities across these boundaries merely to make a strategy easier to implement.

## Strategy contracts

The core interface is:

```go
type Strategy interface {
    Init(bus *core.EventBus, ctx context.Context, cfg *viper.Viper)
    OnUpdate(e core.Event, o Observation, stateSnap state.Snapshot) []OrderIntent
}
```

Optional capabilities:

```go
type TickStrategy interface {
    OnTick(now time.Time, o Observation, stateSnap state.Snapshot) []OrderIntent
}

type ExecutionAwareStrategy interface {
    OnExecution(ev core.ExecutionEvent, o Observation, snap state.Snapshot) []OrderIntent
}

type MarketResolved interface {
    OnResolved(info *sdk.ResolvedInfo)
}
```

Use `OnUpdate` for event-driven decisions, `OnTick` only when periodic evaluation is actually required, and `OnExecution` when execution outcomes must change strategy behavior.

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

`OrderType` is optional and defaults to GTC at the framework boundary.

For CANCEL, provide `OrderID`.

For SPLIT/MERGE, provide `Size` and the required `Tokens`.

A Strategy must return intents. It must not call the Executor directly and must not directly mutate State.

## Observation

`runtime.Observation` provides:

- `At`
- `MarketID`
- `Slug`
- `Tokens`
- `TokenIds`
- `Probability`
- `TimeLeftSec`
- `Confidence`
- `Features`
- `GetOrderBook`
- `FetchPrices`
- `CheckResolved`

Current probability-engine features include:

- `latestZ`
- `zWindows`
- `openPrice`
- `latestPrice`
- `endTime`
- `diffPrice`

Feature keys are dynamic and typed as `any`. Always use checked assertions:

```go
v, ok := o.Features["latestZ"].(float64)
if !ok {
    return nil
}
```

Never assume a feature exists or has the expected type.

## State and inventory

Use the supplied `state.Snapshot` for current orders and positions. Do not build a second authoritative order/position ledger inside a Strategy.

If a Strategy needs transient deduplication or intent state, keep that state explicitly and make its lifecycle clear. Do not confuse strategy-local state with the framework's authoritative State.

Remember that PolyPilot maintains provisional/order reservations and reconciles with the exchange. Do not manually emulate that accounting in strategy code.

## Execution behavior

Strategy execution failures can arrive asynchronously. If the strategy implements `ExecutionAwareStrategy`, handle only the execution outcomes relevant to the strategy.

Do not assume a submitted order is filled. Do not treat PLACE intent as a fill.

Do not use local intent creation as proof of position change; use the provided snapshot/execution events.

## Risk boundary

The runtime sends strategy intents through Risk before Executor.

Do not duplicate global risk rules in every Strategy unless the rule is specifically part of the strategy's alpha logic.

Framework-level risk includes daily loss, market exposure, slippage, open orders, and market cooldown. Strategy code should not bypass these controls.

## Market/event handling

Use the event type explicitly:

```go
switch e.Type {
case core.EventMarket:
    ...
case core.EventOrderBook:
    ...
}
```

Return `nil` when no action is required.

Do not assume every event contains the same payload type. Validate event data before type assertions.

## Time and market lifecycle

All framework time fields are UTC.

For short-duration markets:

- check `o.TimeLeftSec` before entering;
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

Follow the existing Viper pattern:

```go
type StrategyConfig struct {
    Threshold float64 `mapstructure:"threshold"`
}

func DefaultStrategyConfig() StrategyConfig {
    return StrategyConfig{
        Threshold: 1.0,
    }
}

func (s *Strategy) Init(bus *core.EventBus, ctx context.Context, cfg *viper.Viper) {
    s.Bus = bus
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

## Testing

For every new Strategy:

- test entry conditions;
- test non-entry conditions;
- test exit/stop conditions;
- test missing/malformed Features;
- test insufficient time remaining;
- test position-dependent behavior;
- test duplicate/no-op behavior where applicable;
- test generated OrderIntent fields.

Prefer deterministic unit tests over live Polymarket calls.

Run:

```bash
go test -race ./...
```

At minimum, run the relevant package tests.

## Safety checklist before finishing

Verify:

- [ ] No direct Executor call from Strategy.
- [ ] No direct State mutation from Strategy.
- [ ] No unchecked `Features[...]` type assertions.
- [ ] No future data.
- [ ] No accidental order duplication.
- [ ] PLACE/CANCEL/SPLIT/MERGE intent fields are valid.
- [ ] Position checks use `state.Snapshot`.
- [ ] Time-left conditions are explicit.
- [ ] Configuration has safe defaults.
- [ ] New behavior has tests.
- [ ] `go test -race` passes or failures are explicitly reported.

## Source-of-truth rule

The repository source is authoritative. If this Skill conflicts with the current source, inspect the source and follow the current API. Do not invent an API based on this document.

For implementation examples, prefer:
- `main.go`
- `runtime/types.go`
- `strategy/strategy.go`
- `strategy/*_test.go`
- `probability/features.go`
- `indicators/*`
- `risk/*`
