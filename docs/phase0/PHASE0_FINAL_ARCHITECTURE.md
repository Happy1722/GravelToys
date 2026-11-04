# GravelToys Race — Phase 0 Final Architecture Baseline

**Date:** 2026-11-04 · **Status:** PROPOSED, awaiting approval.
**Supersedes:** `GravelToys Race Phase 0 Architecture Bundle.pdf` wherever they conflict. The bundle is kept as a vision document only.
**Repository access: UNAVAILABLE.** No section is repository-verified. All file paths in this and the Phase 1 documents are **proposed new paths**, not claims about existing files.

Labels: **[Repo]** none yet · **[Official]** officially documented · **[Tested]** none yet · **[Proposed]** · **[Unknown]** · **[Third-party]** community source, not authoritative.

---

## 1. Principles (kept from the bundle, all [Proposed])

1. Simulator stability beats every overlay feature.
2. Only official game interfaces (shared memory, UDP, published plugin APIs). No injection, hooks or process-memory access (ADR-09).
3. Deterministic Rust code does all the maths; any LLM only explains numbers it was given.
4. Every value says whether it's available and how fresh it is; missing data is shown as missing, never guessed.
5. No unbounded queues; a slow consumer can never slow the source.
6. Everything that matters can be recorded and replayed deterministically.
7. The GravelToys Wheelbase app is off-limits: no modification, ever. Integration only through a verified existing read-only interface (ADR-07).

## 2. Decisions

All decisions with reasoning and tests: `ARCHITECTURE_DECISION_LOG.md` (ADR-01 … ADR-13). Short form:

| Area | Baseline |
|---|---|
| Coordinates | World: right-handed, Z-up. Vehicle: ISO 8855 (X fwd, Y left, Z up). Quaternion orientation (ADR-01) |
| Data model | `Copy` hot frame with per-group availability; opponent poses fixed-capacity; slow snapshots as `Arc`, on change only (ADR-02) |
| Distribution | `FrameBus` (lossy, drop-oldest, gap-reporting) + `EventBus` (lag-reporting, never silent) (ADR-03) |
| Protocols | `ProtocolId` with source-reported versions; unknown versions rejected (ADR-04) |
| Camera | Deferred to feasibility spike; not in Phase 1 (ADR-05) |
| Communication | Two separate security boundaries (§4) (ADR-06) |
| Wheelbase | Blocked (§7) (ADR-07) |
| Anti-cheat | Interface-presence detection; forbidden-API policy test (ADR-09) |
| First adapter | F1 25 if owned, else ACC (§6) (ADR-10) |

## 3. Module map and status

| Module | Purpose | Phase | Status |
|---|---|---|---|
| `gt-schema` | Units, geometry, frames, snapshots, protocol IDs, clock trait | 1 | Specified (Phase 1 spec) |
| `gt-bus` | FrameBus, EventBus, statistics | 1 | Specified |
| `gt-recorder` | Raw capture file format, writer, tolerant reader | 1 | Specified |
| `gt-replay` | Deterministic replay harness, decoder trait | 1 | Specified |
| `gt-session` | Session lifecycle state machine, lap/pit events | 1 | Specified (sim-agnostic part) |
| `gt-diagnostics` | Source health, freshness, counters | 1 | Specified |
| `gt-adapter-f1` (or `-acc`) | First game adapter | 2 | Pending ADR-10 confirmation |
| Camera spike | Feasibility only, throwaway | 3 | Optional, needs your go-ahead |
| `gt-storage` | Lap/session persistence | 4 | Not specified |
| `gt-track` | Track identity, 2D map | 5 | Not specified |
| `gt-analytics` | Delta, sectors, fuel, corner loss | 6 | Not specified |
| `gt-line` | Best-lap reference line | 7 | Not specified; physics modes deferred |
| UI bridge (Boundary 1) | Desktop UI IPC | 8 | Depends on stack (ADR-13) |
| `gt-engineer` | Template radio + LLM explanation | 9 | Not specified |
| `gt-remote-api` (Boundary 2) | Authenticated LAN/Tailscale API | 10 | Rules fixed (§4); not specified |
| Android companion | Phone dashboard | 10 | Not specified |
| `gt-voice` | PTT + STT/TTS | 11 | Not specified |
| `gt-wheelbase-bridge` | Read-only wheelbase status | 12 | **BLOCKED** |

