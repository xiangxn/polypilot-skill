# PolyPilot Strategy Review Checklist

## Correctness

- [ ] Does the strategy react only to intended event types?
- [ ] Are event payload types checked?
- [ ] Are required Observation fields present?
- [ ] Are dynamic Features accessed with checked type assertions?
- [ ] Are token IDs treated explicitly?
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

## Concurrency

- [ ] Strategy-local shared state is protected.
- [ ] No unsafe concurrent map access.
- [ ] No assumptions that all callbacks occur on one goroutine.
- [ ] No blocking operation in a latency-sensitive callback without justification.

## Tests

- [ ] Entry test.
- [ ] Exit test.
- [ ] Negative test.
- [ ] Missing feature test.
- [ ] Boundary test.
- [ ] Position test.
- [ ] Intent field assertions.
- [ ] Race test.

## Final commands

```bash
go test -race ./...
go vet ./...
```

Use repository-specific lint/build commands when present.
