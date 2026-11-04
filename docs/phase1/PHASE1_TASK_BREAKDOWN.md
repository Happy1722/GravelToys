# GravelToys Race — Phase 1 Task Breakdown

**Date:** 2026-11-04 · **Status:** PROPOSED, awaiting approval. **No task may start until you approve.**
**Repository access: UNAVAILABLE.** Task P1-00 (audit) blocks everything else. All paths are **proposed new paths** from `PHASE1_IMPLEMENTATION_SPEC.md` and may be corrected by the audit.

Effort is a rough estimate in focused working days **[Proposed]**.

| ID | Task | Depends on | Files (proposed, new) | Tests added | Done when | Effort |
|---|---|---|---|---|---|---|
| **P1-00** | **Repository audit** (Part A of the handoff): tree, manifests, stack, existing telemetry/overlay/DB/network/tests/CI. Produce `PROJECT_AUDIT.md` with Preserve/Refactor/Replace/Not-implemented + file evidence. Confirm or correct every path in this plan; decide ADR-13 (stack) | Repo access | `docs/architecture/PROJECT_AUDIT.md` | — | Audit approved by you | 0.5–1 |
| P1-01 | Workspace scaffolding: add crate skeletons as workspace members; lint config (`unsafe_code` forbid, `unwrap_used`/`expect_used` deny in libs); `rust-toolchain.toml` if absent | P1-00 | `crates/*/Cargo.toml`, `crates/*/src/lib.rs` (empty modules with docs only, no placeholder logic), workspace `Cargo.toml` member list (**the only existing file touched**) | `cargo build` | Workspace builds on Linux + Windows | 0.5 |
| P1-02 | Units + geometry | P1-01 | `crates/gt-schema/src/{units,geometry}.rs`, `docs/architecture/COORDINATES.md` | `tests/units.rs`, `tests/geometry.rs` (incl. proptest quaternion round trip, ISO 8855 sign tests) | Spec §4 acceptance passes | 1 |
| P1-03 | Availability, time, protocol IDs | P1-01 | `crates/gt-schema/src/{availability,time,protocol}.rs` | protocol `supports` tests; `Group::get` property test | Spec §2.1, §5, §11 time types pass | 0.5 |
| P1-04 | Hot frame, opponent poses, snapshots | P1-02, P1-03 | `crates/gt-schema/src/{frame,opponents,snapshot}.rs` | `tests/frame_layout.rs` (size asserts, `Copy` asserts, serde round trip incl. custom 64-car array encoding) | Spec §3 acceptance passes | 1 |
| P1-05 | FrameBus | P1-03 | `crates/gt-bus/src/{lib,frame_bus,latest,stats}.rs` | `tests/gaps.rs`, `tests/isolation.rs`, `tests/alloc.rs`, stress test; `benches/publish.rs` | Spec §6.1/§6.3 acceptance; baseline recorded | 1.5 |
| P1-06 | EventBus | P1-05 | `crates/gt-bus/src/event_bus.rs` | `tests/events_lag.rs`, unsubscribe test | Spec §6.2 acceptance | 0.5 |
| P1-07 | Recorder format + writer | P1-03 | `crates/gt-recorder/src/{lib,format,writer}.rs` | non-blocking/stalled-writer test, rotation test, `benches/write.rs` | Spec §7.1–7.2 acceptance | 1.5 |
| P1-08 | Recorder reader | P1-07 | `crates/gt-recorder/src/reader.rs` | `tests/roundtrip.rs` (proptest), `tests/truncation.rs` (every offset), `tests/corruption.rs` | Spec §7.3 acceptance | 1 |
| P1-09 | Session tracker | P1-04 | `crates/gt-session/src/{lib,state,events,laps}.rs` | `tests/lifecycle.rs`, `tests/laps.rs`, `tests/edge_cases.rs` (table-driven, all cases in spec §9) | Spec §9 acceptance | 2 |
| P1-10 | Diagnostics | P1-04, P1-05 | `crates/gt-diagnostics/src/{lib,health,snapshot}.rs` | `tests/health.rs` | Spec §10 acceptance | 1 |
| P1-11 | Replay harness + synthetic fixtures (test-only feature, labelled synthetic) | P1-06, P1-08, P1-09, P1-10 | `crates/gt-replay/src/{lib,decoder,runner,digest,synthetic}.rs`, `crates/gt-replay/bin/gtr-replay.rs` | `tests/determinism.rs` (golden digest), `tests/unknown_protocol.rs`, `tests/end_to_end.rs`, `benches/replay.rs` | Spec §8 acceptance; digest identical on Linux and Windows | 2 |
| P1-12 | Policy tests | P1-01 | `crates/gt-policy-tests/tests/{forbidden_apis,dependency_rules}.rs` | the tests themselves, plus a self-test that a planted forbidden string is detected (in a temp file, not committed) | Spec §12.4 passes | 0.5 |
| P1-13 | CI wiring (only if CI exists or you approve adding it) | P1-12, audit | CI workflow file: path depends on the audit | runs §12.5 commands on Linux + Windows | Green CI | 0.5 |
| P1-14 | Documentation reconciliation: update spec/ADRs with anything that changed during build; record benchmark baselines | all | `docs/architecture/*.md` | — | You approve Phase 1 completion | 0.5 |

**Total estimate:** ~14–15 days of focused work after the audit **[Proposed; not a commitment]**.

## Critical path

`P1-00 → P1-01 → P1-02/P1-03 → P1-04 → P1-09 ┐`
`                     P1-03 → P1-05 → P1-06 ─┤→ P1-11 → P1-14`
`                     P1-03 → P1-07 → P1-08 ─┘`
P1-10 and P1-12 run in parallel.

## Build and test commands (per task and at phase end)

```bash
cargo fmt --all -- --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
cargo test -p gt-replay --features test-fixtures
cargo bench -p gt-bus -p gt-recorder -p gt-replay
```

## Review checkpoints (where I stop and report to you)

1. After P1-00: audit + corrected paths + stack decision. **Approval required.**
2. After P1-04: schema frozen for Phase 1 (sizes, field list). Your sign-off avoids churn later.
3. After P1-11: determinism demo (two identical digests) + benchmark baselines.
4. After P1-14: Phase 1 completion review. **Approval required before Phase 2.**

## Out of scope for Phase 1 (explicit)

Game adapters, any Windows API usage, UI changes, Tauri/Electron work, network listeners, remote API, Android, storage/database, track, analytics, racing line, AI, voice, wheelbase anything.

## Rollback

Each task is one or more commits touching only new crates (plus the single workspace member list). Rollback per task = revert its commits; whole phase = remove `crates/gt-*` and the member entries.
