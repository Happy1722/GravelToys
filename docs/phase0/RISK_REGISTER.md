# GravelToys Race — Risk Register (rev. 2)

**Date:** 2026-11-04 · Rev. 2 adds R-21 to R-28 and updates mitigations to match `ARCHITECTURE_DECISION_LOG.md`.
Severity / likelihood: H / M / L. Labels: **[Official] [Third-party] [Unknown] [Proposed]**. Nothing **[Tested]** yet.

| ID | Risk | Sev | Likely | What it means for you | Mitigation [Proposed] | ADR | Phase | Status |
|---|---|---|---|---|---|---|---|---|
| R-00 | Repository not yet audited | H | Certain | All paths are proposals; the stack is unknown | Provide repo; task P1-00 | 13 | 1 | **Open — blocking** |
| R-01 | No live camera pose/FOV in most sims | H | Certain for F1/ACC/iRacing/AMS2 | Road projection may work in 0–2 sims | Optional standalone spike with pixel-error pass/fail; HUD ribbon + map as default | 05, 08 | 3 | Open |
| R-02 | Anti-cheat action from touching game processes | Critical | L if rules followed | User bans | Interface-presence detection; forbidden-API + dependency policy tests | 09 | 1+ | Mitigation specified |
| R-03 | Exclusive fullscreen hides overlays | M | H | Overlay invisible | Detect + guide user | — | 8 | Open |
| R-04 | Local LLM/STT steals CPU/GPU from the sim | H | M | Stutters | Template radio first; priority limits; measure frame times | 08 | 9, 11 | Open |
| R-05 | Torn shared-memory reads | M | H | Value spikes | Sequence double-read + stress test; `torn_reads_retried` counter | — | 2+ | Counter specified |
| R-06 | Wheelbase/FFB disruption | Critical | L | Safety and trust | Bridge **blocked**; no HID; policy test | 07 | 12 | Blocked |
| R-07 | Coordinate convention error | H | Certain if unfixed | Mirrored map/line | Z-up world + ISO 8855 body; fixtures | 01 | 1–2 | Mitigation specified |
| R-08 | Wrong units (e.g. ACC fuel "kg" in the official doc) | H | M | Wrong fuel-to-finish | `FuelAmount` enum forces explicit unit; unit table from captures | 02 | 2 | Mitigation specified |
| R-09 | Yearly protocol changes (F1 25 has 2 formats) **[Official]** | M | Certain | Adapter breaks after updates | `ProtocolId` + reject unknown versions + fixtures per version | 04 | 2+ | Mitigation specified |
| R-10 | No source for track edges | H | Certain except maybe LMU **[Third-party]** | Fake boundaries would mislead | Only driven corridor until a real source is verified | — | 5, 7 | Open |
| R-11 | Wheelbase bridge needs a forbidden Wheelbase-app change | H | Unknown | Feature may be impossible | Unblock only with evidence of an existing read-only interface | 07 | 12 | **Needs your answer** |
| R-12 | Unauthenticated localhost/remote endpoints | H | M if built as in bundle | Web pages or LAN devices could read data or send commands | Boundary 1 has no socket; Boundary 2 authenticates everything incl. health; Phase 1 has no listener | 06, 11 | 8, 10 | Mitigation specified |
| R-13 | Long-lived tokens; QR visible on stream | M | M | Device takeover | Short-lived tokens + refresh; desktop confirm; revoke-all | 06 | 10 | Open |
| R-14 | Overlay latency | M | H | Line lags the road | Measure in spike; hide above error threshold | 05 | 3, 7 | Open |
| R-15 | VR users see no overlay | M | Certain for VR | Feature absent in VR | Out of scope; phone dashboard alternative | — | — | **Needs your decision** |
| R-16 | Driving-aid rules online | M | Unknown | League penalties | "Aids off online" setting | — | 7 | Open |
| R-17 | Scope overload | H | H | Nothing finished | One adapter end to end; ADR-11 scope boundary | 08, 11 | All | Mitigated by plan |
| R-18 | Stack mismatch (Tauri assumed; Studio is Electron) | M | Unknown | Wrong UI plan | Phase 1 stack-independent; decide at audit | 13 | 1 | Open |
| R-19 | Third-party plugin licensing (rF2 plugin, CSP) | M | Unknown | Can't bundle | Link, don't bundle; check licences | — | 13 | Open |
| R-20 | PDF performance targets lost their units | L | Certain | Ambiguous acceptance | Baselines first; targets with units afterwards | — | 1 | Mitigation specified |
| **R-21** | Process-inspection libraries may open game processes internally **[Unknown]** | H | M | Anti-cheat exposure through a dependency | Don't use them; dependency allowlist test | 09 | 1+ | Mitigation specified |
| **R-22** | LMU's interface is undocumented officially; may change with any game update **[Third-party only]** | M | M | LMU adapter breaks without notice | LMU second, not first; fixtures from real captures; unknown-version rejection | 04, 10 | 13 | Open |
| **R-23** | Tailscale/LAN misconfiguration exposes the remote API more widely than intended | H | M | Data or commands reachable by unexpected devices | Explicit interface selection; never `0.0.0.0` by default; tailnet still requires pairing; listener test | 06 | 10 | Mitigation specified |
| **R-24** | First adapter (F1 25) not owned, or not the sim GravelToys users care about | M | Unknown | Wasted Phase 2 or weaker product fit | Confirm ownership; ACC fallback; LMU as second adapter | 10 | 2 | **Needs your answer** |
| **R-25** | Recordings contain other drivers' names (F1 participant data, online lobbies) | M | H for online sessions | Privacy issue if shared | `contains_other_drivers_names` flag; recordings stay local; warn before sharing | — | 1–2 | Mitigation specified |
| **R-26** | Recording disk growth | L | M | Disk fills on long sessions | Rotation at 1 GiB; disk-full stops recording only; compression later | — | 1 | Mitigation specified |
| **R-27** | Synthetic fixtures mistaken for real game support | M | L | False confidence | `SimId::Synthetic`, test-only feature flag, refused by release builds | — | 1 | Mitigation specified |
| **R-28** | Determinism broken by hidden wall-clock reads | M | M | Replay tests flaky or meaningless | `Clock` trait everywhere; golden digest on two OSes | 12 | 1 | Mitigation specified |
