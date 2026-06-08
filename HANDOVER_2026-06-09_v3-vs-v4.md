# Session note 2026-06-09: v3 extraction and demo-alignment context

For the session working on v4. Complements `HANDOVER.md` (read that for the full case background). This note covers only what changed today and the open question on which build to demo.

## Git and folder state (read first)

- **v4 work is committed and safe.** The in-progress edit to `poc-v4/index.html` is in commit `3c714af` ("Unify U8: unified event log + remove dead V3 code", authored 2026-06-09 00:39). Working tree is clean. `poc-v4/` was not modified by the other session.
- **A runnable v3 was extracted** from the `poc-v3` git tag (commit `813da39`, "Add Value tab") into a new sibling folder `poc-v3/`: `index.html` (181 KB) plus its own `assets/`. This is the snapshot optimizer: no 48h feed, no continuous rolling-horizon optimizer, no injectable disruptions, no realized-cost ledger.
- **`poc-v3/` is currently untracked** (not committed). Decide whether to commit it.
- **launch.json updated:** root `.claude/launch.json` gained a `cph-pocv3` config on port 8125. A duplicate entry was also added to the case-subfolder `.claude/launch.json`; harmless, the preview tool reads the root one.

## Port and folder map

- v3 snapshot: `poc-v3/` to port 8125 (`cph-pocv3`)
- v4 continuous: `poc-v4/` to port 8124 (`cph-pocv4`)
- Both run. v3 boots clean: all assets 200, GLPK WASM worker spawns, no console errors.

## Why v3 was extracted

We are weighing whether the lighter v3 snapshot fits the actual constraints better than v4. v3 now runs alongside v4 so both can be compared without disturbing either. No build direction has been decided.

## Demo-alignment findings that may affect v4 work

1. **10-minute hard cap on the whole presentation.** A full v4 click-through is roughly 9 minutes by itself, leaving no room for the problem, solution, governance, and value narrative. Whatever the build, the live demo must collapse to one ~3-minute thread: inject one disruption, re-optimize, twin reveals a second-order effect, pick a trade-off, show the value gap. This is the largest risk and argues against adding demo surface.
2. **"Is this AI?" tension.** The genuinely-computed core (MIP solver plus discrete-event twin) is operations research, not ML or generative AI. The ML timing layer is synthetic hand-set data. The conversational LLM interface is not wired into the app. The fix is a reframe (layered AI: ML prediction, optimization core, Copilot Studio conversational layer, deterministic on purpose for critical infrastructure), not more building. The gap is narrative, not capability.
3. **Brief-named gaps in the model.** Crew is named in the case as a key dependency and is not modelled (handlers are the proxy). Baggage is shown as a consequence, not optimized. Both are acceptable as stated boundaries; flagging so v4 work does not assume they are covered.
4. **Microsoft-First mismatch.** The case stresses M365 Copilot, Copilot Studio, Power Platform, Azure, and the Airhart AODB. The app is a standalone WASM page off that stack. The `copilot-studio/` agent config is the artifact that bridges this. Keep that bridge intact.
5. **Redirect/cancel decision** must be visibly triggered on screen (an escalation), or one of the brief's three required decisions goes unshown.

## Net

Nothing destructive happened. `poc-v4` is exactly as left. The open question is v3-as-demo versus v4-as-demo, driven mainly by the 10-minute cap and the "where is the AI" framing.
