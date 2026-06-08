# Airside Disruption Copilot: Scenario Knowledge Pack

This document is the single knowledge source for the Airside Disruption Copilot agent. The agent answers questions about one disruption scenario at Copenhagen Airport (CPH) using only the facts and rules written here. Recommendations are produced by a deterministic engine (a hard-constraint filter followed by a weighted score). This document records the inputs, the rules, and the engine's computed output for the base scenario and the hardened scenario so the agent can speak those results accurately.

Simulated time: 14:20 local, a busy weekday afternoon.

---

## 1. The scenario

It is a high-traffic afternoon. Several inbound flights are already delayed because of early-turnaround pressure, European airspace capacity restrictions, and a ground handler running at its limit. A central contact stand has gone out of service after a technical fault. Multiple aircraft are arriving outside their planned slots. Some departures are slipping far enough to threaten the next turnaround on their stand. One ground handler, Aviator, cannot take on more turnarounds.

Airside Operations must decide, in real time, which arriving aircraft go to which stands, and which flights have no feasible stand and must be escalated to the airline and ATC or the Network Manager as redirect or cancellation candidates. Every move ripples into passenger flow, gates, passport control (Schengen and non-Schengen), baggage, crew, and handling.

### Hardened scenario (the "what if it gets worse" injection)

The agent should also be able to reason about a degraded variant where two more problems appear:

- Stand B14 develops a jet bridge fault, so it can no longer be used as a contact stand.
- Stand A12 becomes occupied until 15:40, so it is not available in time for the early-afternoon arrivals.

In the hardened scenario, any earlier assignment that used B14 or A12 is no longer valid and must be re-decided. The aircraft must re-rank against whatever stands remain, which are mostly remote.

---

## 2. Hard constraints (a stand is illegal for a flight if any of these is true)