## 4. Communication architecture (two separate security boundaries)

```
 ┌───────────────────────── Windows PC ─────────────────────────┐
 │                                                               │
 │  Sim (official SM/UDP) ──► Adapter ──► FrameBus / EventBus    │
 │                                          │            │       │
 │              ┌───────────────────────────┘            │       │
 │              ▼                                        ▼       │
 │  ┌─ BOUNDARY 1: Desktop IPC ─┐      ┌─ BOUNDARY 2: Remote API ─┐
 │  │ In-process only           │      │ Separate crate/listener  │
 │  │ Tauri Channels/commands   │      │ TLS 1.3 HTTPS + WSS      │
 │  │ (if the stack is Tauri)   │      │ OFF until a device pairs │
 │  │ NO network socket         │      │ Binds only to interfaces │
 │  │ Full command set          │      │ the user picks: LAN and/ │
 │  └────────────┬──────────────┘      │ or Tailscale (explicit)  │
 │               ▼                     │ Every request & stream   │
 │        Desktop UI (WebView)         │ authenticated; per-device│
 │                                     │ command permissions      │
 │                                     └────────────┬─────────────┘
 └──────────────────────────────────────────────────┼─────────────┘
                                                    │ LAN or Tailscale
                                                    ▼
                                    Android companion / tablet browser
                              (pins desktop cert fingerprint from QR)
```

