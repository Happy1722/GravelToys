# GravelToys Race — Phase 0 Architecture Review

**Reviewed:** `GravelToys Race Phase 0 Architecture Bundle.pdf` (46 pages, 8 documents)
**Date:** 2026-11-04
**Repository status:** NOT PROVIDED. Nothing in this review is verified against code. Every repository-dependent statement is marked *Unknown / needs validation*.

## Evidence labels

| Label | Meaning |
|---|---|
| **[Repo]** | Verified in repository: none yet, because no repository was supplied |
| **[Official]** | Verified from official documentation (source given) |
| **[Tested]** | Tested: none yet |
| **[Proposed]** | A recommendation from this review |
| **[Unknown]** | Unknown / needs validation |
| **[Third-party]** | Reported by third-party software or community sources; plausible but not authoritative |

---

## 1. Summary verdict

The bundle has a strong *direction*: deterministic maths separate from the LLM, explicit field availability, no DLL injection, a read-only wheelbase link and bounded queues. All of these are correct instincts and should stay.

But as an *implementation specification* it is not ready. It contains:

1. **Hard technical errors.** The coordinate system it calls right-handed is actually left-handed, it uses invalid Rust syntax, it uses a single-consumer queue for many consumers, and its "zero allocations per frame" goal contradicts its own data model.
2. **Internal contradictions.** It gives three different camera-confidence thresholds, uses G-units in a "strict SI" schema, and plans all seven sims in Phase 2 while the handoff asks for one.
3. **Compatibility claims stated as "Verified" with no evidence.** Several are wrong or outdated (details in `GAME_COMPATIBILITY_EVIDENCE.md`).
4. **One rule conflict that blocks a whole module.** The Wheelbase Bridge needs an IPC interface that the Wheelbase app must already expose. If it doesn't, building the bridge would mean modifying the Wheelbase app, which your rules forbid.
5. **A headline feature with no proof of feasibility.** Road-projected racing lines need per-frame camera pose + FOV + low-latency compositing. No sim in the matrix has been shown to provide all three through an approved interface.
6. **Many performance numbers that can't be checked.** The PDF's unit symbols (µs, ms, m) were lost when it was exported, e.g. "latency < 150 ;" and "< 10 at 60Hz". So the numbers are ambiguous as written.

**Recommendation [Proposed]:** approve the *principles*, reject the bundle as a contract, and re-baseline each module's interface when its phase starts, against real captured telemetry.

---

## 2. Technically incorrect assumptions

