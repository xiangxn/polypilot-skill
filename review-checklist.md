# PolyPilot Strategy Review Checklist

## Framework contracts

- [ ] Does `Subscribes()` list exactly the event types the strategy acts on?
- [ ] Does `Needs()` list every port the strategy dereferences, and no port it does not?
- [ ] Are all `Dependencies` fields used actually declared in `Needs()`?
- [ ] Is nothing from `Decision` / `*feature.Set` retained across calls?
- [ ] Does the strategy avoid mutating slices read from `Facts`?
- [ ] Are all four callbacks (`OnUpdate`/`OnTick`/`OnExecution`/`OnPositionExpiring`) free of blocking calls?

## Correctness

- [ ] Does the strategy react only to intended event types?
- [ ] Are event payload types checked?
- [ ] Are Feature reads done via `feature.Key.Get` with `ok == false` handled (no silent zero-substitution)?
- [ ] Is any new derived value added as a `feature.Key` + Provider rather than a new `Observation` field?
- [ ] Are token IDs treated explicitly?
- [ ] Are `Tokens` iterated `0..TokenCount-1` (not as a map)?
- [ ] Are time-left boundaries correct?
- [ ] Is market lifecycle handled correctly?
- [ ] Is there any future-data dependency?

## Orders

- [ ] PLACE contains MarketID, TokenID, Price, Side, Size.
- [ ] CANCEL contains OrderID.
- [ ] SPLIT/MERGE contains required token list and amount.
- [ ] OrderType is intentional.
- [ ] Size cannot accidentally become zero/negative.
- [ ] Price is validated where strategy-specific logic requires it.
- [ ] Duplicate orders are prevented where necessary.

## State

- [ ] Position information comes from `state.Snapshot`.
- [ ] Open orders come from `state.Snapshot`.
- [ ] Strategy does not mutate authoritative State.
- [ ] Strategy does not assume submission equals fill.
- [ ] Strategy tolerates reconciliation.

## Risk

- [ ] Strategy does not bypass Risk.
- [ ] Strategy does not duplicate global Risk limits without reason.
- [ ] Strategy-specific sizing is explicit.
- [ ] Strategy handles rejected intents where required.
- [ ] Strategy does not assume all-or-nothing submission — risk filters per intent, so part of a batch can be rejected.

## Concurrency

- [ ] Strategy-local state is touched only from the framework callbacks (which are serialized on the event-loop goroutine), or is explicitly protected.
- [ ] No unsafe concurrent map access if the strategy spawns goroutines or shares state with anything outside the callbacks.
- [ ] No blocking operation in a callback — the event loop is shared with every other strategy.

## Tests

- [ ] Entry test.
- [ ] Exit test.
- [ ] Negative test.
- [ ] Missing feature test (`ok == false`, not zero).
- [ ] Boundary test.
- [ ] Position test.
- [ ] Intent field assertions.
- [ ] Race test.
- [ ] Guards mutation-checked (delete the guarded line, confirm the test fails).

## Final commands

```bash
go test -race ./...
go vet ./...
```

Use repository-specific lint/build commands when present.
