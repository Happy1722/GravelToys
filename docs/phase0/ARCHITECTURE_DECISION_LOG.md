# GravelToys Race — Architecture Decision Log

**Date:** 2026-11-04 · **Status of every entry: PROPOSED, awaiting Vaibhav's approval.**
**Repository access: UNAVAILABLE.** No decision below is verified against code. Where a decision depends on the existing stack, that's stated.

Labels: **[Repo]** verified in repository (none) · **[Official]** officially documented · **[Tested]** tested (none) · **[Proposed]** recommendation · **[Unknown]** needs validation · **[Third-party]** community source, not authoritative.

Decisions A1–A9 come from `PHASE0_ARCHITECTURE_REVIEW.md` §10. Each was re-checked rather than carried over. Two were changed substantially (A1, A6) and one is deferred (A5). ADR-10 to ADR-13 are new.

---

## Summary

| ADR | Decision | Verdict | Changed from review? |
|---|---|---|---|
| ADR-01 (A1) | Coordinate convention | **MODIFY** | Yes. The review's "Y-up, Z-back" is replaced by **Z-up world + ISO 8855 vehicle frame** |
| ADR-02 (A2) | Hot frame / slow snapshots, quaternions, SI | **MODIFY** | Yes. Per-group availability; frames become `Copy` values, not `Arc` |
| ADR-03 (A3) | Bounded multi-subscriber bus | **MODIFY** | Yes. Two buses: lossy frame bus, and an event bus that reports lag |
| ADR-04 (A4) | Versioned protocol identification | **ACCEPT** | Detail added |
| ADR-05 (A5) | `ProjectionReadiness` instead of confidence scalar | **DEFER** | Yes. Out of Phase 1; no camera fields in Phase 1 schema |
| ADR-06 (A6) | Replace loopback WebSocket | **MODIFY** | Yes. Split into two separate security boundaries (desktop IPC ≠ remote API) |
| ADR-07 (A7) | Wheelbase bridge blocked | **ACCEPT** | No |
| ADR-08 (A8) | Phase order | **MODIFY** | Yes. Camera spike decoupled from the first adapter |
| ADR-09 (A9) | Never `OpenProcess` on games | **MODIFY** | Yes. Stricter: also bans process-inspection libraries that open processes internally |
| ADR-10 | First game adapter | **PROPOSED: F1 25** (conditional) | New |
| ADR-11 | Phase 1 has no network listener and no UI change | **PROPOSED** | New |
| ADR-12 | Injectable clock for deterministic replay | **PROPOSED** | New |
| ADR-13 | Stack decision deferred to audit | **PROPOSED** | New |

---

## ADR-01 — Coordinate convention (A1) — **MODIFY**

**Review proposed:** right-handed, X-right, Y-up, Z-back (forward = −Z).

**Re-evaluation.** That choice fixes the bundle's handedness error, but it's a *rendering* convention (OpenGL). Most of the code (analytics, session, track map, racing line) works in the ground plane and in the car's frame. A Y-up world makes the track map use (X, −Z), and a "forward = −Z" car frame produces sign mistakes in every lateral/longitudinal calculation. Vehicle dynamics already has a standard for this: **ISO 8855** (X forward, Y left, Z up, right-handed). **[Proposed]** (ISO 8855 itself is a published standard; that we adopt it is the proposal.)

**Decision.**
- **World frame:** right-handed, **Z up**, X/Y horizontal. The X/Y orientation relative to the sim's own axes is per-adapter and documented in the adapter's unit table. A track map is simply (x, y).
- **Vehicle body frame:** ISO 8855, so X forward, Y left, Z up. Yaw positive = turning left (counter-clockwise seen from above).
- **Orientation:** unit quaternion, body → world.
- **Rendering frames** (camera/view, Y-up or other) exist only inside the future renderer, converted at its boundary.

**Dependencies / consequences.** Every adapter needs a fixed axis-mapping and sign table. Analytics formulas are written in ISO 8855 terms. The renderer (Phase 7+) owns one conversion.

**Document changes.** Replace §4.1 item 2 of the bundle and Module C's coordinate sentence. Add `docs/architecture/COORDINATES.md` **[Proposed path]** with diagrams and the per-sim table.

**Tests.**
1. Schema unit test: the quaternion for yaw +90° rotates body +X onto world +Y.
2. Per-adapter fixtures (Phase 2): straight line → yaw rate ≈ 0; left-hand corner → positive yaw rate and positive lateral (left) acceleration; uphill → positive world Z velocity.
3. Property test: `quat → rotation matrix → quat` round trip within 1e-5.

---

