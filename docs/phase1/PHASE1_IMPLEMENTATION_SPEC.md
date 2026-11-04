# GravelToys Race — Phase 1 Implementation Specification

**Phase 1 goal:** a tested, platform-independent core that can describe telemetry, distribute it safely, record raw game data, replay it deterministically, and track session lifecycle. **No game adapter, UI, network or Windows code** (ADR-11).

**Date:** 2026-11-04 · **Status:** PROPOSED, awaiting approval · Decisions referenced: `ARCHITECTURE_DECISION_LOG.md`.

Labels: **[Repo]** · **[Official]** · **[Tested]** · **[Proposed]** · **[Unknown]** · **[Third-party]**. Everything in this document is **[Proposed]** unless marked otherwise. Nothing is **[Repo]** or **[Tested]** yet.

---

## 1. Repository audit status and technology stack

| Item | Status |
|---|---|
| Repository access | **UNAVAILABLE.** Only the handoff text and the Phase 0 PDF were provided |
| Desktop shell (Tauri 2 / Electron / other) | **[Unknown]**. The bundle assumes Tauri 2; the existing GravelToys Studio app is Electron. Neither is verified for GravelToys Race |
| Existing Rust workspace | **[Unknown]** |
| Existing telemetry, overlay, DB, networking, tests, CI | **[Unknown]** |

**Consequences for this spec:**
- Every path below is a **proposed new path** relative to the repository root, under a `crates/` directory. **None claims that a file exists.** Final placement is confirmed by the audit (task P1-00).
- If the repo already has a Cargo workspace, the new crates are added as members. If it has none, creating one is a separate approval (ADR-13).
- Phase 1 changes no existing files except the workspace member list (and that only after audit and approval).

**Toolchain [Proposed]:** stable Rust (version pinned via `rust-toolchain.toml` after the audit), edition 2021 unless the repo already uses 2024. Dependency versions are chosen at implementation time and pinned in `Cargo.lock`; none are claimed here.

**Proposed dependencies (Phase 1 only):**

| Crate | Used by | Why |
|---|---|---|
| `serde` (derive) | schema, recorder metadata | Serialisation of snapshots and metadata |
| `thiserror` | all | Typed errors |
| `crossbeam-queue` | bus | Bounded `ArrayQueue` with `force_push` (drop-oldest) |
| `crossbeam-utils` | bus | `Parker`/`Unparker` for blocking receive |
| `arc-swap` | bus | Lock-free read of the subscriber list on publish |
| `parking_lot` | bus | Short critical section for the latest-value slot |
| `crc32fast` | recorder | Per-record integrity check |
| `postcard` | replay | Deterministic encoding for output digests |
| `blake3` | replay | Output digest |
| `serde_json` | recorder | Human-readable metadata block |
| dev: `proptest`, `criterion` | tests | Property tests, benchmarks |

No `tokio`, no Windows crates, no HID, no networking crates in Phase 1.

**Proposed workspace layout:**

```
crates/
  gt-schema/        units, geometry, frames, snapshots, protocol ids, time
  gt-bus/           FrameBus, EventBus, stats
  gt-recorder/      .gtraw format, writer, reader
  gt-replay/        decoder trait, replay runner, digest
  gt-session/       lifecycle state machine, events
  gt-diagnostics/   source health, freshness, aggregated snapshot
  gt-policy-tests/  workspace policy tests (forbidden APIs, dependency rules)
docs/architecture/  these documents + COORDINATES.md
```

---

## 2. Normalised telemetry schema — `gt-schema`

**Responsibility:** the canonical, simulator-independent vocabulary: units, geometry, frames, snapshots, protocol identity, time. A leaf crate with no I/O.

**Files [Proposed]:**
`crates/gt-schema/Cargo.toml`, `src/lib.rs`, `src/units.rs`, `src/geometry.rs`, `src/availability.rs`, `src/frame.rs`, `src/opponents.rs`, `src/snapshot.rs`, `src/protocol.rs`, `src/time.rs`, `tests/units.rs`, `tests/geometry.rs`, `tests/frame_layout.rs`.

**Dependencies:** `serde`, `thiserror`. Nothing else.

### 2.1 Availability

```rust
// src/availability.rs
#[repr(u8)]
#[derive(Clone, Copy, Debug, PartialEq, Eq, Default, serde::Serialize, serde::Deserialize)]
pub enum Availability {
    /// Not provided by this source. `data` must not be read.
    #[default]
    Unavailable = 0,
    /// Provided directly by the game's official interface.
    Native = 1,
    /// Deterministically computed by GravelToys from native values.
    Derived = 2,
    /// From a configuration file or user calibration.
    Configured = 3,
}

/// A group of related values that arrive together from the source.
#[derive(Clone, Copy, Debug, PartialEq, Default, serde::Serialize, serde::Deserialize)]
#[repr(C)]
pub struct Group<T: Copy> {
    pub avail: Availability,
    /// Host time (see `time.rs`) when this group was last updated by the source.
    /// Freshness = now - updated_at. 0 if never updated.
    pub updated_at: HostTime,
    pub data: T,
}

impl<T: Copy> Group<T> {
    /// Returns the data only when available. This is the only accessor consumers should use.
    pub fn get(&self) -> Option<&T>;
}
```

"Stale" is **not** an availability state. Staleness is computed from `updated_at` by consumers or `gt-diagnostics` (§10), because it depends on the consumer's tolerance.

### 2.2 Contracts

- **Input:** none (types only).
- **Output:** types used by every other crate.
- All public structs are documented with units in their field docs.
- No `f32::NAN` is ever stored to mean "missing"; missing = `Availability::Unavailable`.

**Acceptance:** compiles with `#![forbid(unsafe_code)]`; all unit and geometry tests pass (§12).

---

## 3. Hot telemetry frame and slow-changing snapshots

### 3.1 Hot frame (`src/frame.rs`)

`Copy`, fixed size, no heap types (ADR-02). Published on the `FrameBus` at the source's native rate.

