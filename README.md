# GravelToys Race — AI Race Engineer

Sim-racing telemetry, HUD and AI race-engineer platform. **Status: architecture baseline (Phase 0) awaiting approval. No implementation code yet.**

## Documentation

### Phase 0: Architecture
- [Final architecture baseline](docs/phase0/PHASE0_FINAL_ARCHITECTURE.md): start here
- [Architecture decision log (ADR-01 … ADR-13)](docs/phase0/ARCHITECTURE_DECISION_LOG.md)
- [Architecture review of the original Phase 0 bundle](docs/phase0/PHASE0_ARCHITECTURE_REVIEW.md)
- [Game compatibility evidence](docs/phase0/GAME_COMPATIBILITY_EVIDENCE.md)
- [Risk register](docs/phase0/RISK_REGISTER.md)
- [Project audit](docs/phase0/PROJECT_AUDIT.md)

### Phase 1: Core telemetry foundation
- [Implementation specification](docs/phase1/PHASE1_IMPLEMENTATION_SPEC.md)
- [Task breakdown](docs/phase1/PHASE1_TASK_BREAKDOWN.md)

## Ground rules

- Official game interfaces only: no DLL injection, hooks or game-process memory access.
- The GravelToys Wheelbase application, firmware and FFB configuration are never modified by this project.
- Every technical claim in the docs is labelled: Verified in repository, Officially documented, Tested, Proposed, or Unknown.
