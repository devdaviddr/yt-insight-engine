# yt-insight-engine

AI-powered semantic search and Q&A over YouTube videos using RAG, local LLMs, and vector embeddings.

## Overview

**yt-insight-engine** transforms your YouTube subscriptions into a private, searchable knowledge base. Instead of hunting through hours of video content for that one explanation you watched months ago, ask natural language questions and get precise answers with timestamp citations.

This system uses **Retrieval Augmented Generation (RAG)** to:
- Automatically monitor YouTube channels for new content
- Transcribe videos locally using Whisper
- Generate vector embeddings and store them in PostgreSQL
- Enable semantic search across your entire video library
- Synthesize grounded answers using Llama 3

**Key Features:**
- 🔒 **100% Local & Private** – No API keys, no cloud dependencies, no tracking
- 💰 **Zero Cost** – Runs entirely on your hardware using Ollama
- 🎯 **Semantic Search** – Find content by meaning, not just keywords
- ⏱️ **Timestamp Citations** – Jump directly to relevant video moments
- 🐳 **Dockerized** – One command to launch the entire stack

***

## Architecture

```text
                                  INTERNET (YouTube)
                                      ^     ^
                                      |     |
      (1) User Visits UI              |     | (3) Download Audio (yt-dlp)
      (Browser)                       |     |
          |                           |     |
          v                           |     |
+-----------------------------------------------------------------------+
|  HOST MACHINE (Port 8501)           |     |                           |
+-------------------------------------+-----+---------------------------+
|                                     |     |                           |
|  DOCKER NETWORK (yt-net)            |     |                           |
|                                     |     |                           |
|  +-------------------+       +------+-----+------+                    |
|  |   Streamlit UI    |       | Ingestion Service |                    |
|  | [LangChain Client]|<------|   (The Watcher)   |                    |
|  |                   |       |                   |                    |
|  | Ports: 8501:8501  |       |   [yt-dlp/RSS]    |                    |
|  +--------+-----+----+       +---------+---------+                    |
|           |     |                      |                              |
|           |     | (2) Search           | (4) Push Job                 |
|           |     |     Vector           |     (AMQP)                   |
|           |     v                      v                              |
|           |  +-----------------------------------+                    |
|  (5) Gen  |  |            RabbitMQ               |                    |
|   Query   |  |        (Message Broker)           |                    |
|   Embed   |  |                                   |                    |
|   (HTTP)  |  | Ports: 15672:15672 (Mgmt UI)      |                    |
|           |  +-----------------+-----------------+                    |
|           |                    |                                      |
|           |                    | (6) Pull Job                         |
|           |                    |     (AMQP)                           |
|           |                    v                                      |
|           |          +---------+---------+                            |
|           |          | Processing Worker |                            |
|           |          |    (The Brain)    |                            |
|           |          |                   |                            |
|           |          | [faster-whisper]  |                            |
|           +--------->|     [ffmpeg]      |                            |
|           ^          +----+---------+----+                            |
|           |               |         |                                 |
|  (9) Chat |      (7) Gen  |         | (8) Store                       |
|      With |      Embed    |         |     Data                        |
|      Data |      (HTTP)   |         |     (SQL)                       |
|           |               v         v                                 |
|  +--------+----------+    +---------+---------+                       |
|  |      Ollama       |    |    PostgreSQL     |                       |
|  |    (AI Model)     |    |   (Data Layer)    |                       |
|  |                   |    |                   |                       |
|  | [nomic-embed-text]|    |    [pgvector]     |                       |
|  |     [llama3]      |    |   [Videos/Subs]   |                       |
|  +-------------------+    +-------------------+                       |
|                                                                       |
+-----------------------------------------------------------------------+
```

**Pipeline Stages:**
1. **Ingestion Service** – Monitors RSS feeds, detects new videos
2. **Message Queue** – RabbitMQ manages background jobs
3. **Processing Worker** – Downloads audio, transcribes with Whisper, generates embeddings
4. **Vector Database** – PostgreSQL + pgvector stores 768-dim embeddings
5. **RAG Interface** – Streamlit UI for chat and subscription management

***

## Tech Stack

| Layer | Technology |
|-------|-----------|
| **Frontend** | Streamlit |
| **Backend** | Python 3.9, LangChain |
| **AI Models** | Ollama (Llama 3, nomic-embed-text) |
| **Database** | PostgreSQL 16 + pgvector |
| **Message Queue** | RabbitMQ |
| **Transcription** | faster-whisper |
| **Audio Processing** | yt-dlp, ffmpeg |
| **Orchestration** | Docker Compose |

***

## Prerequisites

- Docker & Docker Compose
- **Hardware:** Minimum 8GB RAM (16GB recommended for faster processing)
- **Optional:** NVIDIA GPU for faster transcription (requires `nvidia-docker`)

***

## Quick Start

### 1. Clone and Setup

```bash
git clone https://github.com/devdaviddr/yt-insight-engine.git
cd yt-insight-engine

# Create required directories
mkdir -p database ingestion_service processing_worker streamlit_app
```

### 2. Configure Environment

Create `.env` in the project root:

```ini
# Database Credentials
DB_USER=admin
DB_PASS=your_secure_password_here
DB_NAME=yt_knowledge_base

# RabbitMQ Credentials
RABBIT_USER=guest
RABBIT_PASS=guest
```

### 3. Initialize Database Schema

Create `database/init.sql`:

