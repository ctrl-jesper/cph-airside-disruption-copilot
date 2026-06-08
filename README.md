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

## The full-scale build

The `full-scale/` app is a single self-contained HTML page that extends the core with five capabilities, each governed by a tunable policy constant in a Config tab. Every one defaults to off or neutral, so the verified base solve is unchanged until it is enabled.

- **Tow-to-remote and fleet capacity.** A long-dwell aircraft deboards on its contact stand, is towed to a remote parking position for the parked middle of its dwell, then returns to board, which frees the contact stand for another aircraft. The tow is a binary in the objective with a tow cost, and tug and bus fleet limits raise a soft cost under saturation.
- **Uncertainty handling.** The conflict buffer can be sized from each flight's arrival-time uncertainty band, visualized as a fuzzy Gantt-bar edge. A Monte Carlo over a few hundred arrival-time draws reports how often the plan stays conflict-free and flags fragile stands.
- **Bottom-up value.** A live euro figure against the naive first-come-first-served rule an incumbent allocator would run: protected connections, bus trips avoided, contact-stand-hours gained, less tow cost, each at a configurable unit cost labelled an assumption.
- **Governance view.** An EU AI Act screen that classifies the tool honestly as limited-risk, names the autonomy threshold that would move it to high-risk, and states what binds regardless of tier.
- **Rolling horizon.** A manual re-optimization step that re-solves from the committed picture with a stability term, so the plan does not thrash when nothing material has changed.

## Governance and EU AI Act

The tool is most likely limited-risk under the EU AI Act, not Annex III high-risk. A read-only, advisory stand allocator is not a safety component of critical infrastructure, even though an airport is. The threshold to high-risk is autonomy: writing allocations back to the airport operational database, or cancelling and diverting flights without a person. Transparency duties (Article 50) apply to the conversational interface, and GPAI obligations sit with the model provider, not the deployer. The position is to classify honestly as limited-risk, apply Article 14 oversight voluntarily, and treat crossing the threshold as a governance decision. GDPR and Danish co-determination bind in practice regardless of tier.

## Running it

Everything is static and offline. Serve any sub-app with a local web server, for example:

```bash
python3 -m http.server 8124 --directory full-scale
# then open http://localhost:8124
```

The full-scale app vendors `glpk.js` locally, so it needs no network access. The deck is a self-contained HTML file at `deck/deck.html`.

## Repository layout

| Path | What it is |
|---|---|
| `full-scale/` | The production-vision app: whole-apron MIP, Pareto front, digital twin, tow, uncertainty, value, governance, rolling horizon |
| `poc/` | The proof-of-concept prototype, the shape of a Copilot Studio answer |
| `poc-v2/` | An earlier polished PoC iteration, kept for reference. See `poc-v2/README.md` |
| `deck/` | The presentation, as a self-contained HTML deck and a PDF |
| `copilot-studio/` | The Copilot Studio agent configuration, build guide, knowledge pack, and scenario |
| `Solution Overview and Assumptions.md` | The full technical reference: scope, architecture, real-versus-simulated, assumptions, limitations, standards |
| `v2_build_notes.md` | The development log: the full-scale build phase by phase, with the decisions and verification behind each |
| `synthetic_aodb_*.{csv,json}` | The synthetic AODB snapshot, flights, and stands |

## Standards referenced

Stand allocation and gate assignment are provably NP-hard, which is why a solver is the right tool at scale. The design follows EU AI Act Article 14 (human oversight), EASA NPA 2025-07 Levels 1 and 2 (assistance and human-AI teaming), and the Microsoft reference architecture for contextual AI decision support (no auto-write, human review-and-accept).

## Disclaimer

This is an independent worked solution built for an interview case. It uses synthetic data throughout and is not affiliated with, endorsed by, or built on data from Copenhagen Airports A/S. Cost units and operational parameters are illustrative, not calibrated to CPH operations.

## License

MIT. See [LICENSE](LICENSE).