```rust
#[derive(Clone, Copy, Debug, PartialEq, serde::Serialize, serde::Deserialize)]
#[repr(C)]
pub struct HotFrame {
    pub header: FrameHeader,
    pub inputs: Group<Inputs>,
    pub powertrain: Group<Powertrain>,
    pub motion: Group<Motion>,
    pub timing: Group<Timing>,
    pub fuel: Group<Fuel>,
    pub tyres: Group<Tyres>,
    pub race: Group<RaceStatus>,
}

#[derive(Clone, Copy, Debug, PartialEq, serde::Serialize, serde::Deserialize)]
#[repr(C)]
pub struct FrameHeader {
    /// Monotonic, assigned by GravelToys per source connection. Starts at 1.
    pub frame_seq: u64,
    /// The source's own counter (e.g. packet frame identifier). Used for torn-read and duplicate checks.
    pub source_seq: u64,
    /// Host receive time.
    pub host_time: HostTime,
    /// The game's session clock, if the source provides one.
    pub sim_time: Group<Seconds64>,
    pub protocol: ProtocolId,
    pub session: SessionUid,
}

#[derive(Clone, Copy, Debug, PartialEq, Default, serde::Serialize, serde::Deserialize)]
#[repr(C)]
pub struct Inputs {
    pub throttle: Unit01,        // 0 = released, 1 = fully pressed
    pub brake: Unit01,
    pub clutch_pedal: Unit01,    // pedal POSITION: 0 = released (ADR-02 / review T12)
    pub steering: SignedUnit,    // -1 = full left .. +1 = full right
    pub steering_wheel_angle: Group<Radians>, // positive = left (ISO 8855 sign), if provided
}

#[derive(Clone, Copy, Debug, PartialEq, Default, serde::Serialize, serde::Deserialize)]
#[repr(C)]
pub struct Powertrain {
    pub speed: MetersPerSecond,  // ground speed magnitude
    pub engine_speed: Rpm,
    pub engine_speed_max: Group<Rpm>,
    pub gear: Gear,              // -1 reverse, 0 neutral, 1.. forward
}

#[derive(Clone, Copy, Debug, PartialEq, Default, serde::Serialize, serde::Deserialize)]
#[repr(C)]
pub struct Motion {
    pub position: Vec3<Meters>,               // world frame (§4)
    pub velocity_world: Vec3<MetersPerSecond>,
    pub orientation: UnitQuat,                // body -> world
    pub velocity_body: Group<Vec3<MetersPerSecond>>,       // ISO 8855
    pub accel_body: Group<Vec3<MetersPerSecond2>>,         // ISO 8855, m/s² (not g)
    pub angular_velocity_body: Group<Vec3<RadiansPerSecond>>,
}

#[derive(Clone, Copy, Debug, PartialEq, Default, serde::Serialize, serde::Deserialize)]
#[repr(C)]
pub struct Timing {
    pub lap_number: u16,                 // as reported by the source; 1-based
    pub lap_distance: Group<Meters>,
    pub lap_fraction: Group<Unit01>,
    pub current_lap_time: Group<Seconds32>,
    pub last_lap_time: Group<Seconds32>,
    pub best_lap_time: Group<Seconds32>,
    pub sector_index: Group<u8>,         // 0-based
    pub lap_valid: Group<bool>,
}

#[derive(Clone, Copy, Debug, PartialEq, serde::Serialize, serde::Deserialize)] // no Default: unit must be explicit
#[repr(C)]
pub struct Fuel {
    pub remaining: FuelAmount,
    pub capacity: Group<FuelAmount>,
}

/// Sims disagree on fuel units; the adapter must say which one it verified.
#[derive(Clone, Copy, Debug, PartialEq, serde::Serialize, serde::Deserialize)]
pub enum FuelAmount { Liters(Liters), Kilograms(Kilograms) }

#[derive(Clone, Copy, Debug, PartialEq, Default, serde::Serialize, serde::Deserialize)]
#[repr(C)]
pub struct Tyres {
    pub pressure: Group<Wheels<Kilopascals>>,
    pub core_temp: Group<Wheels<Celsius>>,
    pub surface_temp: Group<Wheels<[Celsius; 3]>>, // inner, middle, outer
    pub brake_temp: Group<Wheels<Celsius>>,
    pub wear_remaining: Group<Wheels<Unit01>>,     // 1 = new
}

#[derive(Clone, Copy, Debug, PartialEq, Default, serde::Serialize, serde::Deserialize)]
#[repr(C)]
pub struct RaceStatus {
    pub position: Group<u8>,
    pub pit: PitStatus,   // OnTrack | PitLane | PitBox | Unknown
    pub flag: FlagStatus, // None | Green | Yellow | FullCourseYellow | SafetyCar | Red | Chequered | Unknown
}

/// Always ordered FL, FR, RL, RR.
#[derive(Clone, Copy, Debug, PartialEq, Default, serde::Serialize, serde::Deserialize)]
#[repr(C)]
pub struct Wheels<T: Copy> { pub fl: T, pub fr: T, pub rl: T, pub rr: T }
```

### 3.2 Opponent poses (`src/opponents.rs`)

```rust
pub const MAX_CARS: usize = 64;

#[derive(Clone, Copy, Debug, PartialEq, serde::Serialize, serde::Deserialize)]
#[repr(C)]
pub struct OpponentPoses {
    pub header: FrameHeader,
    pub count: u8,                     // valid entries in `cars[..count]`
    pub player_index: Group<u8>,
    pub cars: [OpponentPose; MAX_CARS],
}

#[derive(Clone, Copy, Debug, PartialEq, Default, serde::Serialize, serde::Deserialize)]
#[repr(C)]
pub struct OpponentPose {
    pub car_index: u8,                 // the source's index; stable within a session
    pub position: Group<Vec3<Meters>>,
    pub velocity_world: Group<Vec3<MetersPerSecond>>,
    pub yaw: Group<Radians>,
    pub lap_fraction: Group<Unit01>,
    pub race_position: Group<u8>,
    pub in_pits: Group<bool>,
}

impl OpponentPoses { pub fn active(&self) -> &[OpponentPose]; }
```

Note: serde's built-in array support stops at 32 elements, so `[OpponentPose; 64]` needs a hand-written `Serialize`/`Deserialize` that encodes only `cars[..count]` (which also keeps recordings and digests small). `Group<Fuel>` has no `Default`; constructors build it explicitly.