1. The stand is out of service.
2. The stand is too small: its maximum aircraft size code is smaller than the aircraft's code. Size codes run A to F by wingspan. An A320 is code C, an A319 is code C, a B737 is code C, a B777 is code E, an A380 is code F. A stand's maximum code must be equal to or larger than the aircraft's.
3. Zone mismatch on a contact stand: a Schengen arrival needs a Schengen contact stand and a non-Schengen arrival needs a non-Schengen contact stand. Remote stands are mixed zone because passengers are bussed and routed to the correct control.
4. Jet bridge fault on a contact stand.
5. The stand is occupied past the time the aircraft needs it (the stand frees later than the aircraft's on-block time).
6. The next scheduled arrival on that stand is too soon: it lands before this aircraft would clear the stand (on-block time plus ground time).

A flight with no legal stand is a redirect candidate (for an arrival) and is escalated. The airport does not redirect or cancel; it recommends.

---

## 3. Soft scoring (rank the legal stands; higher is better)

Every legal stand starts at 100 points. The engine subtracts penalties:

- Remote stand: minus 25.
- Pier change from the aircraft's planned pier: minus 15.
- Oversized stand: minus 5 for each size code the stand is larger than the aircraft needs (for example, a code-C aircraft on a code-E stand loses 10). This keeps scarce large stands free for large aircraft.
- Walking distance: minus 1.5 for each minute of walk from the stand to the central point.
- Baggage belt change from the planned belt: minus 8.
- Remote zone handling and bussing: minus 10.
- Ground handler already at capacity: minus 20.
- Buffer below 10 minutes: minus 40. Buffer between 10 and 25 minutes: minus 20. Buffer is the gap between this aircraft clearing the stand and the next aircraft needing it; the target is 25 minutes or more.

### Confidence

- High: a contact stand scoring 70 or more.
- Medium: any stand scoring 45 or more.
- Low: anything below 45, including remote stands that score well, because a remote stand always carries operational compromise.

---

## 4. Stand inventory

| Stand | Pier | Zone | Type | Max code | Jet bridge | Status | Free from | Next arrival | Walk (min) | Belt | Handler |
|---|---|---|---|---|---|---|---|---|---|---|---|
| B14 | B | Schengen | contact | C | ok (faulted in hardened) | free | 14:20 | 15:30 | 4 | 5 | Menzies (ok) |
| A12 | A | Schengen | contact | D | ok | free (occupied to 15:40 in hardened) | 14:20 | 16:30 | 9 | 3 | Menzies (ok) |
| R6 | Remote | mixed | remote | E | none | free | 14:20 | 18:00 | 12 | 9 | Aviator (at capacity) |
| R8 | Remote | mixed | remote | F | none | free | 14:20 | 18:30 | 12 | 9 | Aviator (at capacity) |
| C30 | C | non-Schengen | contact | E | ok | occupied to 16:00 | 16:00 | 17:00 | 9 | 7 | Menzies (ok) |
| C32 | C | non-Schengen | contact | E | ok | out of service | n/a | n/a | 10 | 8 | Menzies (ok) |
| B10 | B | Schengen | contact | C | fault | free | 14:20 | 16:00 | 5 | 5 | Menzies (ok) |
| D4 | D | Schengen | contact | C | ok | out of service | n/a | n/a | 6 | 4 | Menzies (ok) |
| E2 | B | Schengen | contact | C | ok | occupied to 15:20 | 15:20 | 16:30 | 4 | 5 | Menzies (ok) |
| A30 | A | Schengen | contact | C | ok | occupied to 14:50 | 14:50 | 16:00 | 8 | 2 | Aviator (at capacity) |

R8 is the only stand that can take a code-F aircraft (the A380). R6 is the largest remote that takes up to code E.

---

## 5. Arrivals

| Flight | Airline | Type | Code | Zone | From | Scheduled | Estimated | On-block | Ground (min) | Off-block | Planned stand | Slot | Needs decision |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| SK456 | SAS | A320 | C | Schengen | OSL | 14:10 | 14:25 | 14:25 | 45 | 15:10 | B16 | on | yes, B16 is out of service |
| EK151 | Emirates | A388 | F | non-Schengen | DXB | 13:15 | 14:55 | 14:55 | 90 | 16:25 | none | off | yes, off-slot, no code-F contact stand |
| LH2438 | Lufthansa | A319 | C | Schengen | FRA | 14:05 | 14:40 | 14:40 | 40 | 15:20 | B20 | off | yes, off-slot, planned stand unavailable |
| SK1402 | SAS | A320 | C | Schengen | ARN | 14:00 | 14:30 | 14:30 | 45 | 15:15 | A30 | off | no, on plan |
| EW9011 | Eurowings | A320 | C | Schengen | DUS | 14:15 | 14:15 | 14:15 | 40 | 14:55 | B22 | on | no, on plan |
| AY911 | Finnair | A320 | C | Schengen | HEL | 14:20 | 14:20 | 14:20 | 40 | 15:00 | B18 | on | no, on plan |
| KL1129 | KLM | E190 | C | Schengen | AMS | 14:35 | 14:35 | 14:35 | 35 | 15:10 | D8 | on | no, on plan |

## 6. Departures

| Flight | Airline | Type | To | Scheduled | Estimated | Delay (min) | Stand | Inbound aircraft | Knock-on | Needs decision |
|---|---|---|---|---|---|---|---|---|---|---|
| DY1741 | Norwegian | B738 | BGO | 14:35 | 15:15 | 40 | B12 | available | yes | yes, hold and monitor |
| WF402 | Wideroe | DH8D | AAL | 14:25 | 16:25 | 120 | A8 | not available | yes | yes, cancellation candidate |
| SK1209 | SAS | A320 | LHR | 15:00 | 15:00 | 0 | C5 | available | no | no, on schedule |
| LH837 | Lufthansa | A320 | FRA | 15:05 | 15:05 | 0 | D7 | available | no | no, on schedule |

---

## 7. Computed recommendations: base scenario

### SK456 (A320, code C, Schengen)
- Recommendation: re-stand to B14. Confidence High (score 74).
- Why: B14 is a Schengen contact stand on the aircraft's own B pier, with the shortest walk and no baggage-belt change. The only watch item is the buffer, 20 minutes, just below the 25-minute target.
- Second option: none of the other contact stands are free and legal; the next ranked options are remote.
- Excluded stands include A12 and others that are larger, occupied, faulted, or out of service.

### EK151 (A388, code F, non-Schengen)
- Recommendation: remote stand R8, review with the handler. Confidence Low (score 27).
- Why: R8 is the only stand that fits a code-F aircraft. It is remote, so passengers are bussed and handling is heavier, and the handler at R8, Aviator, is already at capacity. This is a genuine compromise, which is why confidence is Low. It is still the correct call because no contact stand can take an A380, and a remote A380 stand at CPH is an established operation.
- This is not a redirect candidate. A feasible stand exists; the duty manager should confirm and coordinate the handler.

### LH2438 (A319, code C, Schengen)
- Recommendation: re-stand to B14. Confidence Medium (score 66).
- Note: SK456 also ranks B14 first. Two flights cannot hold the same stand. If B14 is assigned to SK456, LH2438 re-ranks to A12 (Medium, score about 59), a code-D Schengen contact stand on A pier with a longer walk and a baggage-belt change.
- This contention between SK456 and LH2438 for B14 is the clearest example of interdependency in the base scenario.

### DY1741 (B738, departure to BGO)
- Recommendation: hold and monitor. The 40-minute delay creates a knock-on risk to the next turnaround on stand B12, but no reallocation is needed yet.

### WF402 (DH8D, departure to AAL)
- Recommendation: cancellation candidate, escalate to the airline and the Network Manager. The inbound aircraft is not available and the delay is 120 minutes. No stand reallocation at CPH resolves this. The airport does not cancel; it recommends and requests a passenger re-accommodation plan.

---

## 8. Computed recommendations: hardened scenario

When B14 faults and A12 is occupied until 15:40, both lose their validity as contact stands for the early arrivals.

- SK456 re-opens. Its contact options are gone. It re-ranks to remote R6 (the right-sized remote for a code-C aircraft), Confidence Low.
- LH2438 re-opens. It also ranks R6 first, so SK456 and LH2438 now contend for R6.
- EK151 still needs R8, the only code-F stand. R8 must be protected for the A380. A narrowbody never ranks R8 above R6, because the oversized-stand penalty makes a code-C aircraft prefer the smaller remote.
- Once R6 is assigned to one narrowbody and R8 is held for EK151, the other narrowbody has no remaining stand and becomes a redirect candidate, escalated to the airline and ATC.

Recommended order of action in the hardened scenario: assign EK151 to R8 first (it has only one feasible stand), then assign R6 to one narrowbody, then escalate the remaining narrowbody as a redirect candidate. Handling this in the wrong order can strand the A380, which is exactly the interdependency the duty manager must manage.

---

## 9. Escalation and governance

- The airport never cancels or diverts an airline's flight. The copilot detects when no feasible stand exists, quantifies the knock-on, and escalates the trade-off to the airline and ATC (redirect) or the airline and the Network Manager (cancellation).
- The copilot is decision support under human oversight. It explains and ranks; it never allocates, and it never writes back to the airport operational database. The duty manager confirms every action.
- Recommendations are computed by the deterministic engine, not generated freely. Hard constraints filter illegal stands, then a transparent weighted score ranks the rest. The score breakdown is always available, so a recommendation can be inspected, not just trusted.

---

## 10. Dependency map

The recommendation surfaces first-order consequences across the dependencies named in the case:

- Passport control: the Schengen and non-Schengen zone rule is a hard constraint.
- Gates and buffer: stand availability and the buffer to the next arrival are modelled directly.
- Handling load: a handler at capacity is a scored penalty and a visible flag.
- Baggage: a belt change from the planned belt is a scored penalty and a visible flag.
- Passenger flow: walking distance is a scored penalty, used as a proxy for connection impact.
- Crew: not modelled in this proof of concept. This is the honest gap, and it is named as such.

The cascading, second-order ripple across the whole apron over time is the job of the full-scale solver and the digital twin, not this proof of concept.

---

## 11. Question and answer reference

These pairs help the agent answer common questions consistently.

- Q: Which stand should SK456 use? A: B14, a High-confidence re-stand. Short walk, same pier, no belt change. Watch the 20-minute buffer.
- Q: Why not put SK456 on A12? A: A12 is feasible but worse: it is a pier change to A, a longer walk, a baggage-belt change, and it is one size code larger than needed. It scores lower than B14.
- Q: Where does the A380, EK151, go? A: Remote stand R8, the only stand that fits a code-F aircraft. Confidence is Low because it is remote and the handler is at capacity, so confirm with the handler before committing. It is not a redirect candidate.
- Q: Can the airport just cancel WF402? A: No. The airport does not cancel an airline's flight. WF402 is a cancellation candidate to escalate to the airline and the Network Manager, because the inbound aircraft is not available and the delay is 120 minutes.
- Q: What happens if B14 also goes out of service? A: SK456 and LH2438 lose their best contact stand and re-rank to the remote stands. They contend for R6, the A380 keeps R8, and the narrowbody that loses the contention becomes a redirect candidate to escalate.
- Q: Is this just a chatbot? A: No. The recommendations are produced by a deterministic constraint-and-scoring engine. This conversational agent surfaces and explains the engine's output in natural language. In the full system the agent calls the solver as an action rather than reading pre-computed results.
