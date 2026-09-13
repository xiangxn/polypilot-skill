# PolyPilot Skill Manifest

Generated for `github.com/xiangxn/PolyPilot`.

Source basis:
- repository branch: `master` (commit `5087db4`, merged 2026-09-14 from `stage2-behavior-fixes`)
- repository URL: https://github.com/xiangxn/PolyPilot
- primary usage example: `main.go`
- primary runtime contracts: `runtime/types.go`
- provider validation / dependency resolution: `runtime/providers.go`
- typed feature keys: `feature/keys.go`, `feature/feature.go`
- strategy implementation: `strategy/strategy.go`
- feature production: `probability/features.go` (writes into `*feature.Set`)
- probability engine / provider: `probability/engine.go`
- core events/bus: `core/constants.go`, `core/event.go`, `core/bus.go`
- indicators: `indicators/`
- risk: `risk/`
- execution: `execution/`
- state: `state/`

The Skill intentionally does not copy the whole repository. The current repository source remains authoritative when APIs change.

**Staleness risk:** this Skill was regenerated against the commit above. If the checked-out source is older than that commit, the `Strategy`/`Provider`/`Decision` contracts described here will not match — verify against `runtime/types.go` before trusting any code sample.

Recommended installation:

```text
.agents/
└── skills/
    └── polypilot/
        ├── SKILL.md
        ├── architecture.md
        ├── strategy.md
        ├── features-and-indicators.md
        ├── workflow.md
        ├── examples.md
        ├── review-checklist.md
        └── MANIFEST.md
```