Clarifications (as requested):
- **Tauri Channels** (or the equivalent in-process IPC if the stack isn't Tauri, **[Unknown]**) are **only** for the Windows Rust backend ↔ the app's own React frontend. They are not a network transport.
- **Android requires a separate, authenticated network transport:** Boundary 2, `gt-remote-api`.
- **LAN and explicitly configured Tailscale access remain supported.** The user picks the interfaces; Tailscale is an opt-in path, and devices reaching the PC over the tailnet still pair and authenticate exactly like LAN devices.
- **No unauthenticated command or telemetry endpoint exists**, including health/version endpoints.
- The two boundaries are separate code, separate listeners (Boundary 1 has none) and separate permission models. Boundary 2 reaches the core only through read-only bus subscriptions and a permission-checked command interface.
- The bundle's `ws://127.0.0.1:48210` loopback socket is **removed**.

## 5. Coordinate system and units (summary; full detail in Phase 1 spec §4)

- World frame: right-handed, Z up. Vehicle frame: ISO 8855. Orientation: unit quaternion body→world.
- Internal units SI via newtypes; named exceptions: engine speed in rev/min, pedal/steering inputs normalised.
- Display-unit conversion happens only in UI code.

## 6. Game adapter comparison (Part 4)

Nothing below is **[Tested]**. "Official" is used only where the publisher or developer documents it.

| Criterion | **EA F1 25** | **ACC** | **LMU** | **AMS2** |
|---|---|---|---|---|
| **Officially documented interface** | UDP spec published by EA for F1 25, plus a separate 2026 Season Pack spec **[Official]** | Kunos "ACC Shared Memory Documentation v1.7.6" (copy hosted on GitHub; not opened from a Kunos URL) **[Official, via mirror]**. Broadcasting SDK shipped with the game since v0.4 (2018 release notes, reported in the press) **[Third-party report of an official release]** | **No official Studio 397 documentation found.** `LMU_Data` shared memory (enabled by *Enable Plugins*) and REST API on port 6397 are known only from community projects **[Third-party]** | `SharedMemory.h` + example app shipped in the game folder `Support\SharedMemory\AMS2_SharedMemoryExampleApp` **[Third-party: reported location on the official Reiza forum; file not opened]**. UDP: forum users say there is no current official source **[Third-party]** |
| **Ease of capturing real telemetry** | Easiest: game sends UDP to any IP; capture with a socket or Wireshark on any OS; works from consoles on the LAN too **[Official: configurable IP/port/rate]** | Windows shared memory → must poll and snapshot pages; Broadcasting UDP separate | Windows shared memory + REST polling; requires *Enable Plugins* and a restart **[Third-party]** | Windows shared memory; updated at the graphics frame rate, so limited to 60 Hz with V-sync on **[Third-party forum report]** |
| **Protocol stability & versioning** | Changes yearly; but every packet header carries packet format / game year / version fields **[Official — re-verify field list against the F1 25 PDF in Phase 2]**. Two live formats in F1 25 **[Official]** | Static page carries `smVersion` **[Official doc]**. Mature, rarely changes **[Unknown]** | New, actively developed interface; no versioning guarantee found **[Unknown]** | Version field at the start of the shared memory **[Third-party]**; PCARS2-derived layout long-lived **[Unknown]** |
| **Testability** | Best: byte-exact datagram fixtures, fuzzing, replay = re-feed datagrams; CI on Linux | Good: page snapshots as fixtures; struct packing must match (`pack(4)` used by community libs **[Third-party]**) | Medium: shared memory snapshots + REST JSON fixtures; interface may change under you | Good: header-defined struct snapshots |
| **Track identification** | Track ID in session packet **[Official — verify]** | Track name in static page **[Official doc]** | Via REST/shared memory **[Third-party]** | Track location/variation strings **[Unknown]** |
| **Track geometry** | World positions of all cars **[Official]**; no edges | `carCoordinates[60][3]` **[Official doc]**; no edges | **Track layout + edges via REST reported [Third-party]**: the only candidate with real boundaries | Participant positions **[Unknown]**; no edges known |
| **Known pitfalls** | Yearly churn; arcade-adjacent audience; no camera | Doc labels fuel as kg (disputed) **[Official doc / Unknown]**; physics updates at graphics step **[Official doc]** | Undocumented; could change with any game update | Graphics-rate updates; UDP undocumented now |

### Recommendation: **EA F1 25** as the single first adapter (ADR-10)

Why:
1. It's the only candidate whose interface **and** versioning are officially documented by the publisher, so Phase 2 can be checked against an authoritative spec rather than community reverse-engineering.
2. It exercises the hardest Phase 1 contracts immediately: recorder, byte-exact replay, versioned protocol detection (two real formats) and fuzzing.
3. Capturing needs no Windows API code at all, so Phase 2 can be built and fully tested on any OS, and the ADR-09 policy is trivially satisfied.
4. All-car world positions give the Phase 5 track map real data.

What this costs: no camera data (the spike must use another sim), no track edges, a yearly format update, and possibly weaker product fit for GravelToys' sim-racing audience.

**Conditions:** you own F1 25 (PC, or a console on the same network). If not, **ACC** is the fallback (official doc, mature). **LMU** should be the *second* adapter because of its potential track-edge data, but only after real captures verify its undocumented interface.

## 7. Wheelbase isolation (Part 5)

- `gt-wheelbase-bridge` is **BLOCKED** (ADR-07).
- It is unblocked only by **evidence** that GravelToys Wheelbase *already* exposes a read-only interface: the code location in that app, plus a capture showing data flowing with no writes from the reader.
- Direct USB HID access to the OpenFFBoard is **rejected** (not read-only in practice).
- No phase touches GravelToys Wheelbase, its firmware, motor configuration, encoder configuration, calibration, FFB settings or controller configuration.
- Phase 1 contains no wheelbase code and no HID dependency (the policy test enforces this; Phase 1 spec §12).

## 8. Deferred (unchanged from review §8, plus)

Racing-line physics modes; native road renderer (unless spike passes); pre-mapped track database; cloud LLMs; wake word; Android remote commands beyond permissions above; VR overlays (out of scope).

## 9. Open items needing your input

1. Repository access (blocks the audit, stack decision ADR-13, and final file placement).
2. Do you own F1 25? (ADR-10)
3. Do you want the camera feasibility spike at all? (ADR-08, phase 3)
4. Does GravelToys Wheelbase already publish any data? (ADR-07)
5. Approve ADR-01 … ADR-13.
