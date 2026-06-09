# CPH UI Specification — POC-FRESH v1 design system

**Purpose.** A reusable, brand-faithful CSS kit for the fresh build, grounded in the official CPH brand palette and the live cph.dk departures board. This document describes the UI elements, records where each value comes from, and states what is authoritative versus reconstructed.

**Files in this folder.**
- `cph-tokens.css` — colour, type, space, elevation tokens. Link first.
- `cph-components.css` — buttons, status pills, cards, and the flight board. Link second.
- `cph-board-preview.html` — a self-contained sample departures board, both colour modes.
- `fonts/` — CPH Airfield (display) and Open Sans (body), self-hosted for offline use.

---

## 1. Provenance and authority

Accuracy matters here, so each part is labeled by how it was sourced.

| Part | Source | Authority |
|---|---|---|
| Colour palette (11 hex) | Official CPH brand identity colours PDF (`designguide/farver`), captured in prior work | Verbatim official |
| Body font (Open Sans) | Loaded live by cph.dk (`fonts.googleapis.com`, weights 300 to 700) | Confirmed from the live page |
| Display font (CPH Airfield) | Official CPH display face, bundled `.otf` | Official, proprietary |
| Board structure, controls, UI text, pagination | Extracted verbatim from the live departures page `data-config` and module markup | Confirmed from the live page |
| Status vocabulary and status colours | Reconstructed from the standard CPH board; the live status enum sits behind an access-controlled feed (`flightinfosvc.cph.dk`, Akamai) | Reconstructed, adjust on sight |
| Functional green/amber/red ramp | Designer extension, not in the brand kit | Extension, deliberate |

The one item I could not extract verbatim is the live status enum and the board's compiled CSS, because the board is hydrated client-side from a feed that returns 403 to non-browser clients. Everything visual below is either official, confirmed from the page config, or clearly flagged as reconstructed.

---

## 2. Colour

### 2.1 Official brand palette (verbatim)

| Token | Name | Hex | Role in UI |
|---|---|---|---|
| `--cph-midnight-blue` | Midnight Blue | `#060E4D` | Primary. Logo, headings, body ink |
| `--cph-evening-blue` | Evening Blue | `#170D81` | Deep accent, critical fills |
| `--cph-twilight-blue` | Twilight Blue | `#4E55AB` | Interactive accent, links, primary button |
| `--cph-sky-blue` | Sky Blue | `#88A8ED` | Light accent, borders on blue surfaces |
| `--cph-horizon-blue` | Horizon Blue | `#DCE7F7` | Calm surface, positive and info background |
| `--cph-sunrise-yellow` | Sunrise Yellow | `#FFDD66` | Primary accent, attention, the signature highlight |
| `--cph-sunshine-yellow` | Sunshine Yellow | `#FEF9D8` | Pale yellow surface |
| `--cph-space-black` | Space Black | `#171717` | Near-black ink |
| `--cph-clear-sky-white` | Clear Sky White | `#FCFBFA` | Off-white page canvas |
| `--cph-misty-grey` | Misty Grey | `#E6E2DC` | Warm light grey, lines and wells |
| `--cph-storm-grey` | Storm Grey | `#C4BFBB` | Warm mid grey, muted and departed states |

The brand guide names Midnight Blue and Sunrise Yellow as the two primary colours, the blues and Sunshine as secondary, and the greys and whites as base.

### 2.2 The no-red, no-green problem (a real decision for the fresh build)

The official palette contains no functional red and no functional green. On the public passenger board this is fine; CPH renders a calm, blue-led board where "on time" is pale Horizon Blue, attention is Sunrise Yellow, and disruption is deep Evening Blue. For a passenger glancing at a sign, that reads well.

The fresh build is an internal operations decision-support tool, where missing a cancellation or a critical conflict is costly, and operators expect a conventional traffic-light read. Relying only on blue and yellow shades removes that instant green/amber/red legibility, and it leans on hue distinctions that are weaker for colour-vision deficiency.

The kit therefore ships two status systems:

- **Brand-pure (default).** Horizon Blue positive, Sunrise Yellow attention, Evening Blue critical. Faithful to cph.dk. Use for passenger-facing surfaces and for the brand moment in the demo.
- **Functional (opt-in).** An accessible green/amber/red ramp, tuned deep so it still sits with the brand. Switch on by adding the class `is-functional` to any ancestor of the status pills. Use for the operational board where a fast, unambiguous read matters.

Recommendation for the POC: brand-pure for the headline and any passenger-facing framing, functional for the live operational board. State the choice in the demo as a deliberate, accessibility-driven extension of the brand, which is exactly the judgement an ops tool requires.

---

## 3. Typography