### 3.3 Slow snapshots (`src/snapshot.rs`)

Heap allowed. Published only when they change, as `Arc<T>` on an `EventBus` (§6).

```rust
#[derive(Clone, Debug, PartialEq, serde::Serialize, serde::Deserialize)]
pub struct SessionInfo {
    pub session: SessionUid,
    pub protocol: ProtocolId,
    pub kind: SessionKind,                  // Practice | Qualifying | Race | TimeTrial | Other(String) | Unknown
    pub track: TrackIdentity,
    pub car: CarIdentity,
    pub total_laps: Option<u16>,
    pub time_limit: Option<Seconds32>,
    pub received_at: HostTime,
}

#[derive(Clone, Debug, PartialEq, serde::Serialize, serde::Deserialize)]
pub struct TrackIdentity {
    /// Exactly what the source reports (string or numeric id rendered as string).
    pub source_key: String,
    pub source_layout: Option<String>,
    pub display_name: Option<String>,
    pub reported_length: Option<Meters>,
}

#[derive(Clone, Debug, PartialEq, serde::Serialize, serde::Deserialize)]
pub struct CarIdentity { pub source_key: String, pub display_name: Option<String>, pub class: Option<String> }

#[derive(Clone, Debug, PartialEq, serde::Serialize, serde::Deserialize)]
pub struct ParticipantList { pub session: SessionUid, pub entries: Vec<Participant>, pub received_at: HostTime }

#[derive(Clone, Debug, PartialEq, serde::Serialize, serde::Deserialize)]
pub struct Participant { pub car_index: u8, pub name: Option<String>, pub team: Option<String>, pub class: Option<String> }

/// Assigned by GravelToys: source-provided session id when available, otherwise a new random id per detected session.
#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash, Default, serde::Serialize, serde::Deserialize)]
#[repr(transparent)]
pub struct SessionUid(pub u64);
```

**Acceptance (§2–3):**
- `HotFrame: Copy + Send + Sync + 'static` and `OpponentPoses: Copy` (compile-time assertions).
- `size_of::<HotFrame>()` and `size_of::<OpponentPoses>()` are asserted in `tests/frame_layout.rs` against the measured value. A change in size must be a deliberate, reviewed edit.
- Serde round trip (postcard) equality for all types.
- `Group::get` returns `None` for `Unavailable`; property-tested.

---

## 4. Coordinate system and unit conventions

**Files:** `crates/gt-schema/src/geometry.rs`, `src/units.rs`, `docs/architecture/COORDINATES.md`.

### 4.1 Frames (ADR-01)

| Frame | Axes | Handedness |
|---|---|---|
| World | X, Y horizontal; **Z up** | Right-handed |
| Vehicle body | ISO 8855: **X forward, Y left, Z up** | Right-handed |
| Angles | Positive rotation counter-clockwise about the axis (right-hand rule); so yaw > 0 = turning left | — |
| Orientation | `UnitQuat` rotating body vectors into world | — |

Each adapter documents its native → GravelToys mapping as a table plus fixture tests (Phase 2).

```rust
// src/geometry.rs
#[derive(Clone, Copy, Debug, PartialEq, Default, serde::Serialize, serde::Deserialize)]
#[repr(C)]
pub struct Vec3<U: Copy> { pub x: U, pub y: U, pub z: U }

/// Unit quaternion (w, x, y, z). Constructors normalise; `from_raw_unchecked` is for tests only.
#[derive(Clone, Copy, Debug, PartialEq, serde::Serialize, serde::Deserialize)]
#[repr(C)]
pub struct UnitQuat { w: f32, x: f32, y: f32, z: f32 }

impl UnitQuat {
    pub const IDENTITY: Self;
    pub fn from_axis_angle(axis: [f32; 3], angle: Radians) -> Result<Self, GeometryError>;
    pub fn from_rotation_matrix(m: [[f32; 3]; 3]) -> Result<Self, GeometryError>;
    pub fn to_rotation_matrix(&self) -> [[f32; 3]; 3];
    pub fn rotate(&self, v: [f32; 3]) -> [f32; 3];
    /// Yaw/pitch/roll in ISO 8855 Z-Y'-X'' order, for display only.
    pub fn to_ypr(&self) -> (Radians, Radians, Radians);
}

#[derive(Debug, thiserror::Error, PartialEq)]
pub enum GeometryError { #[error("zero-length axis")] ZeroAxis, #[error("not a rotation matrix")] NotRotation }
```

### 4.2 Units (`src/units.rs`)

All `#[repr(transparent)]` newtypes over `f32` unless noted, `Copy`, with `const fn new`, `.get()`, and checked constructors from common non-SI inputs.

| Type | Unit | Conversion helpers (inputs from sims) |
|---|---|---|
| `Meters` | m | — |
| `MetersPerSecond` | m/s | `from_kmh`, `from_mph` |
| `MetersPerSecond2` | m/s² | `from_g` (×9.80665) |
| `Radians`, `RadiansPerSecond` | rad, rad/s | `from_degrees` |
| `Celsius` | °C (SI-accepted) | `from_kelvin`, `from_fahrenheit` |
| `Kilopascals` | kPa | `from_psi` (×6.894757), `from_bar` (×100) |
| `Liters`, `Kilograms` | L, kg | — |
| `Seconds32` (f32), `Seconds64` (f64) | s | `from_millis` |
| `Rpm` | rev/min (**named exception**) | — |
| `Unit01` | 0..1, clamped on construction; NaN rejected | — |
| `SignedUnit` | −1..1, clamped; NaN rejected | — |
| `Gear` (i8) | −1, 0, 1..=12 | `try_from(i8)` |

Display-unit conversion (km/h, mph, psi, °F) is **not** in this crate. It belongs to the UI layer.

**Acceptance:**
- Conversion tests: 100 km/h → 27.7778 m/s (±1e-4); 30 psi → 206.843 kPa (±1e-3); 1 g → 9.80665 m/s²; 212 °F → 100 °C.
- Geometry: yaw +90° about Z maps body +X (forward) to world +Y; a left-turning motion gives positive yaw rate; quaternion → matrix → quaternion round trip within 1e-5 (property test over random axes/angles, accounting for q ≡ −q).
- `COORDINATES.md` contains the frame diagrams and an empty per-sim mapping table for Phase 2.

