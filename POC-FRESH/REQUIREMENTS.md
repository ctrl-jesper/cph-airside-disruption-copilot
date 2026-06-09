# POC-FRESH v1 — Requirements Specification

**Project:** CPH Airport, AI Business Lead (Operations) case
**Artifact:** Fresh requirements specification for the rebuilt proof of concept
**Version:** v1, draft for review
**Date:** 2026-06-09
**Author role:** Business and enterprise analyst, airside operations
**Status:** Draft. Supersedes nothing yet. The older v1 to v4 build is set aside on purpose; this spec starts from the case and the operational evidence, not from the existing code.

---

## 0. How to read this document

This is a requirements specification, not a design or a build plan. It states what the POC must do and why, in terms an Operations leader, a Data and AI specialist, and a governance owner can each check against their own concerns.

Requirements are identified so they can be traced: `FR` functional, `NFR` non-functional, `OPS` operational-domain, `GOV` governance, `DEMO` presentation, `ASM` assumption, `OOS` out of scope. Each requirement uses "shall" for a firm obligation and "should" for a strong preference. Illustrative numbers live in the assumptions tables (Part J), so the functional requirements stay parameterized and the demo numbers stay honestly labeled.

Front matter is numbered (Sections 0 to 3); the requirement body is lettered (Parts A to M).

The order follows the request: the decision engine and the learning mechanism first, then the operational dynamics that ground them, then the case goals, then everything else needed to solve and demo the case.

---

## 1. Context and problem statement

Copenhagen Airport runs Airside Operations in real time. On a busy, disrupted afternoon a duty manager must re-decide which arriving aircraft go to which stands while a stand is out of service, several aircraft arrive off their slots, departures slip into the next turnaround, and a ground handler reports it cannot service every simultaneous turnaround well. Each stand choice couples to passenger walking distance, gates, passport routing, baggage, ground handling, onward connections, and aircraft-to-stand compatibility. The best move in one place creates a problem in another, and the consequences are not visible at the moment of the decision.

Today this is handled manually by experienced staff, and the knowledge is `personbåren`, it lives in people's heads. The case asks the candidate to understand the situation, find where AI creates value, and design a solution that can be tested quickly, runs in operations, and can be developed further.

The POC answers this with a deterministic optimization engine that proposes whole-apron allocations and surfaces the genuine trade-offs, kept human-in-the-loop and read-only, plus a calibration loop that learns the operators' tacit trade-offs from the decisions they actually make.

---

## 2. Scope

**In scope.**
- Real-time stand and apron re-allocation across the apron during disruption, with the knock-on effects for passport control (Schengen and non-Schengen), baggage belts, ground handlers, and passenger connections.
- A deterministic mixed-integer optimization engine (the decision engine) with hard constraints and weighted soft costs.
- A Pareto trade-off view that returns genuinely distinct strategies (protect passengers, protect handlers, protect schedule).
- An Operator Calibration Loop that learns soft-cost weights from operator overrides, with a demo seed affordance.
- Escalation of redirect-or-cancel candidates, never an autonomous cancel or divert.
- A governance posture and a business-value view.

**Out of scope (named, not ignored).** See Part M. The headline exclusions are crew availability and positioning, landside and terminal operations beyond passport and connections, autonomous write-back to airport systems, and any airport-wide optimization outside Airside Operations.

---

## 3. Glossary

- **AODB.** Airport Operational Database. At CPH the principal system is **Airhart**.
- **A-CDM.** Airport Collaborative Decision Making. CPH is a connected A-CDM airport; milestone data (in-block, off-block, turnaround durations) is the basis for a timing model.
- **Stand / contact stand / remote stand.** A parking position. A contact stand has a jet bridge to the pier. A remote stand requires bus transfer.
- **ICAO code (A to F).** Aircraft size class by wingspan. Code C is a narrowbody (A320, B737). Code E is a large widebody (A330, B777). Code F is the A380.
- **Turnaround.** The ground process between an aircraft's on-block arrival and its off-block departure.
- **Schengen / non-Schengen.** The passport-control zoning that determines which arrivals piers and channels a flight's passengers can use.
- **Ground handler.** The contracted company servicing a turnaround (at CPH, for example Menzies, SGH, Aviator).
- **MIP.** Mixed-integer program, the optimization model that assigns flights to stands.
- **Pareto front.** The set of non-dominated allocations across competing objectives; improving one objective requires worsening another.
- **Override.** An operator changing the optimizer's proposed allocation to a different legal allocation.

---

# PART A — The decision engine (MIP optimizer)

The engine is the deterministic core. It computes feasibility and ranks legal allocations by a weighted soft cost. It never generates a recommendation from a language model and never writes to airport systems.

## A1. Hard constraints (feasibility)

A stand is a **legal** option for a flight only if every hard constraint holds. Hard constraints are inviolable and are never subject to learning or weighting.

