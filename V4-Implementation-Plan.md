# V4 Implementation Plan: POC v4, Continuous Apron Optimization

Implements `V4-Requirements.md` (signed off 2026-06-08). Evolves the V3 codebase (`poc-v3` tag is the revert point). Single self-contained HTML app, vendored glpk.js, offline.

---

## 1. Architecture: the simulation-and-playback split (the central design)

The decisive design choice resolves three requirements at once: live cost accrual (FR-V4), incurred-only cost (FR-V1), and reproducibility (FR-T4, FR-R4).

- **The run is a deterministic logical simulation in sim-time.** Advance the clock in fixed sim-steps; at each optimization cadence, solve the rolling window and commit; apply any disruptions whose time has passed; accrue realized cost for events that have occurred. Given the dataset, settings, and disruption set, the output trace and total are identical every time, independent of machine speed.
- **Playback is the pacing of that simulation over real-time.** Pressing play steps the simulation forward, paced by the speed multiplier (one sim-minute per real-second at base), rendering as it goes. The cost accrues live because the simulation is producing it live. Higher speeds coalesce rendering but run the same logical steps.
- **The solver runs in the glpk.js Web Worker (already async).** The playback loop (requestAnimationFrame) never blocks on a solve. A cadence solve is dispatched to the worker; its commits apply when it returns. If a solve is still running at the next cadence, that cadence is skipped (the clock keeps moving, the optimizer catches up), except in deterministic compute mode where the step waits for the solve so the trace is exact.
- **Two run modes share one engine.** "Play" visualizes the simulation in real-time for the demo. "Run to end" executes the same simulation without real-time pacing, producing the full 48-hour trace in a few seconds for fast comparison. Both yield the identical deterministic trace.

This keeps playback smooth at all speeds, makes the cost honest (only past events), and makes saved runs reproducible.

---

## 2. Subsystems and the V3 reuse map

| Subsystem | New or reuse | Notes |
|---|---|---|
| 48h dataset generator | New | Realistic schedule for 18 stands, rotation links, early-turnaround cascade |
| Sim-time clock and playback | New | Clock, speed slider, play/pause/scrub, the rAF pacing loop |
| Rolling-horizon optimizer loop | New, wraps existing | Cadence + window driver around an extended `buildModel` |
| `buildModel` extensions | Extend | Windowed subset, committed-fix, tiered stability penalties, gate-change vs tow, existing tow and bounds |
| Live picture (who is inbound / on-stand / departed at t) | New | Derived from the clock and committed assignments |
| Disruption system | Extend | Six types, timed, manual + scripted, recorded, replayable |
| Strategy controller | Extend | Live switching, cadence lock, adaptive-to-binding |
| Realized cost ledger | New, reuse drivers | Incurred-only accrual keyed to sim-time, V3 cost drivers and Config units |
| Run manager and comparison | New, reuse chart | Save, restart, rerun, compare; reuse the Value-tab chart renderer |
| POC v4 reframe | New | Branding, "what full-scale adds" content, folder and docs |
| Reused as-is | Reuse | `simulateTwin`, `costParts`, tow model, Pareto weights, `mulberry32`, glpk vendor, CPH brand |

---

## 3. Dependency graph and build order

```
V4.1 Dataset ─┬─> V4.2 Time engine ─┬─> V4.3 Optimizer loop ─┬─> V4.4 Disruptions ─┐
              │                     │                        ├─> V4.5 Strategies ──┤
              │                     │                        └─────────────────────┼─> V4.6 Cost ledger ─> V4.7 Run manager ─> V4.8 Reframe + docs
              └─────────────────────┴────────────────────────────────────────────┘
V4.0 Branding text can land anytime; the folder move is in V4.8.
```

The dataset and time engine are foundational. The optimizer loop depends on both. Disruptions and strategies layer on the loop and can be built in either order. The cost ledger depends on the loop, disruptions, and strategies producing events. The run manager depends on the ledger. The reframe and folder move land last so paths do not churn during development.

---

## 4. Phased work breakdown

Each phase is independently verifiable and leaves the app working. Verify end-to-end (no console errors, prior tabs and the base solve intact) before the next, the same discipline as Phases A to G.

### Phase V4.1 — The 48-hour dataset
- Build a generator producing ~250 arrivals and ~250 departures across 48 hours on the 18 stands, with a diurnal pattern (morning and afternoon peaks, overnight trough) and a realistic aircraft and carrier mix using real CPH routes.
- Give each arrival a rotation link to its same-aircraft departure, with a turnaround time by aircraft type.
- Seed the early-turnaround cascade: a small number of early-morning rotations carry a long turnaround that pushes their linked departure, and the next rotation of that aircraft, off-slot.
- Keep EK151 as the A380 needing R8. Keep the B16-out premise as the opening condition or fold it into the disruption timeline.
- **Verify:** 48-hour span, realistic volume and diurnal shape, rotation links resolve, the cascade is traceable in the data, EK151 present.