---

## 5. Versioned game protocol identification

**File:** `crates/gt-schema/src/protocol.rs`.

```rust
#[repr(u16)]
#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash, serde::Serialize, serde::Deserialize)]
pub enum SimId {
    F1_25 = 1, Acc = 2, Lmu = 3, Ams2 = 4,
    AssettoCorsa = 5, RFactor2 = 6, IRacing = 7, AcEvo = 8,
    /// Synthetic test data. Never produced by a real adapter; rejected by release builds' UI.
    Synthetic = 0xFFFF,
}

#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash, serde::Serialize, serde::Deserialize)]
pub struct ProtocolId { pub sim: SimId, pub variant: ProtocolVariant }

/// Version information exactly as the source reports it. Adapters add variants in later phases.
#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash, serde::Serialize, serde::Deserialize)]
#[non_exhaustive]
pub enum ProtocolVariant {
    /// F1 UDP: header fields. Field names to be re-verified against the official F1 25 spec in Phase 2.
    F1Udp { packet_format: u16, game_year: u8, game_major: u8, game_minor: u8 },
    /// ACC shared memory: version string from the static page, parsed. [Official doc: smVersion exists]
    AccSharedMemory { sm_major: u16, sm_minor: u16 },
    /// Test fixtures only (SimId::Synthetic).
    Synthetic { version: u16 },
    // Rule: no placeholder variants for adapters that don't exist yet; each adapter adds its own.
}

/// What a decoder declares it can handle.
pub trait SupportsProtocol {
    fn supports(&self, id: &ProtocolId) -> ProtocolSupport;
}

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum ProtocolSupport { Supported, UnknownVersion, WrongSim }
```

**Contract:** an `UnknownVersion` packet is counted (§10) and never decoded. Phase 1 includes only the F1/ACC variants needed by ADR-10 and the synthetic test variant. Variants for LMU/AMS2/etc. are added by their adapters, not pre-declared.

**Acceptance:** unit tests for `supports` with matching, unknown-version and wrong-sim inputs; serde round trip; `ProtocolVariant` is `#[non_exhaustive]`, so adding a variant is non-breaking for downstream `match`es that have a wildcard arm.

---

## 6. Bounded multi-subscriber event architecture — `gt-bus`

**Responsibility:** distribute frames and events to many consumers without ever blocking or (for frames) allocating in the publisher (ADR-03).

**Files [Proposed]:** `crates/gt-bus/Cargo.toml`, `src/lib.rs`, `src/frame_bus.rs`, `src/event_bus.rs`, `src/latest.rs`, `src/stats.rs`, `tests/isolation.rs`, `tests/gaps.rs`, `tests/events_lag.rs`, `tests/alloc.rs`, `benches/publish.rs`.

**Dependencies:** `gt-schema` (for `HostTime` only), `crossbeam-queue`, `crossbeam-utils`, `arc-swap`, `parking_lot`, `thiserror`.

### 6.1 FrameBus (lossy, for `Copy` frames)

```rust
pub fn frame_bus<T: Copy + Send + 'static>(name: &'static str) -> (FramePublisher<T>, FrameBusHandle<T>);

pub struct FramePublisher<T> { /* single producer; !Clone */ }
impl<T: Copy + Send + 'static> FramePublisher<T> {
    /// Never blocks. Never allocates once subscriptions exist.
    /// Writes the latest slot, then force-pushes into every subscriber queue (drop-oldest).
    pub fn publish(&mut self, item: T) -> PublishOutcome;
}
pub struct PublishOutcome { pub seq: u64, pub subscribers: u32, pub dropped_now: u32 }

#[derive(Clone)]
pub struct FrameBusHandle<T> { /* cheap to clone */ }
impl<T: Copy + Send + 'static> FrameBusHandle<T> {
    /// Capacity is fixed at subscribe time. Allocation happens here, not on publish.
    pub fn subscribe(&self, name: &'static str, capacity: core::num::NonZeroUsize) -> FrameSubscription<T>;
    /// Latest value for "show newest" consumers (HUD). Never blocks the publisher for longer than a copy.
    pub fn latest(&self) -> Option<Delivered<T>>;
    pub fn stats(&self) -> BusStats;
}

pub struct FrameSubscription<T> { /* unsubscribes on Drop */ }
impl<T: Copy + Send + 'static> FrameSubscription<T> {
    pub fn try_recv(&self) -> Option<Delivered<T>>;
    pub fn recv_timeout(&self, timeout: core::time::Duration) -> Result<Delivered<T>, RecvError>;
    pub fn stats(&self) -> SubscriberStats;
}

#[derive(Clone, Copy, Debug)]
pub struct Delivered<T> {
    pub seq: u64,          // bus sequence, contiguous at publish
    pub gap_before: u64,   // frames this subscriber missed immediately before this one
    pub item: T,
}

#[derive(Debug, thiserror::Error, PartialEq, Eq)]
pub enum RecvError { #[error("timed out")] Timeout, #[error("publisher dropped")] Closed }
```

Implementation notes [Proposed]: subscriber list in `ArcSwap<Vec<Arc<SubQueue<T>>>>` (copy-on-write on subscribe/unsubscribe only). `SubQueue` = `ArrayQueue<(u64, T)>` + `Unparker` + atomic counters. A subscriber computes `gap_before` from sequence discontinuity, so no extra bookkeeping is needed on publish. The latest slot is a `parking_lot::Mutex<Option<(u64, T)>>` held only for a copy. If benchmarks show contention, replace it with a seqlock in a later phase (an `unsafe` change needing its own review).

### 6.2 EventBus (lag-reporting, for low-rate `Clone` events)