- **FR-A1.1 Stand serviceability.** A stand that is out of service (technical fault, closed jet bridge, blocked) shall not be assignable.
- **FR-A1.2 Size-code fit.** A stand's maximum ICAO code shall be greater than or equal to the aircraft's code. A code-F aircraft (A380) shall only be assignable to a code-F stand.
- **FR-A1.3 Passport-zone fit.** A Schengen arrival shall be assignable only to a Schengen-capable contact stand; a non-Schengen arrival only to a non-Schengen-capable contact stand. Remote stands are mixed-zone because passengers are bussed to the correct control.
- **FR-A1.4 Occupancy without overlap.** No two aircraft shall occupy the same stand in overlapping time windows. The window shall account for a departure blocking its stand for a configurable pre-departure lead before off-block.
- **FR-A1.5 Equipment availability.** A contact stand with a failed jet bridge or required equipment shall be treated as out of service for aircraft that need that equipment.
- **FR-A1.6 Infeasibility handling.** When a flight has no legal stand, the engine shall mark it as an **escalation candidate** (redirect or cancel), never silently drop it and never auto-cancel it.

## A2. Soft costs (the objectives)

Among legal options, the engine ranks allocations by a weighted sum of soft costs. Each soft cost is a tunable parameter; defaults are illustrative (Part J) and are the target of the calibration loop in Part B.

- **FR-A2.1 Passenger cost.** Penalize passenger walking time from stand to the central control point, penalize remote-stand assignment (a fixed remote penalty plus the bus and handling surcharge), and penalize connections placed at risk.
- **FR-A2.2 Handler cost.** Penalize assigning a turnaround to a handler at or over capacity, and penalize servicing far from the handler's apron base (drive time and vehicle cycles).
- **FR-A2.3 Schedule and stability cost.** Penalize short stand buffers (a buffer below a low threshold heavily, a medium buffer moderately), penalize pier changes from the planned pier, penalize baggage-belt changes from the planned belt, and penalize moving a committed aircraft already on the ground.
- **FR-A2.4 Resource-thrift cost.** Penalize assigning an oversized stand to a small aircraft, per size-code gap, so scarce large stands stay free for the aircraft that require them.
- **FR-A2.5 Three named objective groups.** The soft costs shall roll up into three weighted objective groups, **passenger**, **handler**, and **schedule**, so the trade-off is expressible in three dimensions for the Pareto view and the calibration loop.

## A3. The optimization

- **FR-A3.1 Single balanced solve.** The engine shall solve the whole-apron assignment under the current weights and return one allocation, its total cost, and a per-flight cost breakdown.
- **FR-A3.2 Pareto sweep.** The engine shall produce a non-dominated front across the three objective groups by an epsilon-constraint sweep over the same model, and shall return at least three signature strategies (protect passengers, protect handlers, protect schedule).
- **FR-A3.3 Strategies are computed, not authored.** The Pareto points shall be genuine solver outputs, not hand-picked numbers. The spec acknowledges (Part J) that visible corner separation depends on the live cost weights and is a calibration property, not a hand-tuned demo property.
- **FR-A3.4 Re-solve on change.** Any change to a hard input (a stand drops, an arrival time shifts) or to the weights shall re-run the solve and update the allocation and the cost breakdown.

## A4. Engine outputs

- **FR-A4.1 Inspectable allocation.** Every assignment shall show its cost breakdown across the three objective groups, so a human can see why a stand was chosen.
- **FR-A4.2 Escalation list.** Flights with no legal stand shall appear as escalation candidates with the reason.
- **FR-A4.3 Consequence flags.** Each allocation shall surface its at-risk consequences (for example connections placed at risk, a handler pushed over capacity), so the cascade is legible before commit.

## A5. Engine non-functionals

- **NFR-A5.1 Determinism.** Given the same data, weights, and disruption set, the engine shall produce an identical allocation and cost every run, independent of machine speed. Determinism is a deliberate design property for critical-infrastructure decision support, not an incidental one.
- **NFR-A5.2 Interactivity.** A solve shall complete inside an interactive budget (target well under a second on the demo instance) so the operator sees the result without a visible stall.
- **NFR-A5.3 Self-contained.** The engine shall run locally with a vendored solver and no network dependency, consistent with a demo that must not depend on live connectivity.

---

# PART B — The Operator Calibration Loop (learning mechanism)

The loop learns the duty managers' soft-cost weights from the allocations they actually choose, and feeds them back into the engine so future recommendations reflect the team's real trade-offs. It is the mechanism that turns `personbåren` knowledge into an institutional, auditable asset. It calibrates soft weights only; it never touches the hard constraints of Part A.

## B1. Override capture (the data)