## ADR-02 — Hot frame + slow snapshots, quaternion, SI units (A2) — **MODIFY**

**Review proposed:** split the frame; quaternion; SI only.

**Re-evaluation.**
- The split is right. The bundle's single frame mixes 100 Hz values with strings and a `Vec`.
- *Per-field* availability doubles the frame size and creates thousands of almost-identical branches. Sims expose data in **groups** (F1 packet types, ACC pages, LMU telemetry vs scoring). Availability and freshness belong **per group**.
- The review left frames as `Arc<…>`. Allocating an `Arc` per frame contradicts the zero-allocation goal. If the hot frame is a fixed-size `Copy` struct (estimated a few hundred bytes **[Unknown — measure with `size_of`]**), copying it into each subscriber queue costs less than reference counting and needs no allocation.
- "SI only" exceptions: engine speed stays in **rev/min** (no one reads rad/s), and normalised pedals stay 0..1. These are named newtypes, so the exception is explicit.

**Decision.**
- `HotFrame`: `Copy`, `#[repr(C)]`, fixed size, no heap types. Groups: inputs, powertrain, motion, timing, fuel, tyres, race status. Each group is `Group<T> { avail, updated_at, data }`.
- `OpponentPoses`: `Copy`, fixed capacity of 64 cars, published at the source's native rate.
- `SessionInfo`, `ParticipantList`: heap allowed, published only on change, delivered as `Arc`.
- Units: newtypes; conversion to display units only at the UI.

**Consequences.** The UI and analytics must check `Group::avail` before reading. Opponent *names* aren't in any hot path.

**Document changes.** Replace the bundle's §4.2 `NormalizedTelemetryFrame` with the Phase 1 spec §2–3.

**Tests.** `size_of::<HotFrame>()` asserted and recorded; a compile-time check that `HotFrame: Copy`; round-trip serialisation; the unit-conversion table tested against known values (e.g. 100 km/h = 27.7778 m/s; 30 psi = 206.843 kPa).

---

## ADR-03 — Bounded multi-subscriber distribution (A3) — **MODIFY**

**Review proposed:** fan-out with one bounded drop-oldest ring per subscriber.

**Re-evaluation.** Drop-oldest is right for **telemetry frames**: the newest data is what matters and a slow logger must never stall the source. It is **wrong for events**. Silently dropping a `LapCompleted` corrupts lap history. Events need "you fell behind, resynchronise" semantics instead.

**Decision.** Two primitives in one crate:
1. **`FrameBus<T: Copy>`**: a latest-value slot plus per-subscriber bounded queues with drop-oldest. Each delivery carries a sequence number and a `gap_before` count, so a consumer *knows* it missed frames. Publishing never blocks and never allocates after the subscriber list is set up.
2. **`EventBus<E: Clone>`**: per-subscriber bounded queue. On overflow the subscriber is marked **Lagged(n)**: its next receive returns an error with the number of lost events, and it must re-read the current `SessionState` snapshot. The publisher never blocks.

**Consequences.** Lap recorders must mark a lap **incomplete** if any `gap_before > 0` happened during it. Subscriber lists change rarely (copy-on-write).

**Document changes.** Replace the bundle's Module B outputs (`watch` + `rtrb`) with these two types.

**Tests.** Slow-consumer isolation (publisher timing and memory unchanged with a stalled subscriber); drop and gap counting exact under a deterministic schedule; event lag reported, never silent; dropped `Subscription` removed automatically; zero allocations across 10,000 publishes (counting allocator).

---

## ADR-04 — Versioned game protocol identification (A4) — **ACCEPT**

**Re-evaluation.** Confirmed by evidence: F1 25 ships two selectable UDP formats (2025 and 2026 Season Pack) **[Official — EA spec posts]**, and the ACC static page carries a shared-memory version string **[Official — ACC SM doc v1.7.6]**. A sim identity alone isn't enough.

**Decision.** `ProtocolId { sim: SimId, variant: ProtocolVariant }`. Each variant carries the version fields the source itself reports. Decoders declare which variants they support. **Unknown variants are counted and rejected, never decoded on a guess.**

**Consequences.** Recordings store the detected protocol; replay refuses a decoder that doesn't support it.

**Tests.** Header parsing for each supported variant; an unknown version increments `unknown_protocol_versions` and produces no frames; replay of a file recorded with an unsupported variant fails with an explicit error.

---

## ADR-05 — Projection readiness (A5) — **DEFER**

**Re-evaluation.** Replacing the bundle's made-up confidence scalar is still right. But designing the camera types *now*, before any sim has been shown to provide camera data, would freeze an interface built on guesses. Phase 1 needs no camera fields at all.