| # | Bundle claim | Problem | What it means for the product | Fix [Proposed] |
|---|---|---|---|---|
| T1 | "Right-handed: X = Right, Y = Up, Z = Forward" (§2.3 Module C, §4.1) | X-right, Y-up, Z-forward is **left-handed**. Right-handed with X-right, Y-up has Z pointing *backward* (the OpenGL convention). | Every adapter conversion, cross product, yaw sign and the camera projection would be mirrored. The racing line would curve the wrong way. | Pick one convention and test it with a fixture: a known heading plus a left turn must give a positive yaw rate. Suggest X-right, Y-up, Z-back (right-handed), with "forward = −Z". Or keep Z-forward and call it left-handed. |
| T2 | `pub struct TelemetryEngineHandle { … pub fn subscribe_ring(…) }` | You can't declare a method inside a struct body in Rust. It won't compile. | Cosmetic, but it shows the interfaces were never compiled. | Put methods in an `impl` block. Compile every interface in a scratch crate before approving it. |
| T3 | `rtrb::Consumer` for "sequential consumers (Lap Logger, Analytics)" + `subscribe_ring()` | `rtrb` is a **single-producer, single-consumer** ring. One ring can't feed several subscribers. | Either the second consumer silently gets nothing, or someone adds a lock and breaks the real-time guarantee. | Use a fan-out bus with one bounded ring per subscriber, drop-oldest, and a lag counter per subscriber. |
| T4 | "Zero heap allocations per frame" | The frame contains `Vec<OpponentVehicleState>`, `Option<String>` (tyre compound) and `String` (adapter version), and it's published as `Arc<NormalizedTelemetryFrame>` (a new `Arc` allocates every tick). | You can't meet that acceptance test with this schema. | Split into a fixed-size **hot frame** (inputs, kinematics, timing) and slow-changing **session/opponent snapshots** published at 5–20 Hz. Measure allocations with a counting allocator in tests, and don't promise zero until it has been measured. |
| T5 | "Strict SI units everywhere" but `local_accel_g` in multiples of g | Contradiction. | Unit bugs at the analytics boundary. | Store m/s²; convert to g in the UI. Use newtypes (`MetersPerSec2`). |
| T6 | `orientation_euler_rad: [Pitch, Yaw, Roll]` used for camera projection | The rotation order isn't defined. Euler angles also break down (gimbal lock) when the car is pitched steeply, e.g. at the Eau Rouge crest. Each sim uses its own order. | The projected line wobbles or flips. | Store orientation as a unit quaternion or 3×3 rotation matrix. Each adapter converts from its native form, with a per-sim fixture test. |
| T7 | `SimId::F1Series` as one value | F1 UDP formats change every year. F1 25 now ships **two** formats (2025 and "2026 Season Pack"), chosen in the game menu **[Official — EA F1 25 2026 Season Pack UDP spec post]**. | The wrong parser gets used without any error. | `SimId` + `ProtocolVersion` (e.g. `F1 { packet_format: u16 }`); reject unknown formats loudly. |
| T8 | iRacing "360Hz" option implied as higher fidelity | The 360Hz option is 60Hz delivery with 6 samples per cycle, not a true 360Hz live stream **[Third-party — SimXperience staff answer]**. | Polling at 360Hz wastes CPU and gains nothing. | iRacing adapter waits on the SDK's data-valid event and reads at 60Hz. **[Unknown]** until checked against the SDK headers. |
| T9 | ACC "333Hz (SM)" | ACC's own doc says physics members "change at each graphic step" **[Official — ACC Shared Memory v1.7.6 doc]**. So the rate follows the game's frame rate; it isn't a fixed 333Hz. | Freshness logic built on a fixed rate will raise false "stale" alarms. | Base freshness on the *packet counter changing*, not on an assumed rate. |
| T10 | ACC fuel in litres | The ACC doc itself labels `fuel` as "kg" **[Official — ACC doc v1.7.6]**. The community widely reports it's actually litres **[Unknown]**. | Fuel-to-finish could be off by ~25–35% if the unit is wrong. | Confirm with a capture: fill a known number of litres and read the value. Record the result in the adapter's unit table. |
| T11 | `host_timestamp_ns` from `Instant` | `Instant` has no defined epoch and can't be turned into nanoseconds you could store and compare later. | Recordings from two runs can't be lined up. | Store the time since engine start plus one wall-clock anchor per session. |
| T12 | `clutch` "1.0 = fully engaged / foot off" | Sims disagree on whether clutch is pedal position or engagement. | Analytics shows the pedal backwards. | Field name `clutch_pedal_norm` (0 = released); adapters must convert. |
| T13 | rF2 "mFOV exposed in rF2GraphicsInfo!" | Third-party documentation of the GraphicsInfo struct lists camera position, orientation, ambient light and camera type, with **no FOV field** **[Third-party — rF2SharedMemoryNet API docs]**. | Tier-1 projection for rF2/LMU loses its main justification. | Treat rF2/LMU FOV as **[Unknown]** until checked against ISI's `InternalsPlugin.hpp`. A config-file FOV would make it Tier 2, not Tier 1. |
| T14 | AC "ac.getCameraMatrix" via the official Python app API | I could not find this function in the official AC Python documentation. Camera access in AC is normally done through Custom Shaders Patch (CSP) Lua, which is third-party. **[Unknown]** | AC Tier 1 depends on a third-party patch and code running *inside* the game process. | Treat it as optional and CSP-dependent. Anything that runs inside the game process must be the user's choice and clearly labelled. |
| T15 | rF2 shared-memory plugin described as "official Internals Plugin, fully supported by Studio 397" | `rF2SharedMemoryMapPlugin64.dll` is TheIronWolf's community plugin, built on the official plugin interface **[Third-party]**. | A support or licensing assumption that may be wrong. | Describe it accurately; check the plugin's licence before bundling it. |
| T16 | LMU uses an rF2-derived plugin | LMU now publishes its own `LMU_Data` shared memory when **Settings → Gameplay → Enable Plugins** is on, plus a local REST API on port 6397 **[Third-party — LMU Pitwall, NeoPedal docs]**. The REST API reportedly provides **official track layout and track edges** **[Third-party]**. | Good news. LMU may be the **only** sim with real track boundaries, which matter for the track map and racing line. | Re-evaluate LMU as the first adapter (see §7). |