- **FR-B1.1 Decision record.** When the engine proposes allocation A and the operator chooses a different legal allocation B, the system shall record the cost-component vectors of both, `cost(A)` and `cost(B)`, across the three objective groups, with the situation context and a timestamp.
- **FR-B1.2 Signal definition.** The learning signal for one override shall be `cost(B) − cost(A)`. The operator's reasoning need not be stated for the loop to work; the choice is the signal.
- **FR-B1.3 Legality precondition.** Only overrides to a legal allocation (all Part A hard constraints satisfied) shall be recorded. An operator cannot teach the system to violate a hard constraint.
- **FR-B1.4 The reason channel.** Beyond the numeric signal, an override carries a natural-language reason. Capturing, distilling, and querying that reason is the conversational layer's responsibility; see Part B7.

## B2. The learning rule

- **FR-B2.1 Transparent update.** The system shall infer weights with an inspectable structured-perceptron update: each override where the current weights would have selected A but the operator chose B nudges the weights toward the objective group that B favored, by a bounded step, then clips and averages over the batch.
- **FR-B2.2 Explainability.** For any learned weight change, the system shall be able to state which decisions moved which weight and by how much, in plain language (for example, "passenger weight rose because the operator protected a connection at the cost of handler load in N decisions").
- **FR-B2.3 Production method named.** Full inverse optimization (fitting weights by an LP or QP so the observed overrides are optimal or minimally suboptimal) shall be named as the production-grade version of the same idea, with the perceptron rule as the legible PoC stand-in.

## B3. Demo seed affordance

- **FR-B3.1 Seed-history control.** The POC shall provide a "seed operator history" control that injects a batch of synthetic overrides at once, so the calibration effect is visible in seconds rather than requiring dozens of live overrides.
- **FR-B3.2 Hidden true profile.** The seed shall draw overrides from a hidden "true" profile for a named duty manager (for example, one who weights passenger connections above the house default), so the demo can show the system recovering a preference it was never told.
- **FR-B3.3 Visible recovery.** As the batch is processed, the POC shall show the weights traveling from the default toward the hidden profile, the live allocation shifting as a result (for example, a flight moving off a remote stand to a contact stand), and a readout of the recovered profile against the house default.
- **FR-B3.4 Audit of learning.** The POC shall list which seeded decisions drove the change, so the learning step is auditable on screen.

## B4. Profiles and the governed house style

- **FR-B4.1 Per-operator capture.** The system shall capture a per-operator preference profile from each duty manager's own overrides.
- **FR-B4.2 Governed consolidation.** Only a governed **house style** weighting shall drive live recommendations. Per-operator profiles feed a review step that consolidates them into the house style; they do not silently drive operations.
- **FR-B4.3 Approval before live.** A learned recalibration shall be proposed, reviewed, and approved by a named owner before it takes effect, the same human-in-the-loop stance as the allocations. The learning step shall not auto-commit.

## B5. Calibration guardrails

- **GOV-B5.1 Soft weights only.** Learning shall change soft weights only. Hard constraints (Part A1) are immovable.
- **GOV-B5.2 Bounds and regularization.** Learned weights shall be bounded to a sane positive range and regularized toward the default, so a handful of overrides cannot swing the system. It shall take a body of evidence to move the house style. (This is also why the demo uses a seeded batch; see Part J, honesty note.)
- **GOV-B5.3 Traceability.** Every learned change shall link to the specific decisions that produced it, as the audit trail for Article 14 oversight.
- **GOV-B5.4 Personal data.** Per-operator profiles are personal data under GDPR. The profile store shall be access-controlled and the obligation shall be stated, not waved away.

## B6. PoC versus production boundary for the loop

| Dimension | PoC-FRESH (build) | Production (roadmap) |
|---|---|---|
| Override source | Synthetic, via the seed control | Passive capture from the live accept/override flow |
| Inference | Transparent perceptron rule | Regularized inverse optimization with confidence |
| Profiles | One operator's hidden profile recovered | Per-operator, consolidated to house style under governance |
| Rationale | Explain and query live; capture-and-distill described | Full capture, distill, and govern, with natural-language "why" on each override |
| Hosting | Client-side in the POC app | Azure ML for learning, Dataverse for the override store with Entra ID RBAC and Microsoft Purview classification |

## B7. Conversational layer and tacit capture (the two channels)

An override has two separable channels. The **choice** is numeric (`cost(B) − cost(A)`) and drives calibration (Part B2). The **reason** is natural language and is the conversational layer's domain. Keeping the language model out of the choice-and-weight path is what preserves determinism and auditability.

