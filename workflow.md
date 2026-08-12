# Agent Workflow for PolyPilot

## New strategy

### Phase 1 — Understand

1. Read `main.go`.
2. Read `runtime/types.go`.
3. Read `strategy/strategy.go`.
4. Inspect relevant tests.
5. Inspect existing indicators/features.
6. Identify the exact event(s) that should drive the strategy.

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

Use checked type assertions.

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

Check:

```text
Architecture
[ ] Strategy only decides
[ ] Risk remains authoritative
[ ] Executor remains authoritative
[ ] State remains authoritative

Data
[ ] No future data
[ ] No unchecked feature assertions
[ ] No accidental map-order dependency

Orders
[ ] Correct Action
[ ] Correct MarketID/TokenID
[ ] Correct Side
[ ] Correct Price
[ ] Correct Size
[ ] CANCEL uses OrderID

Concurrency
[ ] Shared state protected
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

## Adding a probability feature

Use the probability layer when the feature is a shared market observation.

Verify that the feature is populated consistently after market reset and during updates.

The feature should be safe for concurrent snapshot reads.

## Debugging

When a strategy behaves unexpectedly:

1. Inspect event type.
2. Inspect Observation timestamp and MarketID.
3. Inspect TimeLeftSec.
4. Inspect Tokens.
5. Inspect Features.
6. Inspect state snapshot.
7. Inspect generated OrderIntent.
8. Inspect Risk rejection.
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