## 3. Conflicting interfaces, types and dependencies

| # | Conflict | Resolution [Proposed] |
|---|---|---|
| C1 | Projection threshold: §2.4 says Tier 1 ≥ 0.90 and Tier 2 ≥ 0.75. §4.2 says the renderer needs ≥ 0.85. RISK-01 says < 0.85 disables. | Remove the made-up scalar. Use an explicit `ProjectionReadiness` enum based on measured pixel error (see §5, camera). |
| C2 | `CameraTelemetryState` lives inside the telemetry frame, but Module I takes a separate `ValidatedCameraMatrix`. | The camera estimator is its own module; the frame carries only the raw camera inputs the sim provides. |
| C3 | Module E (Session) depends on Module F (Track), and F's learning mode needs lap events from E. | Circular dependency. Make lap/sector detection use only the sim's lap counters at first; track learning subscribes to session events. |
| C4 | Two local transports: a loopback WebSocket `ws://127.0.0.1:48210` *or* Tauri v2 `ipc::Channel`. | Pick Tauri Channels (no open port). Drop the loopback socket (security, see §6). **[Unknown]** whether the repo uses Tauri at all. |
| C5 | Stack assumption: Tauri 2. The existing GravelToys Studio is described as **Electron**. | **[Unknown]**. The audit must show what GravelToys Race actually uses. Don't migrate frameworks inside Phase 1. |
| C6 | Roadmap Phase 2 = all 7 adapters; Phase 3 = track ID "across all 7 sims"; the handoff says one adapter first. | Follow the handoff: one adapter, proven end to end, before any expansion. |
| C7 | `OpponentVehicleState.driver_name: [u8; 32]` "to avoid heap allocs", but the frame holds a `Vec` of them anyway. | Covered by T4 (split the frame). |
| C8 | AI Engineer `TelemetryContextSummary` "~600 tokens max", but the example payload includes opponent names and free-text causes. | Fine as a target; enforce it with a test that builds the context and asserts a token budget. |

## 4. Unrealistic or unverifiable performance targets

All of these are **[Unknown]** until measured. Several have units missing from the PDF.

| Target | Concern | Proposed approach |
|---|---|---|
| Normalisation < 150 (unit lost) | Probably µs. Easy to hit, but meaningless without a benchmark. | Benchmark with `criterion` on recorded frames; set a budget relative to the measured baseline. |
| 3D overlay GPU < 0.8 ms, 60–144 Hz | Plausible for drawing, but the real issue is **latency**: a separate DWM-composited window is typically 1+ frames behind the game. | Measure the end-to-end error in pixels during the camera spike before setting any target. |
| ARLI local horizon < 3.5 ms at 30–60 Hz, global line < 250 ms | Minimum-curvature optimisation on a full track in 250 ms is possible. A full minimum-time line with a friction-circle model is research-grade. | Defer. Start with "reference line = your own best lap" (see §5). |
| Whisper transcript < 450 ms, Piper first audio < 150 ms, while a sim is running | Depends heavily on the CPU and on the game's load. | Measure on the user's actual rig while driving; until then, mark it best effort. |
| Android 60 fps and < 8% battery per hour | Untested; depends on the device. | Set a target only after the first build. |
| "Clean scan under EasyAntiCheat / EA AntiCheat" (Phase 11) | There is no public anti-cheat "scan" you can pass. You can only follow the rules (official interfaces only, no injection, no process handles). | Replace with a checklist: no `OpenProcess` on game PIDs, no DLLs, only official shared memory/UDP. Verify by code review and by running with each game. |
| 12-hour soak with zero leaks | Achievable as a test, but it needs replay fixtures first. | Keep it, after the replay harness exists. |