**Decision.** Defer to the camera feasibility spike (Phase 3, ADR-08). Phase 1 schema contains **no camera fields**. Recorded intent: readiness will be an enum derived from *measured* projection error, never a hand-set number.

**Consequences.** Adding camera data later means a new group in the schema (an additive, versioned change).

**Document changes.** Remove `CameraTelemetryState` from the Phase 1 schema; mark the bundle's §2.4 "Camera Projection & Confidence" section *superseded, pending spike*.

**Tests.** None in Phase 1. The spike defines its own pass/fail (pixel error).

---

## ADR-06 — Communication boundaries (A6) — **MODIFY**

**Review proposed:** drop the loopback WebSocket; use Tauri Channels.

**Re-evaluation.** The review was correct about the localhost risk but conflated two different problems:

1. **Desktop UI ↔ Rust core** (same machine, same app). Should be in-process IPC with no network socket at all. If the stack is Tauri 2, that's Tauri commands + `ipc::Channel`. **[Unknown]** whether the repo is Tauri. GravelToys Studio is Electron, and the audit decides what Race uses.
2. **Android / tablet ↔ desktop** (another device). This *requires* a network transport; Tauri Channels can't serve it. It must work over the **LAN** and over **Tailscale when the user explicitly configures it**.

Treating these as one channel is how unauthenticated localhost endpoints get created. They must be **separate security boundaries** with separate code, separate listeners and separate permission models.

**Decision.**

| | Boundary 1: Desktop IPC | Boundary 2: Remote API |
|---|---|---|
| Peers | The app's own UI window(s) | Paired Android devices / browsers on other machines |
| Transport | In-process IPC (Tauri Channels/commands if Tauri; equivalent if not) | TLS 1.3 HTTPS + WSS |
| Network socket | **None** | Yes, bound only to user-selected interfaces |
| Default | Always on | **Off** until the first device is paired |
| Auth | Implicit (same process) | Pairing (QR + one-time code + desktop confirmation) → per-device credential; **every** endpoint and message authenticated, including telemetry streams and health |
| Commands | Full | Only commands granted per device (e.g. view-only, HUD toggle, PTT) |
| Crate | UI bridge (Phase 8) | `gt-remote-api` (Phase 10) |

**Rules (all [Proposed], to be enforced by tests in Phase 10):**
- No unauthenticated endpoint exists on the remote API. An unauthenticated request gets no data, no version string and no command execution.
- **Binding:** the user chooses interfaces explicitly: LAN adapter(s) and/or the Tailscale interface. Never `0.0.0.0` by default. Tailscale is never assumed; it's an explicit option.
- **Tailscale is a network path, not a trust grant.** Devices on the tailnet still go through pairing and authentication. Certificate pinning still applies.
- The pairing QR may list several candidate addresses (LAN IP, Tailscale IP or MagicDNS name). The pinned certificate fingerprint is the same for all of them.
- The remote API process code has **no access** to Boundary 1 internals beyond read-only bus subscriptions and an explicit, permission-checked command interface.
- The localhost WebSocket from the bundle is **removed**, not just moved.

**Consequences.** Two code paths to maintain, deliberately. Android work (Phase 10) depends on the remote API, not on the desktop UI bridge.

**Document changes.** Replace bundle §5.1 "Local Desktop IPC" and the loopback port; rewrite §5.2/§5.3 auth rows (no "Public" endpoints); add the boundary table to `PHASE0_FINAL_ARCHITECTURE.md`.

**Tests (Phase 8 / 10):** a test that the desktop build opens no listening socket when the remote API is off; unauthenticated, expired-token, revoked-device and wrong-`Origin` requests all rejected; telemetry stream refuses unauthenticated subscription; a device without PTT permission can't trigger PTT; binding only to the selected interfaces (checked by enumerating listeners).

---

## ADR-07 — Wheelbase bridge blocked (A7) — **ACCEPT**

**Re-evaluation.** Still correct. No read interface from GravelToys Wheelbase is known **[Unknown]**. Talking to the OpenFFBoard over USB HID isn't read-only in practice (its configuration protocol sends command reports) **[Unknown — but enough to rule it out]**. Changing the Wheelbase app is forbidden.

**Decision.** Module P stays **BLOCKED**. Unblocking requires *evidence* of an existing interface that is read-only by construction (e.g. a published shared-memory section or a documented pipe). The evidence: source location in the Wheelbase app, and a capture showing data flowing with no writes from the reader side. **No work in GravelToys Wheelbase, its firmware, motor configuration, encoder setup, calibration or FFB settings is part of any phase.**

**Tests (only if unblocked):** isolation test (killing Race leaves the Wheelbase unaffected); static check that only read-only access flags appear in the bridge crate.

