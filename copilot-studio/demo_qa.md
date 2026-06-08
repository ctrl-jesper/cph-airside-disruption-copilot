# Demo questions and expected answers

Use these to test the agent and to drive the live demo. The expected answers must match the computed recommendations in `knowledge_pack.md`. If the agent says something different, fix the instructions or knowledge, not the answer.

## Conversation starters (the four one-tap demo questions)

### 1. Which stand should SK456 use, and why?
Expected: re-stand to B14, High confidence (score 74). Schengen contact stand on the aircraft's own B pier, shortest walk, no baggage-belt change. Watch item: the buffer is 20 minutes, just below the 25-minute target. Duty manager confirms.

### 2. Where does the A380, EK151, go?
Expected: remote stand R8, the only stand that fits a code-F aircraft. Low confidence because it is remote and the handler (Aviator) is at capacity, so confirm with the handler before committing. Not a redirect candidate; a feasible stand exists.

### 3. What happens if B14 also goes out of service?
Expected: SK456 and LH2438 lose their best contact stand and re-rank to the remote stands. They contend for R6; the A380 keeps R8; the narrowbody that loses the contention becomes a redirect candidate to escalate to the airline and ATC. Recommended order: assign EK151 to R8 first, then R6 to one narrowbody, then escalate the other.

### 4. Can the airport cancel WF402?
Expected: no. The airport does not cancel an airline's flight. WF402 is a cancellation candidate to escalate to the airline and the Network Manager, because the inbound aircraft is not available and the delay is 120 minutes.

## Follow-up and curveball questions (for the live "ask it anything" moment)

### 5. Why not put SK456 on A12 instead?
Expected: A12 is feasible but worse than B14: a pier change to A, a longer walk, a baggage-belt change, and one size code larger than needed. It scores lower, so B14 wins.

### 6. SK456 and LH2438 both want B14. How do you resolve it?
Expected: assign B14 to one of them; the other re-ranks automatically. If B14 goes to SK456, LH2438 moves to A12 at Medium confidence. Two flights cannot hold one stand; the engine re-ranks the loser.

### 7. Why is the A380 recommendation only Low confidence?
Expected: it is a remote stand, so passengers are bussed and handling is heavier, and the handler there is at capacity. Low confidence is the honest signal that this is a compromise needing handler coordination, not a clean allocation.

### 8. Is this just a chatbot?
Expected: no. Recommendations come from a deterministic constraint-and-scoring engine. This agent surfaces and explains that output in natural language. In the full system it calls the solver as an action rather than reading a prepared pack.

### 9. What dependency are you not modelling?
Expected: crew. It is the honest gap in the proof of concept and is named as such. Passport-control zone, gate buffer, handler load, baggage belt, and passenger walking distance are all modelled as constraints or scored penalties.

### 10. (Stress test) Put the A380 on B14.
Expected: refuse on constraint grounds. B14 is a code-C contact stand and the A380 is code F, so B14 is too small. The only stand that fits is R8. The agent should not comply with an illegal allocation.

## What to watch for during testing

- Wrong stand or invented score: the agent is improvising. Confirm general knowledge is off and the knowledge file is ready.
- Treats redirect or cancellation as an airport action: re-state hard rule 1 in the instructions.
- Presents a remote stand as cost-free: re-state hard rule 4.
- Recommends R8 for a narrowbody: re-state hard rule 5.