## 5. Areas you asked me to focus on

### 5.1 Camera pose and FOV for road-projected lines

To draw a line on the asphalt you need, **every frame**: camera position, camera orientation, the FOV *and aspect/projection the game actually uses*, the render resolution and window rectangle, and a latency close enough to the game's frame. Missing any one makes the line "swim".

- **No sim in the matrix is verified to provide all of these through an approved interface.** rF2 provides camera position and orientation (third-party docs) but no FOV was found. F1 provides car pose for all cars **[Official — F1 UDP spec motion packet]** but no camera. iRacing, ACC and AMS2 provide no live camera.
- "Tier 2: cockpit-locked camera from car pose + config offset" ignores the head-movement effects most sims apply in the cockpit view (look-to-apex, g-force head motion, seat vibration). With those switched off it might be acceptable. **[Unknown]**
- Overlay latency: an external window composited by DWM over a borderless game is generally at least one frame behind. At 250 km/h (~70 m/s) and 60 fps, one frame is ~1.2 m of travel; in a fast corner the yaw change alone moves the line visibly. **[Unknown — needs measurement]**
- **VR:** a desktop overlay window is not visible inside a VR headset at all. Showing anything in VR would need an OpenXR API layer, which is a separate, larger project. The bundle doesn't mention VR. **[Proposed: declare VR out of scope for road projection.]**
- **Exclusive fullscreen:** external overlays don't show. The bundle covers this correctly (RISK-03).

**Proposed:** a time-boxed **camera feasibility spike** (≤ 1 week) on the single best-candidate sim, with one measurable result: the pixel error of a projected known point (e.g. a start/finish line marker) at standstill, at constant speed and through a corner. If the error is > ~1% of screen height, ship the HUD perspective ribbon and 2D map as the product, and stop there for that sim. Never promise road projection in marketing until the spike passes.

### 5.2 Track geometry and layout identification

- Most sims expose a track **name/ID** and **length**, and the car's **world position**. None of the main telemetry APIs expose **track edges**. The bundle's "left/right boundary" and "track width" fields have no source. **[Unknown]**
- Learning edges from your own driving only gives *the corridor you drove*, not the real edges.
- Possible real sources: LMU REST API track edges **[Third-party]**; AC's on-disk AI spline files (in the game install, read-only) **[Unknown]**; manual mapping. The "142 tracks" database in the UI mock-up has no source and raises licensing questions. **[Proposed: remove from the spec.]**
- Layout identification: use sim-provided track ID + layout + measured lap length as the key. Use position matching only as a sanity check.
- Bridges/overpasses (Suzuka): the elevation-gated projection the bundle proposes is the right idea. Keep it as a test case for Phase 4.

### 5.3 Telemetry units, timestamps, freshness and unavailable fields

- The availability idea (`TelemetryField<T>` with `ValueSource`) is good. **Keep it.** But putting it on every field roughly doubles the frame size. Apply it per *group* (tyres, fuel, kinematics) and keep the frequently updated scalars plain, with a group-level availability flag.
- Every adapter needs a **unit table** checked by fixtures: for each source field, its native unit, conversion and capture evidence.
- Freshness: mark data stale when *the source counter stops changing*, not by wall-clock rate. Separate cases: "game paused", "game in menu", "game closed" and "adapter crashed".
- Timestamps: record the sim's session time, the host's monotonic receive time and the sim's packet counter. Analytics uses sim time; latency diagnostics use host time.

### 5.4 Bounded queues and slow-consumer isolation

The principle is right (no unbounded channels). Concrete design:
- Latest value (HUD): a `watch`-style single slot. Readers always get the newest frame and never block the producer.
- Sequential (logger, analytics): one bounded ring per subscriber, drop-oldest, with a **drop counter visible in diagnostics**. A lap logger that drops frames must mark that lap "incomplete" and not save it as valid.
- Network clients: the bundle's "drop client after 32 frames backlog" is sensible. Keep it.
- **Test:** a deliberately slow consumer must not change the producer's timing or memory use.

### 5.5 Tauri / Rust / React overhead