---

## ADR-08 — Phase order (A8) — **MODIFY**

**Re-evaluation.** The review put the camera spike right after the first adapter. But if the first adapter is F1 25 (ADR-10), which has **no camera data** **[Official — no camera fields in the F1 UDP spec; re-verify in Phase 2]**, the spike can't run on it. The spike must target whichever sim might provide camera pose (rF2/LMU, or AC+CSP), independent of the first adapter. Also confirmed: racing line v1 = "your best lap as a reference" (no vehicle model needed).

**Decision (phase order):**
1. Core schema, buses, recorder, replay, session lifecycle *(this spec)*
2. First adapter end to end (F1 25 if owned, see ADR-10)
3. Camera feasibility spike: standalone, time-boxed, no product code, on a sim that may expose camera pose. **Runs only if you still want road projection.**
4. Lap storage + session history
5. Track identification + 2D map
6. Deterministic analytics
7. Racing line v1 (best-lap reference) + HUD ribbon; road projection only if the spike passed
8. Desktop UI bridge + HUD/overlay
9. AI Race Engineer (template callouts first, then LLM explanations)
10. Remote API + Android companion (ADR-06 rules as entry criteria)
11. Voice
12. Wheelbase bridge, **only if unblocked** (ADR-07)
13. Further adapters, performance, release

**Change vs. review:** UI bridge moved before AI (the engineer needs a screen); voice after Android (PTT from the phone is cheaper than wake-word on a busy PC).

**Tests.** Each phase gets its own acceptance list before it starts.

---

## ADR-09 — Anti-cheat process hygiene (A9) — **MODIFY (stricter)**

**Re-evaluation.** "Never `OpenProcess` on game processes" is right but incomplete. General process-information libraries may call `OpenProcess`/`NtQueryInformationProcess` internally to read memory or CPU stats for every process **[Unknown — check the specific library before use]**. Also, Phase 1–2 don't need process detection at all: the shared-memory section existing, or UDP packets arriving, *is* the detection.

**Decision.**
- Detect a sim by its official interface only: the named shared-memory section can be opened read-only, or a valid packet arrives.
- Optional process-name detection (later, only for "game launched" UI hints): process **names** from a snapshot only, never handles to game processes. Process-inspection libraries are not allowed for this unless reviewed.
- Forbidden in the whole workspace: `OpenProcess`, `ReadProcessMemory`, `WriteProcessMemory`, `CreateRemoteThread`, `SetWindowsHookEx` for game processes, DLL injection, graphics-API hooking.
- In-game extensions (AC Python, CSP Lua, rF2/LMU plugin DLLs) are separate, opt-in, labelled components, and never used for anti-cheat titles.

**Tests.** A policy test scans workspace sources and lockfile-listed crates for the forbidden API names and fails the build; the list of allowed exceptions is empty.

---

## ADR-10 — First game adapter — **PROPOSED: EA F1 25, conditional on you owning it on PC or a console on the same LAN**

Full comparison: `PHASE0_FINAL_ARCHITECTURE.md` §6.

**Reasoning.** It's the only candidate where both the transport and its versioning are **officially specified by the publisher** **[Official]**. UDP datagrams are trivial to record and replay byte-for-byte, testable on any OS in CI, and fuzzable. Car positions for all cars are in the motion packet **[Official]**. It exercises ADR-04 immediately, since the two formats in F1 25 are a real-world version test.

**Costs, stated honestly.** No camera; no track edges; yearly format churn; it may not be the sim GravelToys users care most about.

**Fallback:** ACC (official shared-memory doc; mature). **Second adapter:** LMU, for its reported track-edge data, once captures verify its undocumented interface.

**If you don't own F1 25, this ADR flips to ACC.** Please confirm.

---

## ADR-11 — Phase 1 scope boundary — **PROPOSED**

Phase 1 adds **new, platform-independent crates only**: no network listener, no UI change, no game adapter, no Windows API calls, no change to existing code beyond registering the new crates in the workspace. This makes Phase 1 fully reversible (delete the crates) and testable on Linux CI.

## ADR-12 — Injectable clock — **PROPOSED**

All time comes from a `Clock` trait. Live runs use a monotonic clock; replay uses recorded timestamps. Without this, replay output can't be byte-identical, and the determinism test (Phase 1 acceptance) is impossible.

## ADR-13 — Stack decision deferred to audit — **PROPOSED**

Whether the desktop shell is Tauri 2, Electron or something else is **[Unknown]**. Phase 1 deliberately doesn't depend on it. If the repo has no Rust workspace, creating one is a separate decision you'll need to approve after the audit.