- **GOV-B7.1 The LLM never decides.** The conversational layer (Copilot Studio) shall never compute an allocation, commit a decision, or change a MIP weight from free text. It works on language only.
- **FR-B7.2 Explain.** It shall narrate the engine's allocation and per-flight cost breakdown in plain language and answer "why this stand", reading computed output, not generating it.
- **FR-B7.3 Capture the reason.** At an override it shall capture the operator's natural-language rationale, stored verbatim with structured tags, enriching the numeric signal and surfacing considerations outside the three objective groups (for example a security rule, or an unreliable jet bridge).
- **FR-B7.4 Distill into reviewable knowledge.** It shall cluster captured rationales into recurring themes that become candidate model features or governance rules, as proposals for human review, never automatic changes.
- **FR-B7.5 Queryable institutional memory.** It shall answer operator questions from captured rationale plus engine logic, so person-bound knowledge becomes searchable and onboarding is supported.
- **FR-B7.6 Fidelity guardrail.** Rationale shall be stored verbatim; any LLM summary is a derived view subject to human review, so the model does not become the source of truth.
- **FR-B7.7 PoC scope.** With work-tenant access assumed available (ASM-8), the PoC builds a live Copilot Studio agent for explain (FR-B7.2) and query (FR-B7.5). The capture-and-distill loop (FR-B7.3, FR-B7.4) is demonstrated as narrative and is production roadmap.

---

# PART C — Operational dynamics (airside operations grounding)

This part records the operational reality the engine must respect, drawn from the operational analysis across the prior sessions. It is written so an experienced Airside operator would recognize it as faithful, with illustrative parameters flagged as such.

## C1. The apron and stand inventory

- **OPS-C1.1 Mixed contact and remote apron.** The model shall represent a realistic CPH-like apron: contact stands grouped by pier, with Schengen piers, a non-Schengen pier, and mixed or remote positions. Illustrative scale is roughly 18 contact stands plus a remote apron.
- **OPS-C1.2 The single code-F stand.** The model shall include exactly one code-F stand, on the remote apron, as the only position that can take an A380. This single scarce resource is a deliberate pressure point.
- **OPS-C1.3 Pier and zone attributes.** Each stand shall carry its pier, its passport zone (Schengen, non-Schengen, or mixed-remote), its maximum size code, its walking distance to the central control point, and its serviceability state.

## C2. Aircraft-to-stand compatibility

- **OPS-C2.1 Size compatibility is physical.** Larger aircraft fit fewer stands. A code-F or code-E aircraft can use only a small subset of positions; a narrowbody can use most. This scarcity is the source of much of the contention and is a hard constraint, not a preference.
- **OPS-C2.2 Zone routing is regulatory.** Schengen and non-Schengen passengers must reach the correct passport control. A contact stand is tied to a zone; a remote stand defers the zoning to the bus destination. This is a hard constraint.
- **OPS-C2.3 Oversize is wasteful, not illegal.** Putting a small aircraft on a large stand is legal but penalized, because it consumes a scarce resource (FR-A2.4).

## C3. Turnaround and occupancy dynamics

- **OPS-C3.1 Occupancy windows.** A stand is occupied from on-block to off-block, with a pre-departure lead during which the departing aircraft blocks the stand. Turnaround durations vary by aircraft type (illustratively, a narrowbody around 45 minutes, a widebody around 90 minutes).
- **OPS-C3.2 Knock-on slip.** A delayed departure extends its stand occupancy and can collide with the next planned arrival on that stand, forcing a re-allocation. The model shall represent this chaining so a single delay can cascade.
- **OPS-C3.3 Buffers absorb small disruptions.** Short delays are normally absorbed by stand buffers, not by re-allocation, because moving an aircraft carries passenger-movement and handling cost. Re-allocation is for genuine conflicts, not for every minute of slip. This shapes when the engine should propose a move at all.

## C4. The dependency web

The engine's value rests on representing the couplings an experienced operator holds in their head. Each is named with how it constrains the stand decision.

- **OPS-C4.1 Passenger flow and walking distance.** Stand choice sets how far arriving passengers walk and how long they take to reach passport control. This drives both passenger experience and connection feasibility. Modeled as a soft cost and as a connection-risk input.
- **OPS-C4.2 Passport control, Schengen and non-Schengen.** Zone routing is a hard constraint; the queue load at each control is a consequence of the allocation. An A380 sent to remote can spike the non-Schengen queue at peak, putting connecting passengers at risk. Modeled as a hard constraint plus a downstream consequence.
- **OPS-C4.3 Baggage.** CPH's baggage system (illustratively a 10 km belt and sortation system with capacity bottlenecks) ties a flight to a planned belt; a belt change carries cost and a belt has finite capacity. Modeled as a soft cost and a consequence, not optimized in the PoC.
- **OPS-C4.4 Ground handling.** Each turnaround needs a handler with crews and equipment. A handler at capacity cannot service every simultaneous turnaround well; servicing far from base adds drive time. Modeled as a soft cost and a capacity pressure surfaced for human coordination, not an automatic block.
- **OPS-C4.5 Connections.** A connecting passenger is at risk when walking time plus modeled passport wait plus a fixed overhead exceeds the time to their onward flight. Modeled as a derived consequence in the twin or connection view, flagged before commit.
- **OPS-C4.6 Gates and pier continuity.** A pier change from plan reorganizes gate use and lengthens passenger walks. Modeled as a soft cost.
- **OPS-C4.7 Crew (named boundary).** Crew duty limits, positioning, and qualification live in airline crewing systems, not the airport AODB. The PoC shall not model crew and shall state this as a boundary requiring airline data sharing, with the ground handler as the visible resource proxy.

