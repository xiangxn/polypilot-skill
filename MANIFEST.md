# PolyPilot Skill Manifest

Generated for `github.com/xiangxn/PolyPilot`.

Source basis:
- repository branch: `master`
- repository URL: https://github.com/xiangxn/PolyPilot
- primary usage example: `main.go`
- primary runtime contracts: `runtime/types.go`
- strategy implementation: `strategy/strategy.go`
- probability feature implementation: `probability/features.go`
- probability engine: `probability/engine.go`
- core events/bus: `core/event.go`, `core/bus.go`
- indicators: `indicators/`
- risk: `risk/`
- execution: `execution/`
- state: `state/`

The Skill intentionally does not copy the whole repository. The current repository source remains authoritative when APIs change.

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
