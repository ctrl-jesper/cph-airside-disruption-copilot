# V4 Requirements: Continuous Apron Optimization

**Status:** Signed off 2026-06-08. **Author role:** Business analyst and solution engineer. **Predecessor:** V3 (`poc-v3` tag), the app with Phases A to G plus the four-policy Value tab.

**Framing.** V4 is named **POC v4**. "Full scale" was always a misnomer: there is no real A-CDM data, no trained ML timing model, and no live integration. V4 states this plainly and lists what a real full-scale implementation would add (see FR-Z), so the boundary between what is demonstrated and what is production work is explicit.

V4 turns the snapshot optimizer into a continuous one. A 48-hour feed plays forward; the operator can switch continuous apron optimization and trade-off strategies on and off and inject disruptions while it runs; and the Value tab records the cost that was actually incurred, minute by minute, so two full runs under different settings can be compared.

---

## 1. Vision and the one-line goal

Show that continuously re-optimizing the apron and adapting the trade-off as conditions change produces a measurably lower realized cost across a full operating cycle than a static or occasionally-optimized plan, while keeping disruptive stand changes to a minimum, the way a real operation would.

---

## 2. Scope

- **In scope.** A 48-hour synthetic feed, a time-playback engine, continuous apron optimization with a research-grounded on-ground-reassignment policy, six injectable disruption types, cadence-limited trade-off strategy control, realized cost accounting, and run save-and-compare.
- **Out of scope.** Live integration, a trained ML model, crew rostering, landside and terminal operations beyond passport and connections, and any write-back to airport systems. The system stays read-only, advisory, and human-in-the-loop.

---

## 3. Glossary

- **Sim-minute / real-second.** One simulated minute of the feed maps to a configurable number of real seconds. Base is one sim-minute per one real-second.
- **Commit.** A flight's stand is fixed for execution once the flight is within a commit lead time of its on-block (arrivals) or off-block (departures).
- **On-ground aircraft.** An aircraft physically on a stand between on-block and off-block.
- **Gate change.** Moving an on-ground, passenger-facing aircraft to a different contact stand. Expensive and passenger-disruptive.
- **Tow to remote (intermediate parking).** Towing a long-dwell aircraft, with no passengers aboard, to a remote parking position to free its contact stand. Costs a tug and crew, not passenger disruption.
- **Rolling horizon.** Re-solving only the flights inside a forward time window at each cadence, rather than the whole 48 hours at once.
- **Run.** One full 48-hour playback under a fixed set of settings, producing a realized cost trace and a total.

---

## 4. Domain policy: reassigning aircraft already on the ground (research-grounded)

This is the operational crux and the requirement that needs the most care. The literature on the Stand Allocation Problem is consistent, and V4 must follow it.

- **DP-1.** Continuous optimization assigns or re-assigns **incoming flights that are not yet committed**. This is the default and the bulk of the work.
- **DP-2.** Short delays and early arrivals are absorbed by **buffer time**, not by reassignment. A flight whose timing shifts within its buffer keeps its stand.
- **DP-3.** An **on-ground aircraft is reassigned only for cause**: its stand becomes unserviceable, an unavoidable hard conflict, or a planned tow-to-remote for a long dwell whose value exceeds the tow cost.
- **DP-4.** A **passenger-facing gate change** (an on-ground aircraft that is boarding or boarded moved to another contact stand) is the most expensive move and must carry the highest stability penalty. The optimizer avoids it unless forced.
- **DP-5.** A **tow to remote during dwell** is permitted when towing is enabled and the freed contact-stand value exceeds the tug-and-crew cost, consistent with the V3 tow model.
- **DP-6.** Every reassignment of an on-ground aircraft must beat a **stability penalty** (material-gain threshold). The optimizer reports each such move with its reason, so the operator can see why a grounded aircraft was moved.