```rust
pub fn event_bus<E: Clone + Send + 'static>(name: &'static str) -> (EventPublisher<E>, EventBusHandle<E>);

impl<E: Clone + Send + 'static> EventPublisher<E> {
    /// Never blocks. If a subscriber is full, that subscriber is marked lagged and its queue is cleared.
    pub fn publish(&mut self, event: E);
}
impl<E: Clone + Send + 'static> EventBusHandle<E> {
    pub fn subscribe(&self, name: &'static str, capacity: core::num::NonZeroUsize) -> EventSubscription<E>;
    pub fn stats(&self) -> BusStats;
}
impl<E: Clone + Send + 'static> EventSubscription<E> {
    pub fn try_recv(&self) -> Result<Option<E>, EventRecvError>;
    pub fn recv_timeout(&self, timeout: core::time::Duration) -> Result<E, EventRecvError>;
}

#[derive(Debug, thiserror::Error, PartialEq, Eq)]
pub enum EventRecvError {
    /// The subscriber missed `missed` events and must resynchronise from a state snapshot.
    #[error("lagged by {missed} events")] Lagged { missed: u64 },
    #[error("timed out")] Timeout,
    #[error("publisher dropped")] Closed,
}
```

Events may allocate (they're rare). The **guarantee** is different: an event is never lost *silently*.

### 6.3 Statistics (`src/stats.rs`)

```rust
#[derive(Clone, Debug, Default, serde::Serialize)]
pub struct BusStats { pub name: &'static str, pub published: u64, pub subscribers: Vec<SubscriberStats> }

#[derive(Clone, Debug, Default, serde::Serialize)]
pub struct SubscriberStats {
    pub name: &'static str,
    pub capacity: usize,
    pub delivered: u64,
    pub dropped: u64,      // FrameBus: overwritten frames. EventBus: events lost to lag.
    pub lag_events: u64,   // EventBus only: number of times Lagged was raised
    pub high_water: usize, // max queue depth observed
}
```

### 6.4 Planned buses (wired in Phase 2; types defined now)

| Bus | Type | Payload | Typical subscriber capacity |
|---|---|---|---|
| `hot` | FrameBus | `HotFrame` | logger 512, analytics 256, UI bridge 4 |
| `opponents` | FrameBus | `OpponentPoses` | 64 |
| `session` | EventBus | `SessionEvent` (§9) | 256 |
| `snapshots` | EventBus | `Arc<SessionInfo>`, `Arc<ParticipantList>` | 16 |
| `diagnostics` | FrameBus-like latest only | `DiagnosticsSnapshot` (§10) | latest only |

**Acceptance:**
- **Isolation:** with one subscriber never reading, publishing 100,000 frames keeps memory constant (queue capacity bound) and p99 publish time within 2× of the no-subscriber baseline (criterion; numbers recorded as the baseline, not promised in advance).
- **Gap accuracy:** deterministic test: capacity 4, publish 10 frames, read once → `gap_before == 6` and the delivered frames are the newest 4.
- **Zero allocation:** counting `GlobalAlloc` in `tests/alloc.rs` shows 0 allocations across 10,000 `publish` calls with 3 subscribers.
- **Event lag:** capacity 2, publish 5 events without reading → next `try_recv` returns `Lagged { missed: 5 }` (queue cleared), then subsequent events are received normally.
- **Unsubscribe:** dropping a subscription removes it; `stats().subscribers` shrinks.
- **Thread safety:** a stress test with 1 publisher and 4 subscriber threads for 5 s shows no lost sequence numbers beyond those reported as `gap_before`/`dropped`.

---

## 7. Raw telemetry recorder — `gt-recorder`

**Responsibility:** record exactly what the game sent (bytes, not normalised frames) so any later decoder change can be re-tested against real data. Recording must **never** slow the source.

**Files [Proposed]:** `crates/gt-recorder/Cargo.toml`, `src/lib.rs`, `src/format.rs`, `src/writer.rs`, `src/reader.rs`, `tests/roundtrip.rs`, `tests/truncation.rs`, `tests/corruption.rs`, `benches/write.rs`.

**Dependencies:** `gt-schema`, `crc32fast`, `serde`, `serde_json`, `thiserror`.

### 7.1 File format `.gtraw`, version 1 (little-endian)

```
FileHeader (fixed 32 bytes)
  magic            [u8; 8]  = b"GTRAW\0\0\x01"
  format_version   u16      = 1
  flags            u16      = 0 (reserved)
  sim_id           u16      (SimId)
  reserved         [u8; 2]
  created_unix_ms  u64      (wall clock, informational only)
  metadata_len     u32      (bytes of the JSON block that follows)
  header_crc32     u32      (over the preceding 28 bytes)
Metadata (metadata_len bytes, UTF-8 JSON) — RecordingMetadata
Records, repeated until EOF:
  record_len       u32      (bytes from `kind` through `payload`, excluding the CRC)
  kind             u8       (1 = Datagram, 2 = MemorySnapshot, 3 = Marker, 4 = SourceEvent)
  channel          u8       (source-defined: e.g. which shared-memory page or UDP port)
  reserved         u16
  host_time_ns     u64      (monotonic, since recording start)
  payload          [u8; record_len - 12]
  record_crc32     u32      (over kind..payload)
```

Maximum `record_len`: 16 MiB (rejects corrupted lengths).

```rust
#[derive(Clone, Debug, serde::Serialize, serde::Deserialize)]
pub struct RecordingMetadata {
    pub app_version: String,
    pub sim: SimId,
    pub detected_protocol: Option<ProtocolId>,
    pub capture_settings: serde_json::Value,   // adapter-specific, e.g. UDP port, poll rate
    pub notes: Option<String>,
    pub contains_other_drivers_names: bool,    // privacy flag (F1 participant packets include names)
}

#[repr(u8)]
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum RecordKind { Datagram = 1, MemorySnapshot = 2, Marker = 3, SourceEvent = 4 }

pub struct RawRecord<'a> { pub kind: RecordKind, pub channel: u8, pub host_time_ns: u64, pub payload: &'a [u8] }
```

### 7.2 Writer

```rust
pub struct RecorderConfig {
    pub directory: std::path::PathBuf,
    pub queue_capacity: core::num::NonZeroUsize, // records buffered between source and disk thread
    pub flush_interval: core::time::Duration,    // default 1 s
    pub fsync_interval: core::time::Duration,    // default 5 s
    pub rotate_at_bytes: u64,                    // default 1 GiB
}

pub struct Recorder { /* owns a background writer thread */ }
impl Recorder {
    pub fn start(cfg: RecorderConfig, meta: RecordingMetadata, clock: std::sync::Arc<dyn Clock>) -> Result<(Recorder, RecordSink), RecorderError>;
    pub fn stop(self) -> Result<RecordingSummary, RecorderError>; // flush + fsync + close
}

#[derive(Clone)]
pub struct RecordSink { /* non-blocking handle given to the capture loop */ }
impl RecordSink {
    /// Copies the bytes into a pooled buffer; NEVER blocks. If the queue is full, the record is dropped and counted.
    pub fn record(&self, kind: RecordKind, channel: u8, payload: &[u8]) -> RecordOutcome;
}
pub enum RecordOutcome { Queued, DroppedQueueFull, Stopped }

pub struct RecordingSummary { pub files: Vec<std::path::PathBuf>, pub records_written: u64, pub records_dropped: u64, pub bytes: u64 }
```

File name: `YYYYMMDD-HHMMSS_<sim>_<n>.gtraw` (local time, `n` increments on rotation). Writes go to `<name>.gtraw.partial` and are renamed on clean stop. A `.partial` file is still readable (§7.3).

### 7.3 Reader

```rust
pub struct RecordingReader { /* buffered file reader */ }
impl RecordingReader {
    pub fn open(path: &std::path::Path) -> Result<Self, ReaderError>;
    pub fn metadata(&self) -> &RecordingMetadata;
    /// Iterates valid records. On a truncated or CRC-failing record, iteration ends and `tail()` reports it.
    pub fn next_record(&mut self) -> Result<Option<RawRecord<'_>>, ReaderError>;
    pub fn tail(&self) -> TailStatus;
}
pub enum TailStatus { Clean, Truncated { valid_records: u64, bytes_discarded: u64 }, CorruptRecord { at_offset: u64, valid_records: u64 } }

#[derive(Debug, thiserror::Error)]
pub enum ReaderError {
    #[error("not a .gtraw file")] BadMagic,
    #[error("unsupported format version {0}")] UnsupportedVersion(u16),
    #[error("header checksum mismatch")] HeaderCrc,
    #[error("metadata invalid: {0}")] Metadata(String),
    #[error(transparent)] Io(#[from] std::io::Error),
}
```

**Acceptance:**
- Round trip: 100,000 random records written and read back byte-identical (property test).
- Truncation: cutting the file at every byte offset in a small fixture never panics; reader returns all complete records before the cut and `TailStatus::Truncated`.
- Corruption: flipping one bit in a record payload ends iteration at that record with `CorruptRecord`.
- Non-blocking: with the disk writer artificially stalled, `record()` returns `DroppedQueueFull` without blocking and `records_dropped` matches.
- Rotation produces multiple files; reader handles each independently.

---

## 8. Deterministic replay harness — `gt-replay`

**Responsibility:** feed a recording through a decoder into the real buses, using recorded time, and prove the output is identical every run.

**Files [Proposed]:** `crates/gt-replay/Cargo.toml`, `src/lib.rs`, `src/decoder.rs`, `src/runner.rs`, `src/digest.rs`, `src/synthetic.rs` (behind `feature = "test-fixtures"`), `tests/determinism.rs`, `tests/unknown_protocol.rs`, `benches/replay.rs`, `bin/gtr-replay.rs` (developer CLI).

**Dependencies:** `gt-schema`, `gt-bus`, `gt-recorder`, `gt-session`, `gt-diagnostics`, `postcard`, `blake3`, `thiserror`.

```rust
/// Implemented by each game adapter's decoding half (Phase 2). Pure: no I/O, no clock.
pub trait RawDecoder: SupportsProtocol + Send {
    fn sim(&self) -> SimId;
    /// Decode one raw record. May emit zero or more outputs into `out`.
    fn decode(&mut self, record: &RawRecord<'_>, out: &mut DecodeOutput) -> Result<(), DecodeError>;
    fn reset(&mut self);
}

/// Reused buffer (no per-record allocation for hot outputs).
pub struct DecodeOutput {
    pub hot: Option<HotFrame>,
    pub opponents: Option<OpponentPoses>,
    pub session_info: Option<SessionInfo>,
    pub participants: Option<ParticipantList>,
    pub protocol_seen: Option<ProtocolId>,
}

#[derive(Debug, thiserror::Error, PartialEq)]
pub enum DecodeError {
    #[error("payload too short: need {need}, got {got}")] TooShort { need: usize, got: usize },
    #[error("unknown protocol version")] UnknownVersion(ProtocolId),
    #[error("malformed: {0}")] Malformed(&'static str),
}

pub enum Pace { AsFastAsPossible, RealTime { speed: f32 } }

pub struct ReplayConfig { pub pace: Pace, pub stop_after: Option<u64> /* records */ }

pub struct ReplayReport {
    pub records_read: u64,
    pub hot_frames: u64,
    pub opponent_frames: u64,
    pub decode_errors: u64,
    pub unknown_versions: u64,
    pub tail: TailStatus,
    pub session_events: u64,
    /// blake3 over the postcard encoding of every published output, in publish order.
    pub output_digest: [u8; 32],
}

pub fn run_replay(
    path: &std::path::Path,
    decoder: &mut dyn RawDecoder,
    sinks: ReplaySinks,     // publishers for hot/opponents/session/snapshots buses
    cfg: ReplayConfig,
) -> Result<ReplayReport, ReplayError>;
```

Determinism rules:
1. The replay clock (`ReplayClock`, §11) returns each record's `host_time_ns`. No wall-clock reads anywhere in decode, session or diagnostics code.
2. The digest covers **outputs**, not timings, and uses `postcard` (stable encoding for the same type layout).
3. `frame_seq` restarts at 1 per replay.

`synthetic.rs` (test-only feature): `SyntheticDecoder` and a fixture generator producing a `.gtraw` with `SimId::Synthetic` (a car lapping a circle, with pit entry, pause and restart). It's **labelled synthetic in metadata** and refused by release builds. It exists only so Phase 1 can be tested before any real adapter.

`gtr-replay` CLI: `gtr-replay <file.gtraw> [--real-time 1.0] [--print-events]` prints the report. Developer tool only.

**Acceptance:**
- Replaying the synthetic fixture twice gives identical `output_digest`; a golden digest is stored in `tests/determinism.rs`.
- Replaying a fixture whose recorded protocol the decoder doesn't support: `unknown_versions > 0`, `hot_frames == 0`, no panic.
- Truncated fixture: replay completes, report shows `TailStatus::Truncated`.
- Throughput benchmark recorded (target to be set from the baseline; no number promised now).

---

## 9. Session lifecycle and event handling — `gt-session`

**Responsibility:** turn connection status and frames into a clear session state and lap/pit events. Sim-agnostic; adapters provide the inputs. Pure state machine (no threads, no clock reads).

**Files [Proposed]:** `crates/gt-session/Cargo.toml`, `src/lib.rs`, `src/state.rs`, `src/events.rs`, `src/laps.rs`, `tests/lifecycle.rs`, `tests/laps.rs`, `tests/edge_cases.rs`.

**Dependencies:** `gt-schema`, `thiserror`.

```rust
pub enum SessionInput<'a> {
    SourceConnected { protocol: ProtocolId },
    SourceDisconnected,
    SessionInfo(&'a SessionInfo),
    Frame { frame: &'a HotFrame, gap_before: u64 },
    /// Periodic tick (e.g. 10 Hz) with current host time, for staleness and pause detection.
    Tick { now: HostTime },
}

#[derive(Clone, Debug, PartialEq, serde::Serialize)]
pub enum SessionPhase {
    Disconnected,
    WaitingForSession,                 // source connected, no session yet
    Active { on_track: TrackPresence },// Garage | PitLane | OnTrack | Unknown
    Paused,                            // packets arrive but sim time doesn't advance (if sim time is available)
    Stale,                             // no new frames within threshold (§10)
    Ended,
}

#[derive(Clone, Debug, PartialEq, serde::Serialize)]
pub enum SessionEvent {
    SourceConnected { protocol: ProtocolId, at: HostTime },
    SourceDisconnected { at: HostTime },
    SessionStarted { session: SessionUid, at: HostTime },
    SessionChanged { from: SessionUid, to: SessionUid, at: HostTime }, // restart, new session, track/car change
    SessionEnded { session: SessionUid, at: HostTime, reason: EndReason },
    LapStarted { session: SessionUid, lap: u16, at: HostTime },
    LapCompleted(LapRecord),
    PitEntered { session: SessionUid, lap: u16, at: HostTime },
    PitExited { session: SessionUid, lap: u16, at: HostTime },
    Paused { at: HostTime },
    Resumed { at: HostTime },
    BecameStale { at: HostTime },
    Recovered { at: HostTime },
}

#[derive(Clone, Debug, PartialEq, serde::Serialize)]
pub struct LapRecord {
    pub session: SessionUid,
    pub lap: u16,
    pub lap_time: Option<Seconds32>,
    pub lap_time_source: Availability,   // Native (game-reported) or Derived (measured)
    pub valid: Option<bool>,             // None if the game doesn't report validity
    pub completeness: LapCompleteness,   // Complete | HadFrameGaps { frames_missed } | StartedMidLap
    pub included_pit_lane: bool,
    pub started_at: HostTime,
    pub ended_at: HostTime,
}

pub struct SessionTracker { /* state */ }
impl SessionTracker {
    pub fn new(cfg: SessionConfig) -> Self;
    /// Appends any resulting events to `out` (caller reuses the Vec).
    pub fn step(&mut self, input: SessionInput<'_>, out: &mut Vec<SessionEvent>);
    pub fn phase(&self) -> &SessionPhase;
    pub fn snapshot(&self) -> SessionStateSnapshot; // for EventBus resync after Lagged
}

pub struct SessionConfig { pub stale_after: Seconds32, pub lost_after: Seconds32 }
```

Rules [Proposed]:
- Lap boundaries come from the **game's lap counter**, not from position. Phase 5 track geometry may refine this, but not replace it.
- Lap time: the game's reported last-lap time if available (`Native`); otherwise measured from frame times (`Derived`).
- The first lap after connecting mid-lap is `StartedMidLap`.
- Any `gap_before > 0` during a lap makes it `HadFrameGaps`. Analytics (Phase 6) may exclude such laps.
- A lap counter going **down** or a `SessionUid` change emits `SessionChanged` and doesn't produce a `LapCompleted`.
- "Esc to garage" / teleport (position jump with lap counter unchanged) ends the current lap as incomplete. The exact detection per sim is defined in Phase 2.

**Acceptance (synthetic inputs, table-driven):** normal laps; connect mid-lap; disconnect mid-lap; session restart (lap counter resets); new session with a new id; pit entry/exit; pause/resume; stale/recover; frame gaps; lap counter jump by >1. For each: exact expected event sequence asserted.

---

## 10. Diagnostics, freshness and drop counters — `gt-diagnostics`

**Responsibility:** one place that answers "is the data healthy, and if not, why?" for the UI, logs and the future AI engineer.

**Files [Proposed]:** `crates/gt-diagnostics/Cargo.toml`, `src/lib.rs`, `src/health.rs`, `src/snapshot.rs`, `tests/health.rs`.

**Dependencies:** `gt-schema`, `gt-bus` (stats types), `serde`.

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq, serde::Serialize)]
pub enum SourceHealth { NotConnected, Waiting, Healthy, Stale, Lost }