```sql
-- Enable pgvector extension
CREATE EXTENSION IF NOT EXISTS vector;

-- Channels: YouTube channels to monitor
CREATE TABLE IF NOT EXISTS channels (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    url TEXT NOT NULL,
    last_checked_at TIMESTAMP DEFAULT '1970-01-01'
);

-- Videos: Metadata for each video
CREATE TABLE IF NOT EXISTS videos (
    id TEXT PRIMARY KEY,
    channel_id TEXT REFERENCES channels(id),
    title TEXT NOT NULL,
    url TEXT NOT NULL,
    published_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status TEXT DEFAULT 'pending'
);

-- Transcript chunks with vector embeddings
CREATE TABLE IF NOT EXISTS transcript_chunks (
    id SERIAL PRIMARY KEY,
    video_id TEXT REFERENCES videos(id) ON DELETE CASCADE,
    chunk_text TEXT NOT NULL,
    start_time DOUBLE PRECISION,
    end_time DOUBLE PRECISION,
    embedding vector(768)  -- nomic-embed-text dimensions
);

-- Create HNSW index for fast similarity search
CREATE INDEX ON transcript_chunks USING hnsw (embedding vector_cosine_ops);
```

### 4. Launch the Stack

```bash
# Build and start all services
docker-compose up -d --build

# Pull AI models (one-time setup)
docker exec -it yt_ai ollama pull llama3
docker exec -it yt_ai ollama pull nomic-embed-text

# Watch processing logs
docker logs -f yt_worker
```

### 5. Access the Interface

Open your browser to **http://localhost:8501**

***

## Usage

### Adding Channels

1. Navigate to the **"Manage Subscriptions"** tab
2. Paste a YouTube channel URL (e.g., `https://www.youtube.com/@Fireship`)
3. Click **"Subscribe"**
4. The ingestion service will automatically queue new videos for processing

### Chatting with Your Knowledge Base

1. Go to the **"Chat with Knowledge"** tab
2. Ask questions like:
   - *"How does NetworkChuck explain VLANs?"*
   - *"What JavaScript frameworks were mentioned this week?"*
   - *"Summarize the Docker tutorial from TechWorld with Nana"*
3. Get AI-generated answers with clickable timestamp citations

### Monitoring Progress

- **RabbitMQ Management UI:** http://localhost:15672 (guest/guest)
- **Processing Logs:** `docker logs -f yt_worker`
- **Database Status:** Check video status in Streamlit's "Manage Subscriptions" tab

***

## Project Structure

```
yt-insight-engine/
├── docker-compose.yml          # Orchestration config
├── .env                        # Environment variables
├── database/
│   └── init.sql               # PostgreSQL schema
├── ingestion_service/
│   ├── Dockerfile
│   ├── requirements.txt
│   └── main.py                # RSS monitor + job queue
├── processing_worker/
│   ├── Dockerfile
│   ├── requirements.txt
│   └── worker.py              # Whisper + embeddings
└── streamlit_app/
    ├── Dockerfile
    ├── requirements.txt
    └── app.py                 # RAG UI
```

***

## Configuration

### GPU Acceleration (Optional)

To enable NVIDIA GPU support for faster transcription, uncomment the following in `docker-compose.yml`:

```yaml
ollama:
  deploy:
    resources:
      reservations:
        devices:
          - driver: nvidia
            count: 1
            capabilities: [gpu]
```

Then modify `processing_worker/worker.py`:

```python
model = WhisperModel("base", device="cuda", compute_type="float16")
```

### Adjusting Processing Speed

In `processing_worker/worker.py`, change the Whisper model size:
- `tiny` – Fastest, least accurate
- `base` – Balanced (recommended)
- `small` – Slower, more accurate
- `medium` – High accuracy, GPU recommended

### Custom Chunk Size

Modify the chunking logic in `worker.py`:

```python
if len(chunk_buffer) > 500:  # Increase for longer context
```

***

## Troubleshooting

### Videos Stay in "Pending" Status
- Check worker logs: `docker logs -f yt_worker`
- Verify RabbitMQ is running: `docker ps | grep rabbitmq`
- Ensure Ollama models are pulled: `docker exec -it yt_ai ollama list`

### Slow Transcription
- Switch to GPU acceleration (see Configuration)
- Use a smaller Whisper model (`tiny` instead of `base`)
- Process fewer videos concurrently

### "No relevant info found"
- Ensure videos have status `completed` in the database
- Check embedding dimension matches (768 for nomic-embed-text)
- Verify pgvector index exists: `\d transcript_chunks` in psql

***

## Performance Notes

**Average Processing Time (CPU):**
- 10-minute video: ~5 minutes
- 1-hour video: ~30 minutes

**Disk Usage:**
- PostgreSQL + embeddings: ~50MB per hour of video
- Ollama models: ~4GB (one-time)

***

## Roadmap

- [ ] Multi-language transcription support
- [ ] Playlist bulk import
- [ ] Export conversations as markdown
- [ ] Webhook notifications for new videos
- [ ] Dark mode UI

***

## Contributing

Contributions are welcome! Please open an issue before submitting major changes.

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

***

## License

MIT License - See [LICENSE](LICENSE) for details.

***

## Acknowledgments

- **LangChain** for RAG orchestration
- **Ollama** for local LLM inference
- **pgvector** for efficient vector similarity search
- **faster-whisper** for optimized transcription