References: Stand Allocation Problem minimizes towing movements and gate changes ([ScienceDirect, exact and heuristic SAP](https://www.sciencedirect.com/science/article/abs/pii/S0377221715003331)); short delays are not tackled by reassignment due to passenger-movement penalties, buffers absorb small disruptions, and rolling-horizon re-optimization maintains coherence ([Robust gate assignment, ScienceDirect](https://www.sciencedirect.com/science/article/abs/pii/S0305048314000760); [aircraft-gate reassignment after disruptions, ScienceDirect](https://www.sciencedirect.com/science/article/pii/S1000936124002875)); recoverable robust stand allocation ([OR Spectrum, GRU case](https://link.springer.com/article/10.1007/s00291-018-0525-3)).

---

## 5. Functional requirements

Each requirement has an ID and acceptance criteria (AC). "The system" is the V4 app.

### 5.1 Dataset (FR-D)

- **FR-D1. 48-hour horizon.** The dataset covers a continuous 48-hour window of arrivals and departures at the modelled apron.
  - AC: the feed clock runs from hour 0 to hour 48; flights exist across the full span, including overnight low-traffic hours.
- **FR-D2. Early-turnaround delay cascade ("Forsinkelser i tidlige turnarounds").** The dataset seeds delays in early-morning turnarounds that propagate into later rotations, so the cascade is visible without injection.
  - AC: at least one identifiable rotation chain where an early long turnaround pushes a later flight off-slot, traceable in the event log.
- **FR-D3. Realistic structure.** Flights carry the existing fields (carrier, route, aircraft type and size code, zone, handler, scheduled and predicted on/off-block, dwell, pax, connecting pax, planned stand) plus a rotation link (the next flight the aircraft operates).
  - AC: each arrival that has a same-aircraft departure references it; the rotation link drives the early-turnaround cascade.
- **FR-D4. Synthetic and labelled.** The dataset is synthetic, uses real CPH carriers and routes for credibility, and is labelled as not real operational data. EK151 remains the real A380 requiring the single code-F stand.
  - AC: a visible "synthetic data" label; EK151 present with code F.
- **FR-D5. Realistic scale on 18 stands.** The apron keeps ~18 stands. Movement volume over 48 hours is consistent with real operations on that many stands: roughly 7 arrivals per stand per day, so on the order of ~250 arrivals and ~250 departures (~500 movements) across the 48 hours, with overnight troughs and daytime peaks. The rolling-horizon solver only ever works a forward window, so this volume stays interactive (NFR-2).
  - AC: total movements over 48 hours fall within a realistic band for 18 stands; the traffic shows a diurnal pattern; the rolling-horizon window keeps each solve inside the budget.

### 5.2 Time and playback engine (FR-T)

- **FR-T1. Play, pause, and scrub.** The operator can play, pause, and scrub the 48-hour feed.
  - AC: play advances the clock; pause halts it; scrubbing moves the clock and re-renders the picture at that time.
- **FR-T2. Base speed one sim-minute per real-second.** At base speed, one simulated minute elapses per real second, so 48 hours plays in 48 real minutes.
  - AC: with speed at base, the clock advances 60 sim-minutes per 60 real-seconds within a small tolerance.
- **FR-T3. Speed slider, ten times faster to ten times slower.** A slider scales playback from ten times slower to ten times faster than base.
  - AC: at maximum the feed runs ten times faster (48 hours in under five real minutes); at minimum, ten times slower; the current multiplier is displayed. Slider range and steps are a sign-off decision (D-1).
- **FR-T4. Deterministic clock.** Given the same dataset, settings, and injected disruptions, the run is reproducible.
  - AC: two runs with identical settings and injections produce the same realized cost total.

### 5.3 Continuous apron optimization (FR-O)

- **FR-O1. Toggle.** Continuous apron optimization can be switched on and off while the feed plays.
  - AC: a visible toggle; turning it off freezes assignments to whatever is committed; turning it on resumes re-optimization at the next cadence.
- **FR-O2. Rolling-horizon re-solve.** When on, the system re-solves the flights inside a forward window on a cadence as the clock advances, seeded from the committed picture with the stability term.
  - AC: at each cadence the not-yet-committed flights in the window are (re)assigned; committed flights are held per the DP policy; the solve completes within the per-solve budget (NFR-2). The cadence and the forward window length are **configurable in the Config tab**; defaults are re-solve every 5 sim-minutes over a rolling 3-hour window.
- **FR-O3. Assign incoming flights optimally.** New incoming flights are assigned to the optimal stand under the active trade-off strategy.
  - AC: an arriving flight with no committed stand receives one before its commit lead time; the chosen stand reflects the active strategy.
- **FR-O4. On-ground reassignment policy.** The system applies DP-1 to DP-6. Grounded aircraft are not moved for short timing shifts; they are moved only for cause, and passenger-facing gate changes carry the highest penalty.
  - AC: a short delay within buffer causes no stand change; a stand going unserviceable triggers a reassignment of the affected grounded aircraft; each grounded-aircraft move is logged with a reason; passenger-facing gate changes occur only when no alternative exists.
- **FR-O5. Towing.** When towing is enabled, continuous optimization may tow a long-dwell aircraft to remote to free a contact stand, per the V3 tow model and DP-5, and minimizes total tow movements.
  - AC: with towing on, a long-dwell aircraft tows to remote when it frees enough contact value; with towing off, no tows occur; tow count is reported.
- **FR-O6. Comparison baseline behaviour.** When continuous optimization is off, the system runs an "occasional" or static policy (a one-off optimize at the start, then hold) so the two modes can be compared in the Value tab.
  - AC: with the toggle off, assignments do not re-optimize as new disruptions emerge; cost accrues accordingly.

### 5.4 Disruption injection over 48 hours (FR-X)

- **FR-X1. Injectable types.** The operator can inject, at the current feed time, each of: a turnaround delay; a ground-handler capacity reduction; a stand going offline for a technical reason; a flight arriving earlier than its slot; a flight arriving later than its slot; a departure delay; and a longer-than-planned ground-handler turnaround.
  - AC: each type is selectable and takes effect from the injection time forward; the live picture and the optimizer react.
- **FR-X2. Timed and clearable.** An injected disruption has a start time and, where applicable, a duration or clear action, so it can end.
  - AC: a stand offline can be restored; a handler capacity reduction can be cleared; the optimizer re-validates and recovers.
- **FR-X3. Logged.** Every injection is recorded in the event log with its time and parameters.
  - AC: the event log shows each injection; the realized cost reflects its consequences.
- **FR-X4. Mechanism.** Whether disruptions are injected manually during playback, pre-scripted onto the timeline, or both, is a sign-off decision (D-4). For reproducible run comparison, the same disruption set must be replayable across runs.
  - AC: a saved run records its disruption set; a rerun can replay the identical set.

### 5.5 Continuous trade-off strategy control (FR-S)

- **FR-S1. Switchable strategies.** The operator can switch the active trade-off strategy (protect passengers, protect handlers, protect schedule, balanced, or adaptive-to-binding-resource) while the feed plays.
  - AC: changing the strategy changes the weights used by the next re-solve; the active strategy is displayed.
- **FR-S2. Minimum change cadence.** Strategy changes are rate-limited. The operator selects the minimum dwell between changes from a fixed set: no limit, every 30 minutes, every 1 hour, or every 2 hours of sim-time.
  - AC: after a change, the strategy control is locked until the selected dwell elapses; the lock and time remaining are visible; attempts to change earlier are refused; "no limit" allows changes at any time.
- **FR-S3. Adaptive option.** An adaptive strategy that automatically shifts weight to the binding resource (per the V3 Value-tab rule) is available and also respects the minimum change cadence.
  - AC: with adaptive selected, the strategy re-weights no more often than the selected cadence.

### 5.6 Realized cost accounting (FR-V)

- **FR-V1. Incurred only, never forecast.** The Value tab accumulates cost that has actually been incurred up to the current feed time. It does not project future cost.
  - AC: at feed time t, the displayed cumulative cost includes only events at or before t; advancing the clock adds to it; scrubbing back does not add future cost.
- **FR-V2. Minute-by-minute trace.** After a full 48-hour run, the Value tab shows cumulative incurred cost minute by minute and the run total.
  - AC: a cumulative cost curve spanning the 48 hours with one point per sim-minute (or a defined sampling step), plus a single total figure in euros.
- **FR-V3. Cost drivers.** Cost uses the established drivers and Config unit costs: connections at risk, delay minutes from handler overload and turnaround slips, remote bus servicing, tow movements, escalations and reaccommodation.
  - AC: the cost decomposition is available; unit costs are labelled assumptions and tunable in Config.
- **FR-V4. Live accrual during playback.** While the feed plays, the cumulative cost rises in step with the clock.
  - AC: the Value figure updates as the feed advances; pausing halts accrual.

### 5.7 Run management and comparison (FR-R)

- **FR-R1. Save a run.** A completed run, its settings, its disruption set, and its realized cost trace and total can be saved.
  - AC: a save action stores the run; the saved run lists its settings (continuous optimization on/off, towing on/off, strategy and cadence, disruption set) and its total.
- **FR-R2. Restart and rerun.** The feed can be reset to hour 0 and rerun with different settings against the same dataset.
  - AC: reset returns the clock to 0 and clears committed state; a new run can be started with changed settings.
- **FR-R3. Compare runs.** Two or more saved runs are charted together (cumulative cost over time) with their totals, so the value of continuous optimization and adaptive trade-offs is visible.
  - AC: at least two runs (for example continuous optimization plus adaptive strategy versus occasional optimization with a fixed strategy) appear on one chart with their totals and the delta between them.
- **FR-R4. Reproducibility.** Reusing a saved run's dataset, settings, and disruption set reproduces its total.
  - AC: rerunning a saved configuration yields the same total within tolerance.

### 5.8 POC framing (FR-Z)

- **FR-Z1. Branded as POC v4.** The app's title and chrome state that it is a proof of concept (POC v4), not a full-scale production system.
  - AC: the header reads POC v4; the "production vision" wording is removed or reframed.
- **FR-Z2. What full-scale would add.** A visible section (a Definitions or Governance card, and a line in the docs) lists what a real full-scale implementation requires beyond this POC: real A-CDM data, a trained ML timing model with calibrated uncertainty, live Airhart integration, write-back under high-risk-grade governance, crew as a modelled resource, and cost units calibrated to real business value.
  - AC: the list is present in the app and the docs; each item is stated as a production gap, not a current capability.

---

## 6. Non-functional requirements (NFR)

- **NFR-1. Read-only and human-in-the-loop.** No write-back, no autonomous cancel or divert. The continuous optimizer proposes; the operator's settings govern; everything is logged.
- **NFR-2. Browser performance.** The app stays interactive while playing. Each rolling-horizon solve completes inside a budget (target under ~150 ms) so playback does not stall. The dataset scale and window length are chosen to meet this; the solver works on a forward window, not the whole 48 hours.
- **NFR-3. Offline and self-contained.** Single HTML file, vendored solver, no network dependency, consistent with V3.
- **NFR-4. Determinism.** Seeded randomness so runs are reproducible for comparison.
- **NFR-5. CPH brand and accessibility.** Same visual system as V3; text-legible status, no functional reliance on colour alone.
- **NFR-6. Honesty.** Synthetic data and assumption labels throughout; the cost model's unit costs are stated as assumptions.

---

## 7. Data model requirements

- **DM-1.** A 48-hour flight schedule with rotation links, sized to NFR-2.
- **DM-2.** A stand inventory (extending the V3 stands) sufficient to make 48-hour contention realistic.
- **DM-3.** A disruption set structure that can be saved with a run and replayed (FR-X4, FR-R4).
- **DM-4.** A run record structure: settings, disruption set, realized cost trace, total.

---

## 8. Resolved decisions (signed off 2026-06-08)

- **D-1. Speed slider.** Resolved: discrete multipliers 0.1x, 0.25x, 0.5x, 1x, 2x, 5x, 10x, default 1x.
- **D-2. Apron and dataset scale.** Resolved: keep ~18 stands; movement volume consistent with real operations on 18 stands (~250 arrivals and ~250 departures over 48 hours), per FR-D5. The rolling-horizon window keeps it interactive.
- **D-3. Optimization cadence and window.** Resolved: re-solve every 5 sim-minutes over a rolling 3-hour forward window, both configurable in the Config tab (FR-O2).
- **D-4. Disruption mechanism.** Resolved: both manual injection during playback and a scriptable timeline; manual injections are recorded into the run's disruption set so the run is replayable.
- **D-5. Strategy cadence options.** Resolved: selectable minimum-dwell set is {no limit, 30 min, 1 h, 2 h} (FR-S2).
- **D-6. Commit lead time and buffer.** Resolved: commit at 20 sim-minutes before on-block; buffer of 10 to 15 minutes (DP-2).
- **D-7. Run persistence.** Resolved: in-session plus browser local storage; file export optional.
- **D-8. Cost model continuity.** Resolved: reuse the V3 cost drivers and unit costs unchanged, tunable in Config.
- **D-9. Naming and location.** Resolved: V4 is named **POC v4** (FR-Z), evolving the current app's codebase. `poc-v3` is the revert tag. Folder and branding move from "full-scale" to a POC name (handled in the implementation plan).

---

## 9. Definition of done

- The 48-hour feed plays at the specified speeds with the early-turnaround cascade present.
- Continuous optimization and towing can be toggled live and assign incoming flights optimally while respecting the on-ground-reassignment policy.
- All six disruption types can be injected over the 48 hours, are logged, and are replayable.
- Trade-off strategies can be switched live within the selected minimum cadence.
- The Value tab accrues realized cost during playback, shows the minute-by-minute cumulative and the total after a full run, and the base solve and prior tabs are unaffected.
- Two runs under different settings can be saved and compared on one chart with their totals.
- No console errors; the app stays interactive; synthetic-data and assumption labels are present.