- The advice (React state at ≤ 10 Hz, canvas for high-rate widgets, binary payloads) is correct. **Keep it.**
- Send to the UI at a **fixed UI rate** (e.g. 30–60 Hz, throttled in Rust), not at sim rate.
- **[Unknown]** until the repo is audited: whether it's Tauri at all, and how the current UI receives data.

### 5.6 Native transparent overlay limitations

- Overlays work only in borderless/windowed mode; not in exclusive fullscreen, and not in VR.
- Click-through + top-most windows can lose top-most status when the game takes focus; the overlay has to re-assert it. Multi-monitor and DPI scaling need explicit handling.
- A second native wgpu window *plus* WebView overlays is two rendering stacks. **[Proposed]** For the MVP, use one approach (WebView transparent window for 2D HUD) and add native rendering only if the camera spike succeeds.
- Overlays can affect the game's presentation mode and add slight input latency to the game. **[Unknown]** Measure it with and without the overlay.

### 5.7 Android pairing and remote access security

The design (QR + one-time code + TLS cert pinning) is a good base. Gaps:
1. **The local loopback WebSocket has no authentication.** Any program on the PC, and potentially any **web page** in the user's browser, can connect to `ws://127.0.0.1:…`. Browsers allow cross-origin WebSockets to localhost. If command messages (PTT, HUD toggles) are accepted there, a malicious page could send them. → Use Tauri Channels; if a local socket must exist, require a token and check the `Origin` header.
2. Long-lived bearer tokens with no expiry or rotation. → Short-lived access tokens + a refresh token bound to the device; a revoke-all control.
3. The QR code appears on screen. Streamers often show their desktop. → Short OTC lifetime (already 60 s), single use, plus a confirm prompt on the desktop ("Pixel 8 wants to pair — Allow?").
4. LAN server off by default; on only when a device is paired; bind to the LAN interface only; Windows firewall prompt explained to the user.
5. `/system/health` "public" leaks the version. Return only "ok" without auth.
6. Remote commands need per-device permissions (the UI spec already shows toggles; enforce them on the server).
7. Rate-limit pairing attempts.

### 5.8 Wheelbase process isolation

- **Blocking conflict.** Module P reads `\\.\pipe\graveltoys_wheelbase_telemetry_v1` or `Local\GravelToysWheelbaseReadOnly`. **Neither is known to exist** **[Unknown]**. If the existing Wheelbase app (GravelToys Studio, Electron, talking to an OpenFFBoard) doesn't already publish data, the only way to get it is to change that app, which your rules forbid.
- Reading the OpenFFBoard **directly over USB HID is not safe to call "read-only"**: its configuration protocol works by sending command reports, and opening the device could interfere with the Wheelbase app. **[Unknown]** Reject this path.
- **Proposed:** Module P stays **blocked** until you confirm whether the Wheelbase app already exposes any read interface. If it doesn't, the options are (a) drop the feature, or (b) you separately approve a small, read-only *publisher* change in the Wheelbase app, as its own decision and outside this project.
- The isolation test the bundle proposes (kill Race → wheelbase unaffected) is good. Keep it.

## 6. Security and anti-cheat risks

- **Good:** ban on DLL injection, DXGI hooks and `ReadProcessMemory`. Keep it as a CI-enforced rule (forbid `OpenProcess`/`ReadProcessMemory`/`CreateRemoteThread` imports; lint for them).
- **Process detection:** the game detector uses `CreateToolhelp32Snapshot` to read process names, which is generally harmless. But **never call `OpenProcess`** on game PIDs, even read-only. The bundle's mention of "permission denied on OpenProcess" implies it would try. → Detect by process name and by shared-memory/UDP presence only. **[Proposed]**
- **In-game extensions (AC Python apps, CSP Lua, rF2 plugin DLLs):** these run *inside* the game process. They're allowed by those games' own mod systems, but they're a different risk class. They must be optional, opt-in and clearly labelled, and never used for iRacing, F1 or other anti-cheat titles.
- **Online racing:** some leagues and services restrict overlays or driving aids. A live "braking colour" racing line could count as a driving aid. **[Unknown]** → Add an "aids disabled in online sessions" policy setting.
- **Local LLM cloud keys:** store in the Windows Credential Manager, not SQLite.