## C5. The disruption cascade scenario

- **OPS-C5.1 Reference scenario.** The demo shall run a constructed but realistic busy-afternoon scenario: high traffic with existing delays, a central stand taken out of service on a technical fault, an A380 arriving off-slot that needs the single code-F stand, further departure slips that push the next turnaround, and a ground handler reporting it cannot service every simultaneous turnaround. The disruption types and their behavior are specified in C7.
- **OPS-C5.2 The three case decisions visible.** The scenario shall make all three decisions the case names appear on screen: which aircraft to which stands, which to redirect or cancel (an escalation), and which compromises are necessary (the trade-off).
- **OPS-C5.3 A second-order effect.** The scenario shall reveal at least one non-obvious downstream consequence that a snapshot view would miss (for example, the remote A380 spiking a passport queue and placing connections at risk), to show why the dependency web matters.

## C6. Fidelity statement

- **OPS-C6.1 Real where it counts, synthetic where labeled.** The engine logic, the constraints, the Pareto sweep, and the cascade shall be genuinely computed. The flight schedule, the timing layer, the live feed, handler capacities, passport capacities, and all cost and euro units shall be synthetic and labeled in the app. EK151, the real Emirates A380 from Dubai, genuinely needs the single code-F stand and may be cited as a real requirement.

## C7. Disruption mechanics

How disruptions behave, carried over from the V4 POC work (session 41ff75f7) and refined there on 2026-06-09 (the handler-capacity rebuild, V4 commit 33ae1d0). These mechanics are the operational engine behind the reference scenario (C5). Magnitudes and durations are illustrative defaults (ASM-9), tunable in config.

**Mapping the case brief to mechanisms.** The case names four disruptions; they map to the six types as:

| Case brief | Mechanism(s) | Operational impact |
|---|---|---|
| En central standplads tages ud af drift | Stand offline (C7.2.1) | Committed aircraft forced to relocate or escalate; remote pushes lengthen walks and tighten connections |
| Flere fly ankommer uden for deres planlagte slots | Early arrival + Late arrival (C7.2.2, C7.2.3) | Occupancy window shifts; early can conflict with the prior occupant, late carries delay down the rotation |
| Enkelte afgange forsinkes, påvirker næste turnaround | Departure delay + Longer turnaround (C7.2.4, C7.2.5) | Aircraft holds its stand longer; the linked next leg slips, or it overruns the next planned arrival |
| Ground handlers kan ikke håndtere alle samtidige turnarounds | Handler capacity cut (C7.2.6, concurrency model C7.4) | Simultaneous excess turnarounds queue, run long, and slip departures; not fixable by the stand lever (C7.5) |

- **OPS-C7.1 A disruption is a replayable record.** Each disruption shall be a timestamped, parameterized record of the form `{type, time, target, params, duration}`. The system shall apply it when the clock reaches its time and, where it has a duration, auto-clear it when the duration elapses. The same disruption set shall replay identically, which is what makes run-to-run value comparison and the demo reproducible (ties to NFR-A5.1, FR-I1).

- **OPS-C7.2 The six disruption types.** The POC shall support each type with the stated behavior:
  1. **Stand offline.** A contact stand goes out of service for a duration (default 120 min, auto-restores). Committed aircraft on it become forced re-decisions; flights planned onto it during the outage relocate or escalate. This is the headline value scenario.
  2. **Early arrival.** The next inbound's predicted on-block moves earlier (default 25 min); its stand-occupancy window shifts forward.
  3. **Late arrival.** The next inbound's on-block moves later (default 40 min) and the delay carries along the aircraft's rotation.
  4. **Departure delay.** The next departure's off-block slips (default 35 min); the aircraft holds its stand longer and its next leg slips.
  5. **Longer turnaround.** Dwell extends (default 40 min); the aircraft can overrun into the next planned arrival on that stand, forcing a reassignment.
  6. **Handler capacity cut.** The busiest handler loses crew teams (default minus 2) for a duration (default 120 min), modeled per OPS-C7.4.

- **OPS-C7.3 Cascade via rotation links.** Each aircraft shall carry a rotation link to its next leg. A timing disruption shall propagate: a delayed inbound shifts the input window the next solve sees, and a held stand or extended dwell slips the linked departure, which slips the next rotation. The cascade is the point; the model shall represent it rather than treat flights independently.

- **OPS-C7.4 Handler capacity as concurrent crew load (the refined mechanic).** Handler capacity shall be modeled as parallel crew teams serving turnarounds first-come-first-served, not as a total count over a window. For a handler's turnarounds, when the number being serviced simultaneously exceeds the available teams, the excess queue: each waiting turnaround starts only when a team frees, which extends its dwell and slips its linked departure and delay minutes in proportion to the wait. This makes "cannot handle all simultaneous turnarounds optimally" literal: it is simultaneity that hurts, not volume.

