# HANDOVER — POC v4 (continuous optimizer)

Resume point for the `poc-v4/` app: the continuous rolling-horizon stand-reallocation copilot.
Read this first. For the broader case context (deck, Copilot Studio, the original `poc/` snapshot demo) read `HANDOVER.md`. The deep dev log is `v2_build_notes.md`.

Last updated: 2026-06-09. Interview / live demo: Friday 2026-06-12.

---

## 1. What POC v4 is

An advisory, human-in-the-loop **Apron Optimizer / Airside Disruption Copilot**. It runs a continuous rolling-horizon optimizer over a synthetic 48-hour CPH schedule. As the feed clock advances, the optimizer re-solves a bounded stand-allocation MIP every cadence, proposes reassignments when a disruption breaks the plan, and the operator approves them. The airport never autonomously cancels or diverts an airline flight; the tool flags candidates and escalates.

This is the evolution of the original `poc/` build (a single 14:20 snapshot). POC v4 replaces that snapshot world with one continuous world. The whole app is now unified on it (see section 5).

- **Single file:** `poc-v4/index.html` (~3,440 lines). Self-contained HTML/CSS/vanilla JS, offline, no framework.
- **Solver:** `glpk.js` v5.0.0 (GLPK MIP via WebAssembly), vendored at `poc-v4/assets/glpk.js`.
- **Determinism:** the 48h feed is seeded (mulberry32, seed 20260612). Every picture is a pure function of the feed clock + `v4sim` state.

---

## 2. Where it runs

- Preview server **`cph-pocv4`** on **port 8124**, serving `poc-v4/` (see `.claude/launch.json`).
- URL: `http://localhost:8124`.
- Also runnable by opening `poc-v4/index.html` directly in a browser.
- Companion servers: `cph-pocv3` (8123, the retired V3 snapshot revert copy in `poc-v3/`), `deck` (8771), `cph-arch` (8126, the architecture diagram in `architecture/`).

---

## 3. The case scenario it must support

A busy afternoon. A central stand goes out of service, aircraft arrive off-slot, departures slip into the next turnaround, and a ground handler cannot service all simultaneous turnarounds. Airside Operations must decide which aircraft go to which stands and which flights to escalate. Decisions are interdependent across passenger flow, gates, passport control (Schengen / non-Schengen), baggage, crew, and handling.

The four disruptions from the case brief, in the app, map to the inject buttons in section 7.

---

## 4. Architecture (the intended pipeline)

The user's signed-off architecture (rendered in `architecture/`) is:

Live feed → ML timing prediction → MIP optimizer → Pareto chooser → Human commit → Live operating picture → Revalidation / self-heal.

Note: the **greedy planner** (`assignPlannedStands`) is NOT an architecture component. It is the synthetic stand-in for the AODB's pre-planned allocation and the optimization-OFF baseline. Do not present it as part of the pipeline.

---

## 5. Completed work

### Phases V4.2–V4.8 (pre-unification)
- **V4.2** — 48h feed time engine with play / scrub / speed controls.
- **V4.3** — continuous rolling-horizon optimizer (re-solve on cadence).
- **V4.4** — six injectable disruptions.
- **V4.5** — strategy control (5 presets) + cadence lock.
- **V4.6** — realized (incurred) cost ledger, accruing all four cost drivers.
- **V4.7** — run save / compare via localStorage.
- **V4.8** — reframe to "POC v4", folder moved `full-scale/` → `poc-v4/`, docs reconciled.

### Adversarial bug sweep (commit `4ecaf35`)
Five real bugs fixed: stale moves leaking across toggle-off/Reset; handler-cap costing €0; escape-hatch ignored; OOS desync on scrub-back; gate dedup in the ledger. Three false positives dismissed.

### Unification U1–U8 (commits `2679031` → `3c714af`)
The app carried two generations layered together (a V3 14:20 snapshot world driving most tabs, plus the V4 continuous world). Fully unified onto V4: one clock (`v4.now`), one data source (`FEED` + `v4sim`), one disruption system. Every surface (sidebar, apron, flights, optimize, trade-offs, Downstream Projection, event log) now derives from the feed at the current clock. Dead V3 code removed. Tag `poc-v5` marks the post-unification state; `poc-v4` is the pre-unification revert point; `poc-v3` is the original snapshot.

