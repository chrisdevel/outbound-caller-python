# CLAUDE.md

## WHAT — Stack & Project Structure

This is a single-file Python project using the **LiveKit Agents Framework** to build an AI-powered outbound call agent.

```
outbound-caller-python/
├── agent.py          # Entire application — agent logic, entrypoint, CLI runner
├── requirements.txt  # Python dependencies
├── taskfile.yaml     # Task runner (install, dev tasks)
├── .env.example      # Required environment variable template
├── .env.local        # Local secrets (gitignored)
└── venv/             # Python virtual environment (gitignored)
```

**Key dependencies** (`requirements.txt`):
- `livekit` — LiveKit real-time SDK
- `livekit-agents[openai,deepgram,cartesia,silero,turn_detector]` — Agents framework with all plugins
- `livekit-plugins-noise-cancellation` — Krisp BVC telephony noise cancellation
- `python-dotenv` — loads `.env.local`

**AI pipeline** (inside `agent.py`):
- **STT**: Deepgram
- **LLM**: OpenAI GPT-4o
- **TTS**: Cartesia
- **VAD**: Silero
- **Turn detection**: `EnglishModel` from `livekit.plugins.turn_detector`
- **Noise cancellation**: Krisp `BVCTelephony`

---

## WHY — Purpose & Functionality

This agent makes **AI-powered outbound phone calls** via SIP trunking. The use case is a dental practice scheduling assistant that:

1. Dials a patient's phone number via a configured SIP outbound trunk
2. Confirms or reschedules their appointment via voice conversation
3. Handles voicemail detection (hangs up automatically)
4. Supports transferring to a human agent on request
5. Handles clean call termination when the user wants to end the call

**`OutboundCaller` agent tools** (function tools the LLM can call):
- `transfer_call` — transfers the SIP call to a human agent
- `end_call` — gracefully hangs up
- `look_up_availability` — returns available appointment slots for a given date
- `confirm_appointment` — confirms an appointment date/time
- `detected_answering_machine` — detected voicemail, hangs up

**Call flow** (`entrypoint` function):
1. Connect to LiveKit room
2. Parse `dial_info` from job metadata (phone number + transfer target)
3. Start `AgentSession` with the full AI pipeline
4. Dial the patient via `create_sip_participant` (blocks until answered)
5. Agent begins conversation; tools fire as needed

---
## REQUIREMENTS — Outbound Caller Specification

### Functional Requirements

1. The agent must initiate outbound phone calls via a configured SIP outbound trunk.
2. The phone number to dial must be passed via LiveKit job metadata (`phone_number`).
3. The agent must block until the call is answered before starting conversation.
4. The agent must support:
   - Transferring calls to a human (`transfer_call`)
   - Ending calls cleanly (`end_call`)
   - Handling voicemail detection (`detected_answering_machine`)
5. The agent must terminate gracefully if the user requests call termination.
6. The agent must log call lifecycle events.

---

### AI Pipeline Requirements

1. Speech-to-Text must use Deepgram.
2. LLM must use OpenAI GPT-4o.
3. Text-to-Speech must use Cartesia.
4. Voice activity detection must use Silero.
5. Turn detection must use EnglishModel.
6. Telephony noise cancellation must use Krisp BVC.

---

### Telephony Requirements

1. A valid `SIP_OUTBOUND_TRUNK_ID` must be configured.
2. Calls must use TLS transport (port 5061).
3. Phone numbers must be in E.164 format.
4. The SIP trunk must authenticate via username/password.
5. Call must fail gracefully if SIP connection fails.

---

### Operational Requirements

1. The agent must run as a LiveKit Worker.
2. The agent must auto-register as `outbound-caller`.
3. Dispatch jobs must pass metadata as JSON.
4. The system must support local dev mode (`python3 agent.py dev`).
5. The worker must reconnect on transient LiveKit disconnections.

---

### Compliance & Safety Requirements

1. No persistent storage of call audio.
2. No persistent storage of transcripts unless explicitly configured.
3. The agent must not provide medical or financial advice beyond scheduling scope.
4. The agent must respect call termination requests immediately.

## HOW — Working on This Project

### Setup

```shell
python3 -m venv venv
source venv/bin/activate        # macOS/Linux
pip install -r requirements.txt
python3 agent.py download-files  # downloads VAD/turn-detection model files
```

Copy `.env.example` to `.env.local` and fill in all values:

```
LIVEKIT_URL=
LIVEKIT_API_KEY=
LIVEKIT_API_SECRET=
OPENAI_API_KEY=
SIP_OUTBOUND_TRUNK_ID=
DEEPGRAM_API_KEY=
CARTESIA_API_KEY=
```

### Running the agent

```shell
python3 agent.py dev
```

The worker connects to LiveKit and waits for dispatch jobs.

### Triggering a call (requires `lk` CLI)

```shell
lk dispatch create \
  --new-room \
  --agent-name outbound-caller \
  --metadata '{"phone_number": "+1234567890", "transfer_to": "+9876543210"}'
```

### Task runner (optional)

[Task](https://taskfile.dev/) is configured via `taskfile.yaml`:

```shell
task install   # set up venv + install deps + download files
task dev       # activate venv + run agent in dev mode
```

### Verifying changes

There are no automated tests. To verify changes:
1. Run `python3 agent.py dev` — confirms the agent starts without import/syntax errors
2. Trigger a real or sandbox call dispatch via the `lk` CLI
3. Check logs — logger is `outbound-caller` at `INFO` level

### Important notes

- All application code lives in `agent.py` — there are no other modules
- `dial_info` is passed as JSON in the LiveKit job's `metadata` field
- The agent name registered with LiveKit is `"outbound-caller"` (set in `WorkerOptions`)
- A pre-configured SIP outbound trunk (`SIP_OUTBOUND_TRUNK_ID`) is required before any calls can be made
- Use `python3`, not `node`/`bun` — this is a pure Python project

**importnatn**: when you work on a new feature or bug, create a git branch first. then work on changes in that branch for the ramainder of the session. 