- **OPS-C7.5 The stand lever cannot fix handler capacity (scope and governance point).** Handler capacity is the one disruption the stand optimizer cannot solve, because a handler is contracted per flight, so moving an aircraft to a different stand does not free crew. For this disruption the system's role shall be to detect, flag, and cost the simultaneous-turnaround conflict so the duty manager can act (call in crew, re-sequence), not to optimize it away. This is a deliberate scope boundary, and a clean interview point: the optimizer's lever is stands; crew conflicts need a human lever. It pairs with the crew boundary (OPS-C4.7).

- **OPS-C7.6 Optimizer response.** When continuous re-solving is enabled, an applied disruption shall trigger a re-solve on the next cadence; the engine sees the disrupted state (shifted times, reduced handler teams, offline stands) and proposes reassignments or escalations, each logged with a reason and priced in the value view. A snapshot build applies the disruption and re-solves once; the mechanics are the same.

---

# PART D — Case primary and secondary goals

This part maps requirements to what the case assesses, so coverage is checkable against the brief. The case scores realism, pragmatism, value, prioritization, and collaboration, and asks the presentation to cover problem understanding, solution idea, PoC versus full scale, governance and compliance and operations, and business value.

## D1. Primary goal

- **FR-D1.1 Solve the stated decision.** The POC shall design and demonstrate AI-supported real-time stand allocation under disruption that a duty manager can trust, addressing the case's three decisions (allocate, redirect or cancel, compromise), testable quickly, runnable in operations, and extendable.

## D2. Secondary goals

- **FR-D2.1 Capture person-bound knowledge.** The POC shall demonstrate the Operator Calibration Loop (Part B) as the direct answer to `personbåren` knowledge, turning individual tacit trade-offs into a governed institutional asset.
- **FR-D2.2 PoC-to-production judgment.** The POC shall state explicitly what is built versus what is handed to Digital and Data and AI, mapping each component to its Microsoft-stack production target and naming the owner.
- **FR-D2.3 Microsoft-first fit.** The solution shall fit the CPH Microsoft-first stack: a conversational Copilot Studio layer on M365, an Azure home for the engine and the ML, the Power Platform shell, and a governed read-only integration to Airhart and A-CDM.
- **FR-D2.4 Human-in-command.** The solution shall keep a human in the loop and remain read-only and advisory; nothing commits to airport systems automatically.

## D3. Presentation coverage

- **DEMO-D3.1 Five required areas.** The presentation shall cover problem understanding, solution idea, PoC versus full scale, governance and compliance and operations, and business value, each traceable to a requirement in this spec.

---

# PART E — Demo and presentation requirements

## E1. The ten-minute constraint

- **DEMO-E1.1 Hard cap.** The presentation shall fit ten minutes. The live demo shall collapse to one thread of roughly three minutes: inject one disruption, re-optimize, reveal a second-order effect, show the trade-off, then seed the operator history and show the weights and allocation calibrate.
- **DEMO-E1.2 No surface beyond the thread.** Features outside the demo thread shall be described, not clicked through, to protect the time budget. The build shall not add demo surface that cannot be shown inside the cap.
- **DEMO-E1.3 One-click case scenario.** The POC shall provide a single "Run the case scenario" control that injects the full compound brief at once as one replayable disruption set: the central stand out of service, a wave of off-slot arrivals (early and late), a departure delay that pushes the next turnaround, and the handler capacity cut (C7). It shall be deterministic, reproducing the same cascade each run, so the demo thread is one action rather than many clicks. Individual injects (C7.2) remain available for explaining a single mechanism.

## E2. The Microsoft-first bridge

- **DEMO-E2.1 Copilot Studio layer.** The solution shall include a live Copilot Studio conversational agent on the CPH Microsoft stack (dependency: work-tenant M365 and Entra access, assumed available; see ASM-8 and Q-4). It shall narrate and explain the engine's output, answer operator questions from the scenario and captured knowledge (the two channels, Part B7), and be presented as the interface layer of the hybrid architecture, never the decision-maker. If tenant access slips, it falls back to an authored artifact (knowledge pack and agent instructions) for the demo.
- **DEMO-E2.2 Tokenomics and cost envelope.** The presentation shall include a cost view: the engine compute is effectively free (client-side or a cheap Azure function), and the real running cost sits in the LLM interface layer (Copilot Studio consumption, M365 tokenomics), with a rough PoC-versus-production cost envelope.

## E3. Honesty and limitations to state

- **DEMO-E3.1 State the boundaries.** The presentation shall state plainly: crew is not modeled and why; soft costs are dimensionless until calibrated, and the calibration loop is how they get calibrated from real decisions; the data is synthetic with real CPH carriers and routes; the Pareto corner separation is a calibration property; and a few live overrides would barely move the weights, which is why the demo seeds a batch.