### Post-unification refinements
- **Apply to the live plan** (`84e35fe`): operator approval button on the Optimize tab; commits the preview to `v4sim.assign`, logs "operator approved" moves, sets `v4sim.applied=true`.
- **UX fixes** (`605875e`): global feed-clock bar at the top; fixed the paused strategy-lock trap; clearer realized-cost labels; Adaptive strategy no longer blocks the other trade-off presets.
- **Value reframe + planner fix** (`0dc03a4`): the greedy planner was leaving 4 flights unplaceable (€78,290 phantom strand cost that swamped the optimizer's contribution). Fixed `assignPlannedStands` to best-fit + remote fallback → 0 unplaceable, baseline strand €0. Reframed the Value Recovery tab from "total cost" to **"value delivered"** — what the copilot recovers when the plan breaks.
- **Sankey** (`44de47e`, `7b1ad1d`, `ea37f30`): restored the passenger-flow Sankey in the Downstream Projection tab, then made it live and feed-grounded (real Schengen/non-Schengen split from the feed), then linked at-risk passengers to their onward departures so you can see which connections are threatened.

### Handler-capacity-cut rebuild (commit `33ae1d0`)
Rebuilt the handler-capacity-cut disruption. Ground handler crew is modelled as a fixed number of parallel teams that serve turnarounds first-come-first-served by on-block time. When more simultaneous turnarounds arrive than there are teams, the excess queue, run long, and their departures slip. The delay is priced into the realized ledger and the stand is held longer. This is the one disruption stand-reallocation cannot fix (a handler is contracted per flight, so moving stands does not free crew), so the tool detects, flags, and costs it rather than optimizing it away. Same commit added the inject-button tooltips and the "Disruptions you can inject" section of the Definitions legend.

### Adversarial bug sweep (commit `eca3e8b`)
Full adversarial sweep of the unified app. Six correctness issues fixed:
1. Handler-cap delay is now priced into the realized ledger. It was invisible because it sat on the departure leg; it is now attributed to the arrival and the stand is held longer.
2. Escalation is now costed on the Trade-offs / Pareto chart and in plan cost. It was free.
3. An aircraft already parked when its stand fails is now counted as stranded. It was only checked at on-block.
4. Backward scrub now rebuilds a clean feed and replays disruptions. Timing mutations were not being reverted.
5. `eur()` no longer renders "NaN".
6. The 16-flight decision-cap overflow is now surfaced as a "Held beyond cap N" chip.

### UI renames + projection removal (commit `9e81b73`)
Renamed the **Digital twin** tab to **Downstream Projection** and the **Value** tab to **Value Recovery**. Removed the Value Recovery tab's "Four-case projection" view; the tab now carries two views, "Value delivered" and "Realized cost · 48h run".

---

## 6. Value scenario (verified numbers for the demo)

The Value Recovery tab picks the busiest contact stand and takes it offline for 2h. Verified instance: stand A8 offline 15:16, 3 flights / 100 pax exposed, €39,680 exposure, €30,180 recovered per event, scaling to ~€1.81M/year at 60 events. These are conservative (the value computation uses synchronous greedy rehoming, no solver, so it is a lower bound).

---

## 7. The six disruptions, mapped to the case's four

Each is a `{type, time, target, params, duration}` record applied as the clock passes it (`v4ApplyOne`), replayable, and the optimizer reacts on the next cadence.

| Case brief (Danish) | Inject button(s) | Mechanism | Status |
|---|---|---|---|
| Central standplads ud af drift | **Stand offline** | busiest occupied contact stand → `v4sim.oos` for 120 min; optimizer relocates committed flights, strands/remotes the rest | Solid. Headline value scenario. |
| Fly ankommer uden for slots | **Early arrival** / **Late arrival** | next inbound `predOn` ±25/40 min; late also adds to `delayMin` | Solid. |
| Afgange forsinkes, påvirker næste turnaround | **Departure delay** (+ **Longer turnaround**) | next departure off-block +35 min, or dwell +40 min; cascades via the rotation link | Solid. |
| Ground handlers kan ikke håndtere alle samtidige turnarounds | **Handler capacity cut** | see below | Solid. The disruption the optimizer cannot fix; tool detects, flags, and costs it. |

### Handler capacity cut — mechanism (rebuilt 2026-06-09, commit `33ae1d0`)
Ground handler crew is modelled as a fixed number of parallel teams serving turnarounds first-come-first-served by on-block time. When more simultaneous turnarounds arrive than there are teams, the excess queue, run long, and their departures slip. The delay is priced into the realized ledger and the stand is held longer.

This models **simultaneity**, not total count in a window: the constraint binds when too many turnarounds overlap at the same moment. It extends the **turnaround** (dwell) and slips the **departure**, reusing the same rotation-cascade machinery as the departure-delay disruption.

This is the one disruption the **stand optimizer cannot fix**. A handler is contracted per flight, so moving stands does not free crew. The tool's role is **detect, flag, and cost** it, not optimize it away. That is a clean governance / scope point for the interview.

---

## 8. Things tried that failed (and the fix)

- **Two `renderFeed()` functions** (V3 + new V4) collided on hoisting. Fix: renamed V4 functions `v4Render` / `v4Controls` / `v4Gantt` / `v4Frame`.
- **glpk.js is synchronous** — it has no `tmlim` / `mipgap`, only `msglev`. A hard set-packing instance freezes the tab. Fix: bound the decision set to 16 (`V4_DECISION_CAP`). For the value computation, avoided the solver entirely (greedy rehoming) to dodge the hang on dense peak instances.
- **Value-delivered initially used `v4SolveWindow`** and hung on the busiest-stand-offline peak. Fix: rewrote `v4ValueDelivered` as synchronous greedy.
- **`showTab` didn't render the repointed tabs** (frozen at startup `v4.now=0`). Fix: render-on-open dispatch in `showTab`.
- **`applyConfig` still called a removed V3 `runOptimization`.** Fix: repointed to `v4MaybeSolve`.
- **Optimizer appeared to add €0 value** across two runs. Root cause: 97.6% of cost was structural (4 unplaceable flights + dataset delay-minutes the ledger doesn't price as soft cost). Fix: completed the planner (0 strands) and reframed value to recovery-when-the-plan-breaks.

### Test-environment gotchas
- The preview MCP throttles a **background tab**: RAF pauses, `setTimeout` clamps to ~1s. Polling loops and full 48h solve sweeps time out in tests. Single foreground solves are ~20–150ms. Verify with bounded counts, idle-waits, or screenshots brought to the foreground.
- The preview **viewport keeps resetting** (screenshots often come back blank or zoomed). This is a tool quirk, not an app bug. Resize to 1280×900 before screenshotting; rely on `preview_eval` DOM assertions for ground truth.

---

## 9. Pending items

1. **Deferred dead-cluster removal** (low priority): the dead V3 cluster (entangled escalation-modal + pareto-signature / `__selectedStrategy`) is documented but not requested for removal.
2. **Optimize-tab boundary alignment** (deferred): the optimize tab's window boundary does not line up cleanly with the solver window edge. Documented, not yet addressed.
3. **Load-run auto-replay** (deferred): loading a saved run does not auto-replay its disruptions; the operator must re-trigger them. Documented, not yet addressed.
4. **Deck / docs reconciliation:** the deck (`deck/deck.html`, 16 Danish slides) and `Solution Overview and Assumptions.md` still reference the V3 14:20 B16/EK151 snapshot as an illustrative example. Assessed as valid (it is a worked example), no rewrite made. Revisit if the demo narrative changes.

---

## 10. Uncommitted Git changes (as of handover)

Branch `main`, **28 commits ahead of origin** (not pushed — push only on explicit confirmation). This session's commits: `33ae1d0` (handler-cap rebuild + legend / tooltips), `eca3e8b` (bug sweep), `9e81b73` (UI renames + projection removal).

- **Modified, unstaged:** `.claude/launch.json` (+6 lines — added the `cph-pocv3` config and reordered ports). Safe to commit.
- **Untracked (not yet added):**
  - `Case - AI Business Lead - Operations.pdf` — the original case brief.
  - `HANDOVER.md`, `HANDOVER_2026-06-09_v3-vs-v4.md` — earlier handovers.
  - `architecture/` — the architecture-diagram mini-site (served on `cph-arch`).
  - `opensky-feed/` — OpenSky feed builder (`build_feed.py`, raw + out). Real-data feed experiment.
  - `poc-v1/` — an early POC iteration kept for reference.
  - `agent_instructions.md`, `copilot_studio_build_guide.md`, `qa_brief_da.md`, `speaker_notes_da.md` — case prep docs.

Nothing is staged. The last POC v4 code change committed was `9e81b73` (UI renames + projection removal). The handler-cap rebuild and the adversarial bug sweep are both committed (`33ae1d0`, `eca3e8b`). The working tree has no uncommitted changes to `poc-v4/index.html`.

---

## 11. Key code map (`poc-v4/index.html`)

- **State:** `v4` (clock), `v4sim` (optimizer/disruption state: `optimize`, `applied`, `assign`, `moves`, `oos`, `disruptions`, `handlerCap`, `strategy`, `weights`, `cadenceLock`).
- **Feed:** `FEED` = 48h schedule from `generate48hSchedule()` (253 arrivals + departures). Seed 20260612.
- **Realized stand:** `v4StandOf(f)` returns `v4sim.assign[f.id] || f.plannedStand` when optimize or applied is on, else the planned stand.
- **Solver:** `V4_DECISION_CAP = 16`; `v4BuildWindowModel(t,w,glpk)`; `v4Solve(t)` (mutating, continuous); `v4SolveWindow(t,w)` (non-mutating preview).
- **Cost:** `v4CostParts(f,s)` → {passenger, handler, schedule}; `v4CostW(f,s,w)`; `v4CostFS(f,s)`.
- **Picture helpers:** `feedArrivalsInWindow`, `onStandAt`, `standOccupant`, `standOfflineNow`, `freeStandsAt`, `needsDecision`, `feedFlightsAround`.
- **Planner (baseline, not architecture):** `assignPlannedStands` — best-fit + remote fallback; produces `plannedStand`.
- **Ledger:** `v4StandOfflineAt`, `v4LedgerEvents` (4 drivers: delay, bus, strand, escalation), `v4LedgerCurve`, `v4RealizedCost`.
- **Optimize tab:** `v4OptimizeNow()` (preview), `v4ApplyOptimize()` (commit / operator approval).
- **Value Recovery tab:** `valueView`, `v4PickValueScenario`, `v4ValueDelivered` (synchronous greedy), `renderDeliveredHTML`, `v4SetValueEvents`. Two views: "Value delivered" and "Realized cost · 48h run" (the former "Four-case projection" view was removed).
- **Downstream Projection tab:** `v4TwinNow()` (KPI cards, delay bar chart, rotation cascade) + `v4Sankey()` (live passenger flow: arrivals → passport zone → on-time / at-risk → onward departures, with at-risk→departure links).
- **Disruptions:** `V4_DISRUPT` (the six types + params), `v4InjectType` (targeting), `v4ApplyOne` (mechanism), `v4ApplyDue`. Inject buttons carry tooltips, and the Definitions legend has a "Disruptions you can inject" section.
- **Controls:** `v4Controls` renders into `#globalControls`; `v4Frame` updates clock/scrub.
- **Tab dispatch:** `showTab` re-renders the opened tab at `v4.now`.

---

## 12. Constraints to preserve (from CLAUDE.md)

- No em dashes anywhere. No emoji in code/commits unless asked.
- No `git push`, PRs, `--force`, or `--no-verify` without explicit confirmation.
- No new packages without telling the user which and why.
- Check before deleting files or data.
- PDF export uses the project-local ReportLab exporter only — never docx2pdf, AppleScript, osascript, Word, or the Word MCP.
- Tracker updates are out of scope for build sessions.
- Commit trailer: `Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>`.