## 7. Recommended phase order (justified)

| # | Phase | Change vs. suggested order | Why |
|---|---|---|---|
| 1 | Core schema, event bus, **raw recorder + replay harness** | Recorder added | Every later phase needs repeatable data. Capturing raw sim data to disk first means everything else can be tested off-track and in CI. |
| 2 | One adapter, end to end, with captured fixtures | Same | Choose by **which sims you own and drive**. Technical ranking: **LMU** (native shared memory, REST track edges, but Windows-only testing) or **F1 25** (official UDP spec, easiest to fuzz and test on any OS, but needs year-versioning). ACC is a solid third. |
| 2b | **Camera feasibility spike** (time-boxed, no product code) | Moved earlier, split off | Road projection is the riskiest promise. Learn early whether it's possible, before building the solver and renderer. |
| 3 | Session detection + lap recording + storage | Same | |
| 4 | Track identification + 2D map | Same | |
| 5 | Deterministic analytics (delta, sectors, fuel, corner losses) | Same | |
| 6 | Racing line, **v1 = "reference line from your best lap"** + HUD ribbon; road projection only if 2b passed | Scope reduced | The bundle's five physics modes (Attack/Defend/Side-by-Side/Wet) need a vehicle model the sims don't provide. Defer them. |
| 7 | HUD + overlay polish | Same | |
| 8 | AI Race Engineer — **template callouts first**, LLM second | Order within phase | Template radio works offline and can't hallucinate. The LLM only *explains*. |
| 9 | Voice | Same | |
| 10 | Android companion | Same | Security items from §5.7 are entry criteria. |
| 11 | Wheelbase bridge | **Blocked** | See §5.8. |
| 12 | Compatibility expansion, performance, release | Same | Add AC Evo (now has documented shared memory **[Third-party / Kunos doc]**), which is missing from the bundle. |

## 8. Features to defer

- ARLI Attack / Defend / Side-by-Side / Wet modes (phase 12+, research).
- Native wgpu road renderer (only after a successful spike).
- Pre-mapped "142 track" database.
- Cloud LLM providers (local/template first).
- Wake-word voice (PTT only).
- Android remote commands beyond view-only (until §5.7 is done).
- iRacing / F1 road projection: no camera data, so these are HUD-only, permanently, unless their APIs change.

## 9. Missing tests the bundle should add

1. Coordinate-convention fixtures per sim (straight line, left turn, climb).
2. Unit fixtures per adapter (speed, fuel, pressure, temperature) from real captures.
3. Torn-read test: a writer thread mutating a buffer while the reader validates the sequence counter.
4. Slow-consumer isolation test (§5.4).
5. Allocation-count test on the hot path.
6. Recorder → replay round trip produces byte-identical normalised output (determinism).
7. Lap logic under: Esc-to-pits, session restart, rewind (F1), lap invalidation, crossing the line in the pit lane.
8. Security: unauthenticated WS rejected, wrong `Origin` rejected, expired OTC rejected, revoked device rejected.
9. Static check: no forbidden Win32 imports (anti-cheat rule).
10. AI: every number in an engineer message is traceable to a context field (the bundle's "50 adversarial prompts" test is good; keep it).

## 10. Architecture changes needed before implementation (require your approval)

1. **A1** Fix the coordinate convention (T1).
2. **A2** Split the frame into hot frame + slow snapshots; orientation as quaternion; SI units only (T4–T6).
3. **A3** Fan-out bus with per-subscriber bounded rings (T3).
4. **A4** Versioned protocol IDs per sim (T7).
5. **A5** Replace the confidence scalar with measured `ProjectionReadiness` (C1, §5.1).
6. **A6** Drop the loopback WebSocket; use in-process channels to the UI (C4, §5.7).
7. **A7** Wheelbase bridge blocked pending your answer (§5.8).
8. **A8** Phase order per §7 (recorder in Phase 1; camera spike as 2b; racing line v1 = best-lap reference).
9. **A9** Game detector must never call `OpenProcess` on game processes (§6).