---

# PART F — Non-functional requirements

- **NFR-F1 Determinism and reproducibility.** Repeated runs on the same inputs shall reproduce the same outputs and the same learned weights from the same seeded history. (Reinforces NFR-A5.1.)
- **NFR-F2 Performance.** The app shall stay interactive during the demo; solves and calibration updates shall complete inside the demo's tempo without a visible stall.
- **NFR-F3 Offline and self-contained.** The POC shall run as a self-contained app with no live network dependency, so the demo cannot fail on connectivity.
- **NFR-F4 Legibility.** Every computed result shown to a human (allocation, cost breakdown, learned weight change) shall be explainable in plain language on screen. Legibility is a first-class requirement because the audience includes non-technical Operations leaders.
- **NFR-F5 Brand and accessibility.** The app shall use the CPH brand (the official palette, the CPH Airfield and Open Sans typefaces, the logo) and shall present text and color with adequate contrast for a presentation setting.

---

# PART G — Governance and compliance requirements

- **GOV-G1 Risk classification.** The solution shall classify itself honestly: a read-only advisory allocator is most likely limited-risk under the EU AI Act, not Annex III high-risk, because it is not a safety component making autonomous decisions. The threshold to high-risk is autonomy (write-back to Airhart, auto-cancel or divert), with Article 14 human oversight as the design boundary.
- **GOV-G2 Human oversight by design.** Article 14 oversight shall apply voluntarily even at limited-risk: nothing auto-commits, the operator reviews and accepts, and the learning step is itself reviewed and approved (Part B4).
- **GOV-G3 Transparency.** The conversational layer shall meet Article 50 transparency (users know they are interacting with an AI). General-purpose-AI obligations sit with Microsoft as the model provider, not with CPH as the deployer; the spec shall state this division.
- **GOV-G4 Data protection.** Passenger, connection, and operator-profile data shall be treated under GDPR. Operator profiles (Part B4) are personal data and shall be access-controlled.
- **GOV-G5 Co-determination.** Danish co-determination and works-council consultation (`samarbejdsudvalg`) shall be named as a real obligation, since the tool changes how experienced staff work and learns from their decisions.
- **GOV-G6 Auditability.** Every allocation accepted, every escalation, and every learned weight change shall be logged with its reason, inside CPH's own tenant in production (Dataverse with Entra ID RBAC, Purview classification).

---

# PART H — Production architecture mapping (build versus involve others)

This part satisfies the case's "what can you build yourself, where do you involve others" requirement and the PoC-to-production judgment.

| PoC component | Production target (Microsoft stack) | Who owns it |
|---|---|---|
| Synthetic AODB snapshot | Governed read-only feed from Airhart and A-CDM into Azure, landed in Dataverse or Azure SQL | Digital integration specialists, Airhart system owner |
| Simulated clock and feed | Azure Event Hubs or Service Bus for the live event stream | Digital |
| Timing layer (synthetic) | Azure Machine Learning, trained on historical A-CDM, deployed as a managed endpoint | Data Scientists in Data and AI |
| MIP optimizer (in-browser) | Azure Function or Container App behind API Management, surfaced to Power Platform via a custom connector | Candidate for PoC logic, Digital to productionize |
| Calibration loop (perceptron) | Azure ML inverse-optimization service; override store in Dataverse | Data and AI, with Governance |
| Operator app (Gantt, Pareto, calibration) | Power Apps shell plus a hosted web component for the heavy visuals | Candidate plus a Power Platform maker |
| Value view | Power BI | Candidate plus Operations |
| Audit and profile store | Dataverse with Entra ID RBAC, Microsoft Purview classification | Data Governance, Legal |
| Conversational layer | Copilot Studio agent calling the solver as an action | Candidate plus Data and AI |

---

# PART I — Business value requirements

- **FR-I1 Value is the gap versus naive allocation.** The POC shall express value as the difference between the optimized allocation and a naive baseline (first-come-first-served or hold-the-plan) under the same disruption, so the value is demonstrable rather than asserted.
- **FR-I2 Value appears under disruption.** The POC shall show, honestly, that value is near zero on an uncontended apron and grows with contention and disruption.
- **FR-I3 Illustrative quantification.** The POC may show an illustrative euro figure (for example, value recovered per stand-outage event, annualized at an assumed event rate) labeled clearly as a conservative, synthetic estimate, with the statement that calibration to real units is a production step.
- **FR-I4 The calibration loop as value.** The POC shall frame the Operator Calibration Loop as a value driver in its own right: it closes the soft-cost calibration gap from the people who already make the trade-off well, and it preserves institutional knowledge against staff turnover.

---

# PART J — Assumptions and illustrative parameters

These are explicitly synthetic or assumed. They make the demo concrete; they are not claims about real CPH operations.

