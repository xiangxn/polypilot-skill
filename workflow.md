# Agent Workflow for PolyPilot

## New strategy

### Phase 1 — Understand

1. Read `main.go`.
2. Read `runtime/types.go`.
3. Read `runtime/providers.go` (startup validation: which ports and features the engine guarantees).
4. Read `feature/keys.go` (which derived values are available to read).
5. Read `strategy/strategy.go`.
6. Inspect relevant tests.
7. Inspect existing indicators/features.
8. Identify the exact event(s) that should drive the strategy.

### Phase 2 — Design

Write a small decision table:

```text
Input event
    |
Market eligible?
    | no -> return nil
    v
Data ready?
    | no -> return nil
    v
Signal true?
    | no -> return nil
    v
Inventory condition?
    | no -> return nil
    v
Build OrderIntent
```

Explicitly define:

- entry
- exit
- stop
- duplicate prevention
- time-left limits
- position dependency
- order cancellation
- execution failure behavior

### Phase 3 — Implement

Prefer a small Strategy implementation.

Do not modify Engine, Executor, or State unless the requested feature cannot be expressed through the existing contract.

Read features through their typed keys and handle `ok == false`; check event payload type assertions before use.

Use existing helpers.

### Phase 4 — Test

Add unit tests before or alongside implementation.

Minimum tests:

- valid signal
- invalid signal
- missing feature
- time-left boundary
- position boundary
- generated intent correctness

Then run:

```bash
go test -race ./...
```

### Phase 5 — Review

Check `review-checklist.md` in full. The short form:

```text
Contracts
[ ] Subscribes() lists exactly the handled event types
[ ] Needs() lists exactly the ports the strategy dereferences
[ ] Nothing from Decision/Facts retained across calls
[ ] No blocking call inside a callback

Architecture
[ ] Strategy only decides
[ ] Risk remains authoritative
[ ] Executor remains authoritative
[ ] State remains authoritative

Data
[ ] Feature reads use Key.Get with ok == false handled
[ ] No future data
[ ] No accidental ordering dependency on Tokens (use Tokens[0]=up, [1]=down)

Orders
[ ] Correct Action
[ ] Correct MarketID/TokenID
[ ] Correct Side
[ ] Correct Price
[ ] Correct Size
[ ] CANCEL uses OrderID

Concurrency
[ ] Local state protected if used outside callbacks
[ ] No unsafe goroutine assumptions

Testing
[ ] Unit tests added
[ ] Race tests pass
```

## Modifying an existing strategy

1. Read the whole current strategy implementation.
2. Read its tests.
3. Identify behavior that must remain unchanged.
4. Change the smallest possible surface.
5. Add regression tests for the changed behavior.
6. Run the relevant tests and then the full race suite.

Do not rewrite a working strategy merely to make it look cleaner unless the user asked for refactoring.

## Adding a reusable indicator

1. Confirm it is not already available.
2. Define exact semantics.
3. Add implementation under `indicators/`.
4. Add edge-case tests.
5. Keep strategy code dependent on the indicator, not its internal implementation.

## Adding a feature

Use a feature (not an `Observation` field) when the derived value is shared across strategies.

1. Declare the key in `feature/keys.go`: `var MySignal = Define[float64]("strategy.my_signal")`. Declare it exactly once — two declarations produce two unrelated slots.
2. Have a `runtime.Provider` write it in `Update(ev, facts)` (pure writing: no events, no orders).
3. Declare `Provides()` / `DependsOn()` so the engine can validate and topologically order providers. A duplicate key, an unprovided dependency, or a cycle is a startup failure.
4. Register the provider in `main.go`.

Verify the feature is populated consistently after market reset and during updates, and that a never-written key reads back as `ok == false` rather than zero.

Providers run on the event-loop goroutine and share a per-event `*feature.Set`, so the set needs no locking. Any state a provider keeps across events (buffers, tickers started in `Init`) does.

## Debugging

When a strategy behaves unexpectedly:

1. Inspect event type (`d.Event.Type`).
2. Inspect `d.At` and `d.Obs.MarketID`.
3. Inspect `d.Obs.TimeLeftSec`.
4. Inspect `d.Obs.Tokens` / `TokenCount` (are both legs priced? is the book present at all?).
5. Inspect `d.Facts` (are the keys you read actually `ok == true`? a missing key silently skips your logic).
6. Inspect the state snapshot.
7. Inspect the generated OrderIntent.
8. Inspect Risk rejections (per intent, with reasons).
9. Inspect execution events.
10. Inspect reconciliation effects.

Do not start by changing order execution code when the problem is a strategy decision.

## Completion report

After coding, report:

- files changed
- strategy behavior
- configuration keys
- tests run
- test result
- known limitations
- whether live trading was exercised

Never claim live execution was tested if only unit tests were run.
