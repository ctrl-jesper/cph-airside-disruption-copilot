# Copilot Studio build guide: grounded-knowledge agent

Goal: stand up the Airside Disruption Copilot in Copilot Studio, grounded in `knowledge_pack.md`, and publish it so it opens on your iPhone over the hotspot. Estimated time once you have access: 30 to 45 minutes.

## Step 0: Access check (2 minutes)

1. Open `https://copilotstudio.microsoft.com` and sign in with your Microsoft work or school account.
2. If you land on the Copilot Studio home with a "Create" or "New agent" option, you have access. Skip to Step 1.
3. If you are blocked, you have three fallbacks, in order of preference:
   - Start the Copilot Studio free trial when prompted (needs the tenant to allow trials).
   - Ask whoever owns your Microsoft tenant to enable Copilot Studio or grant a trial.
   - If neither works before Friday, the local web app is the primary demo and this agent becomes a "here is the same thing on the CPH Microsoft stack" talking point with screenshots. Tell me and I will adjust the script.

Note on a personal Gmail account: Copilot Studio needs a Microsoft 365 / Entra ID work account, not a personal consumer account. If your only login is the Gmail address, use the trial path or a Microsoft work account.

## Step 1: Prepare the knowledge file

Copilot Studio knowledge upload accepts PDF, Word, and similar. It does not take Markdown directly.

- Convert `knowledge_pack.md` to PDF or Word first. Ask me to generate `knowledge_pack.pdf` and I will produce it with the project PDF exporter, or paste the Markdown into Word and save as `.docx`.
- Keep the file name clear, for example `CPH_Airside_Scenario.pdf`.

## Step 2: Create the agent

1. In Copilot Studio, choose Create, then New agent.
2. If a conversational setup assistant appears, skip it (look for "Skip to configure") so you go straight to the configuration screen.
3. Name: `Airside Disruption Copilot`.
4. Description: `Real-time stand-allocation decision support for CPH Airside Operations. Advisory only, human-in-the-loop.`
5. Instructions: paste the full contents of `agent_instructions.md`.
6. Save.

## Step 3: Add the knowledge

1. Open the Knowledge area of the agent.
2. Add knowledge, choose file upload, and upload `CPH_Airside_Scenario.pdf` (the converted knowledge pack).
3. Wait until the source status shows ready.
4. Recommended: turn off general world knowledge / "use general knowledge" if the toggle exists, so the agent answers from your file and does not improvise from the open model. Keep it grounded.

## Step 4: Conversation starters

Add these as suggested prompts (or starter topics) so the demo has one-tap questions. They are also listed in `demo_qa.md`.

- Which stand should SK1425 use, and why?
- Where does the A380, EK0151, go?
- What happens if B16 also goes out of service?
- Can the airport cancel WF402?

## Step 5: Test

1. Open the Test pane.
2. Run every question in `demo_qa.md` and confirm the answers match the computed recommendations in the knowledge pack.
3. If an answer drifts (wrong stand, invented score, or treats a redirect as an airport action), tighten the Instructions and re-test. The usual fixes are: re-state the hard rule it broke, and confirm general knowledge is off.

## Step 6: Publish for the phone demo

1. Publish the agent.
2. Open Channels (or Settings, then the published demo link). Use the "Demo website" or "Custom website" link, or the Test link if your tenant exposes it.
3. Copy the published URL, open it on your iPhone over the hotspot, and run two questions to confirm it works on the phone before Friday.

## Step 7: Break-glass fallback

Cloud demos fail at the worst moment. Before Friday, screen-record a clean run of the four conversation starters on the phone and save it with the local-app recording. If the live agent will not load on the day, you play the recording and keep talking. This is task 6 in the project plan.

## Positioning note for the interview

Say this plainly so the grounded-knowledge design is a strength, not a soft spot: the recommendations come from the deterministic engine shown in the local app; this Copilot Studio agent is the conversational interface layer on the CPH Microsoft stack, making those recommendations queryable in natural language. In the full system the agent calls the solver as an action rather than reading a prepared pack, which is the action-backed version. That sentence turns "is this just a chatbot" into "this is the interface layer of a hybrid architecture, and here is exactly how it connects to the engine."