- **ASM-1 Apron scale.** Roughly 18 contact stands plus a remote apron; one code-F remote stand.
- **ASM-2 Handlers.** A small set of named handlers (for example Menzies, SGH, Aviator) with assumed concurrent-turnaround capacities; one handler seeded near capacity to make the pressure visible.
- **ASM-3 Soft-cost defaults.** Illustrative penalties (for example, a per-minute walking penalty, a fixed remote-stand penalty, buffer-threshold penalties, a pier-change penalty, an oversize per-code-gap penalty). All are tunable and are the target of the calibration loop.
- **ASM-4 Passport and turnaround figures.** Illustrative passport-control throughput per interval and illustrative turnaround durations by aircraft type.
- **ASM-5 Euro units.** All euro cost units in the value view are assumptions, labeled in the app.
- **ASM-6 EK151.** The Emirates A380 from Dubai is a real service and a real code-F requirement; other flight numbers are realistic but illustrative.
- **ASM-7 Honesty note on the calibration demo.** Because learned weights are regularized (GOV-B5.2), a handful of real overrides would barely move the house style. The seed-history control injects a batch precisely to make the effect visible in the demo, and the presentation shall say so.
- **ASM-8 Work-tenant access for Copilot Studio.** A work-tenant M365 and Entra (Azure AD) account is assumed to be available so the live Copilot Studio agent (DEMO-E2.1, Part B7) can be built. A personal account cannot host Copilot Studio. If access is not ready by demo time, the fallback is an authored artifact (knowledge pack plus agent instructions). This is a dependency to track, not a settled fact.
- **ASM-9 Disruption default magnitudes and durations.** The V4 POC defaults carry over as illustrative, tunable values for the C7 mechanics: stand offline 120 min (auto-restore); handler capacity cut minus 2 crew teams for 120 min; early arrival 25 min earlier; late arrival 40 min later plus carried delay; departure delay 35 min; longer turnaround plus 40 min dwell. These are demo defaults, not measured CPH values, and each is a config parameter.

---

# PART K — Decisions and open questions

All four open questions were resolved on 2026-06-09.

- **Q-1 Approval cadence for learned weights. Resolved.** Keep the human-in-the-loop review gate (FR-B4.3): learned recalibration is proposed and approved before it goes live, not continuously adapted. Continuous silent adaptation is rejected as contrary to the governed, deterministic posture.
- **Q-2 Demo thread sequencing. Resolved.** Trade-off reveal first, then the calibration seed (DEMO-E1.1). The narrative is: here are three strategies, and here is how the team's own history decides which trade-off wins.
- **Q-3 Granularity of objective groups. Resolved.** Keep three objective groups (FR-A2.5): passenger, handler, schedule. Connections stay inside the passenger group and are surfaced as the headline consequence the passenger weight moves, so the calibration story lands without adding a fourth axis that would cost demo legibility.
- **Q-4 Copilot Studio access. Resolved by assumption.** Work-tenant M365 and Entra access is assumed available (ASM-8), so the Copilot Studio layer is built live for explain and query, with an authored artifact as the fallback if access slips. Tracked as a dependency.

---

# PART L — Definition of done (POC-FRESH v1)

The POC is demo-ready when:

1. The engine solves the reference scenario deterministically, returns a legal whole-apron allocation with per-flight cost breakdowns, and escalates any flight with no legal stand.
2. The Pareto view returns at least three genuinely distinct strategies from a real sweep.
3. The cascade reveals at least one non-obvious second-order effect on screen.
4. The seed-history control recovers a hidden operator profile, visibly moves the weights and the allocation, and shows an audit of which decisions drove the change.
5. The three case decisions (allocate, redirect or cancel, compromise) are all visible in the demo thread.
6. The governance posture, the Microsoft-stack mapping, the tokenomics cost view, and the business-value gap are each presentable and traceable to this spec.
7. The Copilot Studio layer explains a recommendation and answers one grounded operator question live, with the authored artifact as the fallback.
8. The whole presentation fits ten minutes, with a demo thread of roughly three minutes.

---

# PART M — Out of scope (named boundaries)

- **OOS-1 Crew.** Crew availability, positioning, and qualification are not modeled; they require airline data sharing (OPS-C4.7).
- **OOS-2 Autonomy.** No autonomous write-back to Airhart, no auto-cancel or auto-divert. Crossing this line crosses into EU AI Act high-risk (GOV-G1).
- **OOS-3 Landside and terminal.** Beyond passport and connections, landside and terminal operations are not modeled.
- **OOS-4 Baggage optimization.** Baggage is represented as a constraint and a consequence, not optimized.
- **OOS-5 Airport-wide optimization.** Only Airside stand allocation is in scope, not airport-wide resource optimization.
- **OOS-6 Real integrations.** No live Airhart or A-CDM feed, no trained ML model, no real cost data in v1; these are the production roadmap.
- **OOS-7 Real per-operator data.** The PoC uses a synthetic hidden profile, not real duty-manager behavioral data; real capture is a production step with GDPR and works-council obligations.