#[derive(Clone, Debug, Default, serde::Serialize)]
pub struct SourceCounters {
    pub records_received: u64,
    pub decode_errors: u64,
    pub unknown_protocol_versions: u64,
    pub duplicate_source_seq: u64,
    pub out_of_order_source_seq: u64,
    pub torn_reads_retried: u64,     // shared-memory adapters (Phase 2+)
    pub recorder_dropped: u64,
}

#[derive(Clone, Debug, serde::Serialize)]
pub struct GroupFreshness { pub group: &'static str, pub avail: Availability, pub age_ms: Option<u32> }

#[derive(Clone, Debug, serde::Serialize)]
pub struct DiagnosticsSnapshot {
    pub at: HostTime,
    pub source: SourceHealth,
    pub protocol: Option<ProtocolId>,
    pub last_frame_age_ms: Option<u32>,
    pub measured_rate_hz: Option<f32>,  // over the last 2 s
    pub counters: SourceCounters,
    pub groups: Vec<GroupFreshness>,
    pub buses: Vec<BusStats>,
    pub session_phase: String,
}

pub struct HealthMonitor { /* rolling window */ }
impl HealthMonitor {
    pub fn new(cfg: HealthConfig) -> Self;
    pub fn on_record(&mut self, now: HostTime);
    pub fn on_frame(&mut self, frame: &HotFrame, now: HostTime);
    pub fn counters_mut(&mut self) -> &mut SourceCounters;
    pub fn snapshot(&self, now: HostTime, buses: Vec<BusStats>, phase: &str) -> DiagnosticsSnapshot;
}
pub struct HealthConfig { pub stale_after_ms: u32 /* default 500 */, pub lost_after_ms: u32 /* default 5000 */ }
```

Freshness rule: staleness is based on **new `source_seq` values arriving**, not on an assumed rate (review T9). Defaults (500 ms / 5 s) are **[Proposed]** and per-adapter overridable.

**Acceptance:** health transitions Waiting → Healthy → Stale → Lost → Healthy driven by synthetic timestamps; measured rate within ±2% for a synthetic 100 Hz stream; repeated identical `source_seq` counted as duplicates and not treated as fresh.

---

## 11. Error handling and recovery

**Time (`gt-schema/src/time.rs`, ADR-12):**

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq, PartialOrd, Ord, Default, Hash, serde::Serialize, serde::Deserialize)]
#[repr(transparent)]
pub struct HostTime(pub u64); // nanoseconds since the engine (or recording) started; monotonic

pub trait Clock: Send + Sync { fn now(&self) -> HostTime; }
pub struct MonotonicClock { /* std::time::Instant anchor */ }
pub struct ReplayClock { /* set by the replay runner per record */ }
```

**Policy [Proposed]:**

| Failure | Behaviour | Never |
|---|---|---|
| Malformed packet | Count `decode_errors`, drop it, continue | Panic; guess values |
| Unknown protocol version | Count, report once per session in diagnostics, don't decode | Decode with the nearest known layout |
| Duplicate/out-of-order source seq | Count; drop duplicates; keep out-of-order only if the decoder says it's safe | Treat as fresh |
| Slow frame consumer | Drop-oldest, report `gap_before` | Block the publisher |
| Slow event consumer | `Lagged`, consumer resyncs from snapshot | Lose events silently |
| Recorder queue full | Drop record, count | Block the capture loop |
| Disk full / I/O error while recording | Stop recording, keep the `.partial` file readable, raise diagnostic; telemetry continues | Stop telemetry |
| Corrupt recording | Reader returns all valid records + `TailStatus` | Panic |
| Consumer thread panic | Its subscription is dropped (auto-unsubscribe); publisher unaffected; diagnostic logged | Propagate to the publisher |
| Source disappears | `SourceDisconnected` → session ends with reason; reconnect starts a new connection with `frame_seq` reset | Reuse the old session silently |

All crates: `#![forbid(unsafe_code)]` in Phase 1. Public errors are `thiserror` enums; no `unwrap()`/`expect()` outside tests (enforced by `clippy::unwrap_used`, `clippy::expect_used` = deny in library code).

---

## 12. Tests

### 12.1 Unit and property tests
Per crate as listed in §2–10. Property tests (proptest): unit clamping, quaternion round trips, recorder round trip, bus sequence/gap accounting.

### 12.2 Integration tests
- `gt-replay/tests/determinism.rs`: synthetic recording → replay → buses → session tracker → digest equals golden; run twice.
- `gt-replay/tests/end_to_end.rs`: synthetic recording with pause, restart, pit stop and a deliberate frame gap → exact `SessionEvent` sequence and diagnostics counters.
- `gt-recorder/tests/truncation.rs`: cut at every offset.

### 12.3 Performance tests (criterion; baselines recorded, regressions > 20% fail review)
- `gt-bus/benches/publish.rs`: publish `HotFrame` to 0/1/4/8 subscribers.
- `gt-recorder/benches/write.rs`: record 1,500-byte datagrams at max rate.
- `gt-replay/benches/replay.rs`: replay throughput (records/s).
- `gt-bus/tests/alloc.rs`: zero allocations on publish (a test, not a bench).

No absolute performance targets are promised before baselines exist (review §4).

### 12.4 Policy tests — `gt-policy-tests`
**Files:** `crates/gt-policy-tests/Cargo.toml`, `tests/forbidden_apis.rs`, `tests/dependency_rules.rs`.
- `forbidden_apis.rs`: scans `crates/**/*.rs` for `OpenProcess`, `ReadProcessMemory`, `WriteProcessMemory`, `CreateRemoteThread`, `SetWindowsHookEx`, `VirtualAllocEx`, `LoadLibrary` + injection patterns, `hidapi`, `HidD_Set`; any hit fails (ADR-09, ADR-07).
- `dependency_rules.rs`: reads `Cargo.lock` / `cargo metadata` and fails if Phase 1 crates depend on networking, HID, Windows or process-inspection crates (allowlist approach).

### 12.5 Commands

```bash
cargo fmt --all -- --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
cargo test -p gt-replay --features test-fixtures
cargo bench -p gt-bus -p gt-recorder -p gt-replay   # records baselines
```

All must pass on Linux and Windows (Phase 1 is platform-independent).

---

## 13. Phase 1 overall acceptance criteria

1. Audit completed and this spec's paths confirmed or corrected (P1-00).
2. All crates in §1 exist, build, and pass §12 on Linux and Windows.
3. Golden replay digest stable across two machines.
4. Zero-allocation publish test passes.
5. Policy tests pass; no forbidden APIs or dependencies.
6. No changes to existing code beyond workspace membership; no network listener; no UI change; no wheelbase code.
7. `COORDINATES.md` and these documents updated to match what was built (any deviation recorded in the decision log).

## 14. Known limitations of Phase 1

- No real game data: only synthetic fixtures until Phase 2. Phase 1 proves the *plumbing*, not telemetry accuracy.
- Per-sim lap edge cases (teleport detection, rewinds) are only generically handled until adapters exist.
- Latest-value slot uses a short lock; may need optimisation after benchmarks.
- Recording files are uncompressed (compression deferred).

## 15. Rollback

Phase 1 only adds new crates. Rollback = remove the crates and their workspace entries in one commit; nothing else depends on them yet.