### Phase V4.2 — The sim-time clock and playback
- A sim-clock from hour 0 to 48, with play, pause, and scrub.
- The rAF pacing loop advancing sim-time by `realDelta * 60 * speed` sim-seconds, base one sim-minute per real-second.
- The speed slider with the seven discrete multipliers, default 1x, current multiplier shown.
- Derive the live picture from the clock: each flight is scheduled, inbound, on-stand, or departed at time t.
- **Verify:** plays at each speed within tolerance, pause and scrub work, the picture matches the clock, deterministic across two identical advances.

### Phase V4.3 — Continuous optimizer loop (the core)
- The cadence driver: at each optimization cadence (Config, default every 5 sim-min), assemble the rolling window (flights with on-block inside `[now, now + window]`, Config window default 3 h, that are not yet committed) and dispatch a solve.
- Extend `buildModel` with options: a flight subset (the window), committed flights fixed to their stands and treated as occupying them, tiered stability penalties (a not-yet-arrived flight is cheap to move, a parked aircraft costs a tow, a boarding or boarded aircraft costs a passenger-facing gate-change penalty, the highest), and the existing tow and capacity terms.
- Commit logic: a flight is committed when the clock reaches `on-block minus commit lead` (Config, default 20 min); committed flights enter the live picture and stop being re-optimized except for cause (DP-3).
- The on-ground reassignment policy (DP-1 to DP-6): short timing shifts within buffer do not move a committed flight; a stand going unserviceable or a hard conflict forces a re-decide; each grounded-aircraft move is logged with its reason.
- The toggle: on runs the loop; off runs the occasional or static baseline (optimize once at the start, then hold) for comparison.
- High-speed throttle: if a worker solve is still running at the next cadence, skip it in live mode; wait for it in deterministic compute mode.
- **Verify:** incoming flights get optimal stands before commit; a short delay causes no move; a stand fault forces a re-decide; towing on frees a contact stand; each window solve stays inside the per-solve budget; off-mode does not adapt.

### Phase V4.4 — Disruption injection over 48 hours
- A disruption record `{ type, time, params, duration }` for the six types: turnaround delay, handler capacity reduction, stand offline, early arrival, late arrival, departure delay, longer turnaround.
- Manual injection at the current clock time from a panel; scripted injections pre-loaded onto the timeline; both recorded into the active run's disruption set.
- Application: as the clock passes a disruption's time, it mutates the live picture and the optimizer reacts; duration or a clear action ends it; everything logged.
- Replay: applying a saved disruption set reproduces the same run.
- **Verify:** each type injects and the optimizer responds; clear and restore work; the event log records each; a replay of the same set reproduces the total.

### Phase V4.5 — Continuous trade-off strategy control
- Live strategy switching (protect passengers, handlers, schedule, balanced, adaptive) feeding the next solve's weights.
- The cadence lock: a minimum-dwell selector {no limit, 30 min, 1 h, 2 h}; after a change the control is locked until the dwell elapses, with the remaining time shown; earlier changes refused.
- The adaptive strategy re-weights to the binding resource no more often than the cadence.
- **Verify:** switching changes the next solve's weights; the lock enforces the dwell; no limit allows free changes; adaptive respects the cadence.

### Phase V4.6 — Realized cost ledger
- Accrue cost for events as they occur in sim-time: connections missed, slip and overload delay minutes, bus servicing, tows, escalations and reaccommodation, using V3 drivers and Config units.
- Incurred-only: the ledger sums events at or before the clock; advancing adds, scrubbing back does not add future cost.
- A per-sim-minute cumulative trace and a running total; live accrual during play; the total at the end of a full run.
- Add the realized-cost view as the primary Value content. **The V3 forward-looking four-case projection is kept as a separate view** (a sub-toggle or secondary panel within the Value tab), not retired, so both the projection and the realized trace are available.
- **Verify:** the ledger only counts past events; it rises with the clock and halts on pause; the minute-by-minute trace and total are present after a full run.

### Phase V4.7 — Run management and comparison
- A run record `{ settings, disruptionSet, costTrace, total, label }`.
- Save the current run to in-session memory and browser local storage; restart resets the clock and committed state; rerun plays again with changed settings against the same dataset.
- Compare two or more saved runs on one chart (cumulative cost over time) with totals and the delta, reusing the Value-tab chart renderer.
- "Run to end" computes a full trace without real-time pacing for fast comparison.
- **Verify:** save and reload a run; compare continuous-plus-adaptive against occasional-plus-fixed on one chart with the delta; a rerun of a saved configuration reproduces its total.

