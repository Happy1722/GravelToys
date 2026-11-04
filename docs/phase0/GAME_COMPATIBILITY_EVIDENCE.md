# GravelToys Race — Game Compatibility Evidence

**Date:** 2026-11-04
**Purpose:** replace the bundle's "Verified" cells with what can actually be shown. **Nothing here has been tested on a real game yet.** Each row needs a captured recording before it moves to **[Tested]**.

Labels: **[Official]** official documentation · **[Third-party]** community software/docs, not authoritative · **[Unknown]** needs validation · **[Tested]** none yet.

## 1. Evidence summary

| Sim | Interface | What has evidence | Bundle claims that are wrong or unproven | Live camera for road projection |
|---|---|---|---|---|
| **Le Mans Ultimate** | Built-in `LMU_Data` shared memory (needs *Settings → Gameplay → Enable Plugins*); local REST API on port 6397 | Native shared memory, no plugin DLL needed **[Third-party: LMU Pitwall, NeoPedal]**; ~100 Hz telemetry / 5 Hz scoring **[Third-party]**; REST API exposes track layout, **track edges**, pit lane **[Third-party]** | Bundle says it uses an rF2-derived plugin; that's now the legacy path | Camera pose/FOV in `LMU_Data`: **[Unknown]** |
| **rFactor 2** | rF2SharedMemoryMapPlugin64.dll (TheIronWolf, community) on top of ISI's official plugin interface | Graphics struct includes camera position, orientation, camera type **[Third-party: rF2SharedMemoryNet docs]** | "mFOV exposed" — **not found**; "fully supported by Studio 397" — unproven | Pose: plausible. FOV: **[Unknown]** → Tier 2 at best until proven |
| **EA F1 25** | Official UDP (port 20777 by default) | Spec published by EA **[Official: EA forums, F1 25 UDP spec]**; two formats selectable in-game: 2025 and 2026 Season Pack **[Official: EA 2026 Season Pack UDP spec post]**; motion and lap data at the menu-selected rate **[Official: EA staff reply, F1 24 thread]** | Treats "F1 23/24/25" as one adapter | No camera data. HUD/map only. Tier "1.5" in bundle is not justified |
| **Assetto Corsa Competizione** | Shared memory (`acpmf_physics/graphics/static`) + Broadcasting UDP | Physics updates "at each graphic step"; `carCoordinates[60][3]` + `carID[60]` in graphics page; speed in km/h; `steerAngle` normalised −1..1 **[Official: ACC Shared Memory doc v1.7.6, copy hosted on GitHub]** | "333 Hz" — doc says it follows the graphics step; fuel documented as **kg** in the doc (widely believed litres) **[Unknown]** | None. HUD/map only |
| **Assetto Corsa (original)** | Shared memory + optional in-game Python apps; optional CSP (third-party) | Shared memory structure documented on the official AC forum **[Third-party reference to official forum]** | `ac.getCameraMatrix` in the official Python API — **not found**; camera access is normally via CSP Lua (in-game, third-party) **[Unknown]** | Only via in-game third-party extension. Optional, opt-in |
| **iRacing** | Official iRacing SDK (shared memory + session-info YAML) | 60 Hz live; the "360 Hz" option is 60 Hz delivery with 6 samples per batch **[Third-party: SimXperience]** | Rate claims; live world XYZ for cars not exposed (bundle says so too) **[Unknown — verify against SDK var list]** | None. HUD/map only. Never touch the process (EasyAntiCheat) |
| **Automobilista 2** | Project CARS 2 shared memory (`$pcars2$`) or UDP | Odd/even sequence-number torn-read scheme (bundle describes it) **[Unknown — verify vs. the official header]** | Rates and orientation fields unproven | None found. HUD/map only |
| **Assetto Corsa EVO** *(missing from bundle)* | Shared memory `Local\acevo_pmf_physics/graphics/static` | Documented by Kunos in a Google Doc with a 2026 change log, e.g. `car_ids` added on 2026-04-28 **[Third-party pointer to Kunos doc]** | — | **[Unknown]** |

## 2. What "Tested" will require (per sim)

A sim moves to **[Tested]** only when all of these are in the repo:

1. A raw capture file (≥ 3 laps, including a pit stop and a session restart), recorded with the Phase 1 recorder.
2. A unit table: each native field → unit → conversion → evidence (capture timestamp + what was done in-game).
3. Coordinate fixtures: a straight, a left turn and an elevation change, with the expected signs.
4. A freshness test: pause, Esc-to-garage, alt-tab, game exit.
5. For any camera claim: a projected-point pixel-error measurement (see review §5.1).

## 3. Open questions for you

1. Which of these sims do you **own and actually drive**? The first adapter should be one you can capture from on your own rig.
2. Do you use **VR** or triple screens? (VR rules out desktop road overlays; triples complicate projection.)
3. Do you race **online** in leagues that restrict overlays or aids?

## Sources

- [EA — F1 25 UDP Specification](https://forums.ea.com/blog/f1-games-game-info-hub-en/f1%C2%AE-25-udp-specification/12187347)
- [EA — F1 25: 2026 Season Pack UDP Specification](https://forums.ea.com/blog/f1-games-game-info-hub-en/ea-sports%E2%84%A2-f1%C2%AE25-2026-season-pack-udp-specification/12187347)
- [EA — F1 24 UDP spec thread (send-rate reply)](https://forums.ea.com/discussions/f1-24-general-discussion-en/f1-24-udp-specification/8369125/replies/8369210)
- [ACC Shared Memory Documentation v1.7.6 (PDF copy)](https://github.com/fenegroni/simdisplay/files/6255394/ACCSharedMemoryDocumentationV1.7.6.pdf)
- [LMU Pitwall (LMU_Data shared memory, REST :6397)](https://github.com/Swizzjack/lmu-pitwall)
- [NeoPedal game setup (LMU REST: track edges)](https://github.com/Keenny38/ModernTinyPedals/wiki/Game-Setup)
- [rF2SharedMemoryNet — GraphicsInfo struct](https://domaslau.github.io/rF2SharedMemoryNet/api/rF2SharedMemoryNet.RF2Data.Structs.GraphicsInfo.html)
- [SimXperience — iRacing 360Hz explanation](https://www.simxperience.com/forum/simvibe-community-q-a-5/360hz-telemtry-not-working-37)
- [AC EVO shared memory doc (Kunos Google Doc)](https://docs.google.com/document/d/1WzqMLkW2o_C0LGcvdMRelAV31ZIifux0CSHD9k6ddz0/)
