#  Jarvis Ecosystem

Welcome to the Jarvis Ecosystem! This is a personal AI assistant that helps you organize your life by processing voice notes, syncing your contacts and calendar, and providing intelligent insights.

The system is composed of 4 microservices that work together.

##  Architecture Overview

```mermaid
graph TD
    User[User] -->|Voice Note| Telegram[Telegram Bot]
    User -->|Upload| Drive[Google Drive]
    
    Telegram -->|Upload| Drive
    
    Drive -->|Trigger| AudioPipeline[Audio Pipeline]
    
    AudioPipeline -->|Transcribe| Modal[Modal (WhisperX)]
    AudioPipeline -->|Analyze| Intelligence[Intelligence Service]
    
    Intelligence -->|LLM| Claude[Claude 3.5 Haiku]
    Intelligence -->|Save| Supabase[(Supabase DB)]
    
    Sync[Sync Service] <-->|Bi-directional| Supabase
    Sync <-->|Bi-directional| Notion[Notion]
    Sync <-->|Bi-directional| Google[Google Contacts/Calendar]
```

##  The 4 Microservices

| Service | Description | Tech Stack |
|---------|-------------|------------|
| **[Audio Pipeline](https://github.com/JulienMaterno/jarvis-audio-pipeline)** | Monitors Google Drive, transcribes audio using Modal (GPU), and orchestrates the flow. | Python, Modal, Google Drive API |
| **[Intelligence Service](https://github.com/JulienMaterno/jarvis-intelligence-service)** | The "Brain". Receives transcripts, analyzes them with Claude AI, and saves structured data. | FastAPI, Cloud Run, Anthropic |
| **[Sync Service](https://github.com/JulienMaterno/jarvis-sync-service)** | Keeps your data in sync across Notion, Google Contacts, Calendar, and Supabase. | Python, Notion API, Google APIs |
| **[Telegram Bot](https://github.com/JulienMaterno/jarvis-telegram-bot)** | Simple interface to send voice notes to the system on the go. | Python, Telegram API |

---

##  Getting Started Guide

### 1. Clone the Ecosystem

To set this up, you should clone all repositories into a single folder:

```bash
mkdir Jarvis
cd Jarvis

# Clone the documentation (this repo)
git clone https://github.com/JulienMaterno/jarvis-ecosystem.git

# Clone the services
git clone https://github.com/JulienMaterno/jarvis-audio-pipeline.git
git clone https://github.com/JulienMaterno/jarvis-intelligence-service.git
git clone https://github.com/JulienMaterno/jarvis-sync-service.git
git clone https://github.com/JulienMaterno/jarvis-telegram-bot.git
```

### 2. Prerequisites & Accounts

You will need accounts for the following services:

*   **Supabase**: For the database. [Sign up here](https://supabase.com/).
*   **Modal**: For GPU-powered transcription. [Sign up here](https://modal.com/).
*   **Anthropic**: For the AI intelligence (Claude). [Get API Key](https://console.anthropic.com/).
*   **Google Cloud**: For Drive, Calendar, and Contacts APIs. [Console](https://console.cloud.google.com/).
*   **Notion**: For the frontend UI. [Developers](https://www.notion.so/my-integrations).
*   **Telegram**: To create your bot. Talk to [@BotFather](https://t.me/botfather).

### 3. Database Setup (Supabase)

1.  Create a new project in Supabase.
2.  Go to the **SQL Editor** in the Supabase dashboard.
3.  Run the following SQL scripts to create the necessary tables:

**Core Schema (Contacts, Meetings, Logs):**
```sql
-- Copy content from: jarvis-sync-service/migrations/schema.sql
-- (See file for full content)
```

**Calendar & Email Schema:**
```sql
-- Copy content from: jarvis-sync-service/migrations/add_calendar_and_mail.sql
-- (See file for full content)
```

**Intelligence Schema (Transcripts, Tasks, Reflections):**
```sql
-- Transcripts Table
CREATE TABLE IF NOT EXISTS transcripts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    filename TEXT NOT NULL,
    content TEXT,
    summary TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT timezone('utc'::text, now()) NOT NULL,
    processed BOOLEAN DEFAULT FALSE,
    metadata JSONB DEFAULT '{}'::jsonb
);

-- Tasks Table (AI Generated)
CREATE TABLE IF NOT EXISTS tasks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    description TEXT NOT NULL,
    status TEXT DEFAULT 'todo',
    due_date TIMESTAMP WITH TIME ZONE,
    source_transcript_id UUID REFERENCES transcripts(id),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT timezone('utc'::text, now())
);

-- Reflections Table (AI Generated)
CREATE TABLE IF NOT EXISTS reflections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content TEXT NOT NULL,
    tags TEXT[],
    source_transcript_id UUID REFERENCES transcripts(id),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT timezone('utc'::text, now())
);
```

### 4. Service Setup

Follow the detailed setup guides in each repository:

1.  **[Setup Intelligence Service](https://github.com/JulienMaterno/jarvis-intelligence-service)** (Deploy this first)
2.  **[Setup Audio Pipeline](https://github.com/JulienMaterno/jarvis-audio-pipeline)** (Connects to Intelligence Service)
3.  **[Setup Sync Service](https://github.com/JulienMaterno/jarvis-sync-service)** (Runs periodically to sync data)
4.  **[Setup Telegram Bot](https://github.com/JulienMaterno/jarvis-telegram-bot)** (Optional, for mobile input)

---

##  Common Operations

### How to process a voice note manually?
Drop an audio file into the configured Google Drive folder. The `jarvis-audio-pipeline` will pick it up automatically.

### How to force a sync?
Run the sync script manually:
```bash
cd jarvis-sync-service
python run_full_sync.py
```

### Where do I see my data?
*   **Raw Data**: Supabase Dashboard
*   **User Interface**: Your Notion Workspace (Meetings, Tasks, CRM databases)


---

##  Legal & Privacy Disclaimer

*   **Data Privacy**: This system is designed to be **self-hosted**. You own your data. However, by using third-party APIs (Google, Anthropic, Notion, Telegram), you are subject to their respective privacy policies.
*   **Costs**: You are responsible for all API usage costs.
    *   **Modal**: Charges for GPU usage (transcription).
    *   **Anthropic**: Charges per token (AI analysis).
    *   **Google Cloud**: May incur costs for storage or Cloud Run usage.
*   **License**: This project is open-source under the **MIT License**. You are free to use, modify, and distribute it, but it comes with **no warranty**.
*   **Trademarks**: 'Jarvis' is a reference to the AI assistant from Marvel's Iron Man. This project is a personal hobby project and is not affiliated with, endorsed by, or connected to Marvel Studios or Disney.

