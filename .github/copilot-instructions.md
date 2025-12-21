# Jarvis Ecosystem Copilot Instructions

## 1. Architecture & "Big Picture"
The system is a **Personal AI Ecosystem** composed of 4 distinct microservices. 
**Crucial Rule**: All AI/LLM logic resides EXCLUSIVELY in `jarvis-intelligence-service`. Other services are "dumb" pipes or interfaces.

### Service Boundaries
- **`jarvis-intelligence-service` (The Brain)**: 
  - **Responsibility**: LLM processing, decision making, parsing user intent, database writes for analyzed data.
  - **Tech**: Python, FastAPI (implied), Anthropic API.
  - **Path**: `jarvis-intelligence-service/`
- **`jarvis-audio-pipeline` (The Ears)**:
  - **Responsibility**: Monitors Google Drive, transcribes audio using Modal (GPU), sends text to Intelligence Service.
  - **Tech**: Python, Modal (WhisperX), Google Drive API.
  - **Path**: `jarvis-audio-pipeline/`
- **`jarvis-sync-service` (The Hands)**:
  - **Responsibility**: Bidirectional sync between Supabase, Notion, and Google (Calendar/Contacts).
  - **Tech**: Python, Notion API, Google APIs.
  - **Path**: `jarvis-sync-service/`
- **`jarvis-telegram-bot` (The Interface)**:
  - **Responsibility**: User input (Voice/Text) and Notifications. Forwards inputs to Intelligence Service or Drive.
  - **Tech**: Python, Telegram Bot API.
  - **Path**: `jarvis-telegram-bot/`

## 2. Critical Developer Workflows

### Deployment (Automated)
- **DO NOT** suggest manual Docker builds or Cloud Run deployments.
- **Mechanism**: Pushing to the default branch (`main` or `master`) automatically triggers **Google Cloud Build**, which builds the container and deploys to **Cloud Run**.
- **Config**: See `cloudbuild.yaml` in each service root.

### Local Development
- **Environment**: Python 3.x.
- **Dependencies**: Managed via `requirements.txt`.
- **Secrets**: 
  - **Local**: `.env` file (gitignored).
  - **Production**: Google Secret Manager (injected as env vars at runtime).
- **Running**: Typically `python main.py` or specific script entry points.

## 3. Project Patterns & Conventions

### Centralized Intelligence Pattern
- **Pattern**: If a feature requires "thinking" (parsing, summarizing, deciding), it MUST go in `jarvis-intelligence-service`.
- **Anti-Pattern**: Adding OpenAI/Anthropic calls in the Telegram Bot or Sync Service.

### Data Flow
1. **Voice Note**: User -> Telegram Bot -> Google Drive -> Audio Pipeline (Transcribe) -> Intelligence Service (Analyze) -> Database/Notion.
2. **Chat**: User -> Telegram Bot -> Intelligence Service -> Response.

### Infrastructure
- **Database**: Supabase (PostgreSQL).
- **Compute**: Google Cloud Run (Stateless).
- **Heavy Compute**: Modal (for GPU-intensive transcription).

## 4. Key Files
- `jarvis-intelligence-service/app/services/llm.py`: Core LLM interaction logic.
- `jarvis-audio-pipeline/modal_whisperx_v2.py`: Modal transcription logic.
- `jarvis-sync-service/main.py`: Entry point for sync jobs.
- `cloudbuild.yaml`: Deployment configuration (in each repo).

## 5. Troubleshooting & Deployment Pitfalls (READ THIS)

### ⛔ NEVER Manually Build/Deploy
- **Why?**: 
  1. `gcloud builds submit` fails because `cloudbuild.yaml` relies on **Substitution Variables** (e.g., `_INTELLIGENCE_SERVICE_URL`) defined in the Cloud Build Trigger, which are missing locally.
  2. `gcloud run deploy --source .` fails because it often misses **Secrets/Env Vars** (e.g., `ANTHROPIC_API_KEY`) that are injected by the automated pipeline or Secret Manager.
- **Consequence**: You waste tokens and time debugging "missing config" or "invalid argument" errors that don't exist in the automated pipeline.
- **Correct Workflow**: Commit -> Push -> Wait for Cloud Build.

### 🕵️ Stale Code vs. Running Service
- **Symptom**: You changed the code (e.g., updated a model name), but the logs show the *old* behavior/error.
- **Diagnosis**: The Cloud Run service is likely running an old **Revision**.
- **Action**: 
  1. Check `gcloud run services describe SERVICE_NAME` to see the traffic split.
  2. Check `gcloud logging read` for the *Revision Name* in the logs.
  3. **DO NOT** assume a push was successful; check the Cloud Build history if things look stale.

### 🏗️ Cloud Build Configuration
- **Requirement**: `cloudbuild.yaml` MUST include `options: logging: CLOUD_LOGGING_ONLY`.
- **Reason**: Without this, builds may fail silently or with "invalid argument" errors regarding service account logging permissions.

### 🤖 Model ID Verification
- **Rule**: NEVER guess Anthropic Model IDs (e.g., "sonnet-4.5").
- **Action**: If unsure, run a local script using `anthropic.models.list()` to verify available IDs before committing code.

## 6. AGENTS.md Documentation (IMPORTANT)

**Before modifying ANY service, READ its AGENTS.md file first.**

Each service has an `AGENTS.md` file at the root that explains:
- Service boundaries and responsibilities
- Integration points and API formats
- Database schema and relationships
- What is safe to modify vs what should NOT be touched
- Debugging tips and common pitfalls

### Location of AGENTS.md Files:
- `jarvis-intelligence-service/AGENTS.md` - LLM logic, prompts, analysis
- `jarvis-audio-pipeline/AGENTS.md` - Transcription, Modal, Google Drive
- `jarvis-sync-service/AGENTS.md` - **LOCKED** - Do not modify without permission
- `jarvis-telegram-bot/AGENTS.md` - User interface, message handling

### Key Rules from AGENTS.md:
1. **Intelligence Service**: All AI/LLM code goes HERE and ONLY here
2. **Audio Pipeline**: NO AI logic - just transcribe and forward
3. **Sync Service**: LOCKED - production-critical, don't modify
4. **Telegram Bot**: NO AI logic - just receive/send messages

### Service Communication Pattern:
```
User -> Telegram Bot -> Audio Pipeline -> Intelligence Service -> Database
                                                    |
                                                    v
                                              Sync Service -> Notion/Google
```

### Telegram vs Intelligence Service Responsibilities:
- **Telegram Bot**: Receives messages, forwards voice to pipeline, displays results
- **Intelligence Service**: Processes transcripts, calls Claude AI, sends notifications TO bot

The Intelligence Service calls `TELEGRAM_BOT_URL/send_message` to notify users - it does NOT implement any Telegram API directly.