- **Display and headings:** CPH Airfield. The CPH headline face. Use for `h1`/`h2`, the board headline, and large numerics where brand character helps. Proprietary; bundled here for the user's own demo, not for redistribution.
- **Body, UI, and data:** Open Sans (weights 400, 500, 600, 700). This is what cph.dk loads live. Use for all running text, controls, and table data.
- **Numerics (times, flight numbers, gates):** Open Sans with tabular figures (`font-variant-numeric: tabular-nums`, exposed as `.cph-num` and `--fvn-tabular`), so columns of times align.

Type scale, weights, and tracking are tokenised in `cph-tokens.css` section 4. Headings track slightly tight; an overline style provides the uppercase, wide-tracked labels used on table headers.

---

## 4. The departures board, anatomy

Reconstructed faithfully from the live module. Text in quotes is verbatim from the page config.

### 4.1 Structure, top to bottom

1. **Tabs.** "Arrivals" and "Departures", the selected tab underlined in Sunrise Yellow.
2. **Headline.** "Find your departure" (arrivals: "Find your arrival"), in CPH Airfield.
3. **Controls row.**
   - Search field, placeholder "Search for city, airline or flight number...".
   - Date picker, with a "Time" label.
4. **Meta row.** A "Last updated HH:MM" stamp, and a results line "There are {amount} departures for your search".
5. **Flight table.** Columns below.
6. **Pagination.** "Show more" / "Show less", and "Show previous flights" / "Hide previous flights".

### 4.2 Behaviour constants (verbatim from config)

- Initial rows: 10 on desktop, 15 on mobile. Increment: 10 / 15. Previous flights: 10.
- Feed error time limit: 30 s. No-results header: "There is currently no information about the chosen date or departure".

### 4.3 Columns

The standard CPH departures column set, in order:

| Column | Content | Cell treatment |
|---|---|---|
| Time | Scheduled departure, revised time if delayed | `.cph-cell-time`, bold tabular; revised time struck through before the new time |
| Destination | City, with airline as secondary line | `.cph-cell-dest` |
| Flight | Flight number (IATA) | `.cph-cell-flight`, tabular |
| Terminal | Terminal number | `.cph-cell-term`, tabular |
| Gate | Gate, emphasised | `.cph-cell-gate`, semibold |
| Status | Status pill | `.cph-status--*` |

On narrow screens the Flight and Terminal columns collapse (see the media query), keeping Time, Destination, Gate, and Status.

---

## 5. Status vocabulary and colour mapping

Reconstructed from the standard CPH departures board. Treat the label text as adjustable once the live enum is confirmed; the colour roles are the durable part.

| Status label | Meaning | Brand-pure | Functional |
|---|---|---|---|
| On time | Departing as scheduled | `--ontime` Horizon Blue | green |
| Go to gate | Gate assigned, proceed | `--ontime` Horizon Blue | green |
| Boarding | Boarding in progress | `--attention` Sunrise Yellow | amber |
| Final call | Last call to gate | `--attention` Sunrise Yellow | amber |
| Gate closing | Doors about to close | `--attention` Sunrise Yellow | amber |
| New time / Delayed | Revised departure time | `--attention` Sunrise Yellow | amber |
| Gate closed | Boarding ended | `--muted` Storm Grey | neutral grey |
| Departed | Off-block | `--muted` Storm Grey | neutral grey |
| Cancelled | Flight cancelled | `--critical` Evening Blue | red |

Markup pattern:

```html
<span class="cph-status cph-status--attention">Boarding</span>
```

Switch the whole board to functional colour by adding `is-functional` on a wrapper:

```html
<div class="cph is-functional"> ... board ... </div>
```

---

## 6. How to use in the fresh build

1. Copy this `design-system/` folder (with `fonts/`) into the POC.
2. In the page head, link the two CSS files in order:
   ```html
   <link rel="stylesheet" href="design-system/cph-tokens.css">
   <link rel="stylesheet" href="design-system/cph-components.css">
   ```
3. Wrap the app in `<div class="cph">` so the base type and colour apply.
4. Build the board from the markup in `cph-board-preview.html`.
5. Reference semantic tokens (`var(--c-brand)`, `var(--c-accent)`), never raw hex, so a later brand change is one edit.

The operational POC (the engine, the Pareto view, the calibration panel) should reuse the same tokens, status pills, buttons, and cards, so the decision-support UI and the flight board read as one product.

---

## 7. Fidelity notes and open items

- **Verbatim:** the 11 brand colours, the fonts, and the board's structure, control text, and pagination constants.
- **Reconstructed:** the status label set and the status-to-colour mapping. The live enum sits behind an access-controlled feed; confirm the exact labels by eye against the live board and adjust the pill text. The colour roles will not change.
- **Not replicated:** the live site's exact compiled CSS (spacing, shadows, hover micro-interactions). Those are hydrated from a JS bundle and are not necessary to reuse; this kit reproduces the look from the brand tokens rather than copying the site's stylesheet.
- **Licensing:** CPH Airfield is proprietary, bundled for your own interview demo only. Open Sans is OFL. Do not redistribute the kit publicly with the CPH font embedded.
