# CPH Airside Disruption Copilot

A decision-support copilot for airport stand allocation during disruption. It re-optimizes the whole apron in one pass, lays out the genuine trade-offs instead of hiding them in a single answer, and projects a chosen plan forward in time to reveal its second-order effects before anyone commits. The system proposes and explains; a person decides. Nothing is written back to airport systems automatically.

Built as a worked solution for an "AI Business Lead, Operations" interview case at Copenhagen Airport (CPH). It runs entirely on synthetic data and is not affiliated with or endorsed by Copenhagen Airports A/S.

## The problem

A busy afternoon, a central stand fails, aircraft arrive off-slot, and one ground handler cannot service every turnaround at once. The decisions interact across stands, passport control, baggage, handlers, and passenger connections, so the best choice in one place creates a problem in another, and the consequences are not visible at the moment of decision. Today this is handled manually under time pressure, and the knowledge is person-bound.

## The approach

A hybrid system with a deterministic core and a conversational surface:

- **Optimization engine.** A real mixed-integer program assigns every flight that needs a stand to one stand, or escalates it when none fits. Hard constraints are size code, stand availability, and no two aircraft overlapping in time on a stand. Weighted soft costs rank passenger walking distance and connections, ground-handler load, and schedule stability. The solver is `glpk.js`, GLPK compiled to WebAssembly, running locally in the browser in roughly 10 to 40 milliseconds.
- **Pareto trade-off.** The three objectives genuinely conflict, so there is no single optimum. An epsilon-constraint sweep over the same model traces the non-dominated front and surfaces three signature strategies: protect passengers, protect handlers, protect schedule. The human chooses which trade-off to accept.
- **Digital twin.** A discrete forward simulation projects a chosen allocation across the afternoon and computes the passport queues, handler load, baggage belt load, connection risk, and any turnaround slip, step by step. This is the depth-over-time a snapshot optimizer cannot see.
- **Conversational interface.** A language model narrates the engine's output and captures tacit operator rules. It explains; it never computes or allocates. The recommendation is always produced by the deterministic engine.
- **Governance.** Read-only, advisory, human-in-the-loop. A person accepts a proposal, every committed assignment is revalidated continuously against the live picture, and everything is logged.

## What is real versus simulated

The credibility of the demo rests on being clear about this line.

**Genuinely computed:** the mixed-integer solver and its allocation, the Pareto sweep, the digital-twin simulation, the accept-revalidate-audit loop, and the conflict and capacity logic. EK151, the real Emirates A380 from Dubai, genuinely needs the single code-F stand.

**Synthetic, labelled as such in the app:** the ML timing layer (hand-set on-block times and uncertainty bands stand in for a model trained on A-CDM data), the live Airhart feed (a ticking clock), the flight schedule (a synthetic AODB snapshot using real CPH carriers and routes), the connection bank, and all cost units and operational parameters.

## The continuous optimizer (POC v4)

The `poc-v4/` app is a single self-contained HTML page. It begins as the V3 snapshot optimizer (whole-apron MIP, Pareto front, digital twin, tow, uncertainty, value, governance) and extends it into a continuous optimizer that runs over a 48-hour feed. Every V3 capability is governed by a tunable policy constant in a Config tab and defaults to off or neutral, so the verified base solve is unchanged until it is enabled.

The V4 continuous layer adds:

- **A 48-hour synthetic feed and time engine.** A deterministic schedule of about 250 arrivals and their linked departures on 18 stands, with diurnal traffic, aircraft rotation chains, and early-turnaround delays that cascade. A clock plays it forward at one simulated minute per real second (0.1x to 10x), and the live picture is derived from the clock.
- **Continuous rolling-horizon optimization.** As the clock plays, the apron is re-solved at a configurable cadence over a bounded window. Committed aircraft hold their stand and move only when it is lost; tiered stability keeps on-ground aircraft in place; every reassignment is logged with a reason.
- **Six injectable disruptions.** Stand offline, handler-capacity cut, early or late arrival, departure delay, and longer turnaround, injected at the current feed time, applied as the clock passes them, and recorded as a replayable set.
- **Continuous trade-off strategy control.** Five strategy presets (protect passengers, handlers, schedule, balanced, adaptive) feed the optimizer's weights, with a minimum-change cadence lock so the trade-off cannot thrash.
- **A realized cost ledger.** The cost the run actually incurs, accrued minute by minute (delay, remote bus trips, strands, gate changes), never forecast. Run save-and-compare overlays two runs so the optimize-off versus optimize-on gap, the realized value of optimizing, is drawn directly.

This is a proof of concept on synthetic data. The Governance tab states what a production build would add: real A-CDM and AODB data, a trained ML timing model, live Airhart integration with governed write-back, crew as a modelled resource, and calibrated cost units.

## Governance and EU AI Act

The tool is most likely limited-risk under the EU AI Act, not Annex III high-risk. A read-only, advisory stand allocator is not a safety component of critical infrastructure, even though an airport is. The threshold to high-risk is autonomy: writing allocations back to the airport operational database, or cancelling and diverting flights without a person. Transparency duties (Article 50) apply to the conversational interface, and GPAI obligations sit with the model provider, not the deployer. The position is to classify honestly as limited-risk, apply Article 14 oversight voluntarily, and treat crossing the threshold as a governance decision. GDPR and Danish co-determination bind in practice regardless of tier.

## Running it

Everything is static and offline. Serve any sub-app with a local web server, for example:

```bash
python3 -m http.server 8124 --directory poc-v4
# then open http://localhost:8124
```

The `poc-v4` app vendors `glpk.js` locally, so it needs no network access. The deck is a self-contained HTML file at `deck/deck.html`.

## Repository layout

| Path | What it is |
|---|---|
| `poc-v4/` | The continuous-optimizer app: the V3 snapshot core (whole-apron MIP, Pareto front, digital twin, tow, uncertainty, value, governance) plus the V4 48-hour feed, rolling-horizon optimization, injectable disruptions, strategy control, and realized-cost ledger with run compare |
| `poc/` | The proof-of-concept prototype, the shape of a Copilot Studio answer |
| `poc-v2/` | An earlier polished PoC iteration, kept for reference. See `poc-v2/README.md` |
| `deck/` | The presentation, as a self-contained HTML deck and a PDF |
| `copilot-studio/` | The Copilot Studio agent configuration, build guide, knowledge pack, and scenario |
| `Solution Overview and Assumptions.md` | The full technical reference: scope, architecture, real-versus-simulated, assumptions, limitations, standards |
| `v2_build_notes.md` | The development log: the V3 and V4 build phase by phase, with the decisions and verification behind each |
| `synthetic_aodb_*.{csv,json}` | The synthetic AODB snapshot, flights, and stands |

## Standards referenced

Stand allocation and gate assignment are provably NP-hard, which is why a solver is the right tool at scale. The design follows EU AI Act Article 14 (human oversight), EASA NPA 2025-07 Levels 1 and 2 (assistance and human-AI teaming), and the Microsoft reference architecture for contextual AI decision support (no auto-write, human review-and-accept).

## Disclaimer

This is an independent worked solution built for an interview case. It uses synthetic data throughout and is not affiliated with, endorsed by, or built on data from Copenhagen Airports A/S. Cost units and operational parameters are illustrative, not calibrated to CPH operations.

## License

MIT. See [LICENSE](LICENSE).