### Phase V4.8 — POC v4 reframe, folder move, docs
- Rebrand the header and chrome to POC v4; remove or reframe the "production vision" wording.
- Add the "what a full-scale build would add" content (a Governance or Definitions card and a docs line): real A-CDM data, a trained ML timing model with calibrated uncertainty, live Airhart integration, governed write-back, crew as a resource, calibrated cost units.
- Move the folder from `full-scale/` to `poc-v4/` with `git mv`; update `.claude/launch.json`, the README repo-layout, and the docs.
- Update `Solution Overview and Assumptions.md` and `v2_build_notes.md` for V4.
- **Verify:** branding consistent, the production-gap list present, the server launches at the new path, links resolve, a clean full run end to end with no console errors.

---

## 5. Detailed design notes for the hard parts

### 5.1 Bounding the MIP so continuous solving stays fast (the top risk)
At any cadence only the flights between `now + commit-lead` and `now + window` that are not yet committed are decision variables, roughly 10 to 30 flights, not the 48-hour set. Committed flights are fixed and only occupy their stands as constraints. The solve is therefore the same size class as the V3 whole-apron solve, which runs in tens of milliseconds. Add a glpk time cap and presolve. At 10x speed a cadence fires roughly every half real-second; the worker-async dispatch with skip-if-busy keeps playback smooth, and deterministic compute mode waits per step so the saved trace is exact.

### 5.2 Tiered stability so on-ground aircraft are rarely moved (DP-4, DP-6)
Extend the stability term from V3's single penalty to a tier by the flight's state at solve time: not-yet-arrived, low penalty (free to re-route); parked with no passengers, medium penalty equal to a tow cost (a tow-to-remote is the lever); boarding or boarded, high penalty (a passenger-facing gate change, avoided unless the stand is lost). A forced move (stand unserviceable) overrides by making the current stand infeasible. Every grounded-aircraft move carries a logged reason.

### 5.3 Determinism
One seeded RNG (`mulberry32`) drives dataset jitter and any stochastic disruption parameter. The simulation advances in fixed sim-steps and solves at sim-time cadences, so the logical run is identical regardless of playback speed or machine. Saved runs store their seed and disruption set.

### 5.4 The early-turnaround cascade (FR-D2)
Model the aircraft rotation explicitly: arrival, turnaround, the linked departure, then the aircraft's next arrival. A seeded long turnaround on an early rotation delays its departure, which delays the aircraft's next arrival, which arrives off-slot later in the day. This is data-driven, visible in the event log, and is the textbook "delay in early turnarounds" propagation.

### 5.5 Reusing the cost model unchanged
The realized ledger uses the same drivers and Config units as the V3 Value tab, so V3 and V4 stay coherent. The only change is integration over a played sim-time rather than a single forward projection.

---

## 6. Risks and mitigations

- **Browser performance under continuous solving (high).** Mitigated by the bounded window, the worker-async solve, the skip-if-busy throttle, and a glpk time cap. Fallback: widen the cadence or narrow the window in Config.
- **State complexity across play, pause, scrub, inject, and rerun (medium).** Mitigated by a single deterministic simulation as the source of truth and a clean reset path; playback only reads it.
- **UI density (medium).** The app already has ten tabs; V4 adds playback controls and run management. Mitigated by putting playback in a persistent control bar and run management in the Value tab.
- **Determinism versus real-time skipping (medium).** Mitigated by the two-mode design: live mode may skip for smoothness; compute mode is exact for saved and compared runs.
- **Scope creep (medium).** Mitigated by the phase gates; each phase ships a working app.

---

## 7. Verification and acceptance

- Per-phase verification as listed, run in the preview with DOM and console checks, the same method used through Phases A to G.
- End-to-end acceptance against the Definition of Done in the requirements: a full 48-hour run plays at the specified speeds with the cascade present; continuous optimization and towing toggle live and respect the on-ground policy; all six disruptions inject, log, and replay; strategies switch within the selected cadence; the Value tab accrues realized cost and shows the minute-by-minute trace and total; two runs save and compare with their delta; the base solve and prior tabs are unaffected; no console errors.

---

## 8. Sequencing summary

Build in dependency order: V4.1 dataset, V4.2 time engine, V4.3 optimizer loop, then V4.4 disruptions and V4.5 strategies, then V4.6 cost ledger, V4.7 run manager, V4.8 reframe and folder move. Branding text can land early. Tag a `poc-v4` checkpoint at the end. `poc-v3` remains the revert point throughout.
