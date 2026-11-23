# Customer Support Copilot

A support-agent workspace for banking customer tickets. The FastAPI service stores customers, tickets, and reply drafts in SQLite; it uses a Groq chat model to prepare responses grounded in a local banking knowledge base, customer or company memories, and support lookup tools. A Streamlit dashboard lets agents create tickets, review and edit drafts, inspect the context used, and accept or discard drafts.

Drafts are suggestions for an agent to review. Accepting a draft marks its ticket resolved and stores the reviewed resolution in customer memory. The copilot includes input and output guardrails, PII redaction, and JSONL tracing.

## Features

- Ticket and customer management through a REST API.
- Draft generation with banking policy retrieval (RAG), relevant customer and company memories, and plan/SLA or open-ticket lookup tools when the ticket calls for them.
- Agent review actions: edit, accept, or discard a draft.
- Knowledge-base ingestion from Markdown and text files in `knowledge_base/`.
- Memory search and inspection endpoints and dashboard controls.
- Optional Docker Compose deployment and GitHub Actions EC2 deployment workflow.

## Requirements

- Python 3.11 or newer (the Docker image and CI use Python 3.11).
- A Groq API key for AI draft generation and scope-classifier fallback.
- An embedding provider for customer memory: Google (`GOOGLE_API_KEY`, recommended), OpenAI (`OPENAI_API_KEY`), or local Hugging Face embeddings (`ENABLE_LOCAL_EMBEDDINGS=true`). The local option may download a model on first use.
- Docker Compose is optional if running the services directly.

## Local setup

1. Create and activate a virtual environment, then install dependencies:

   ```bash
   python -m venv .venv
   # macOS/Linux
   source .venv/bin/activate
   # Windows PowerShell: .venv\Scripts\Activate.ps1
   python -m pip install -r requirements.txt
   ```

2. Create a `.env` file in the repository root. Supply a real Groq key and one memory embedding option:

   ```dotenv
   GROQ_API_KEY=your_groq_api_key
   GROQ_MODEL=openai/gpt-oss-20b
   GOOGLE_API_KEY=your_google_api_key
   # Optional; defaults are shown above.
   # ENABLE_LOCAL_EMBEDDINGS=true
   ```

   Google embeddings are used for both knowledge retrieval and Mem0 when configured. Without `GOOGLE_API_KEY`, knowledge retrieval uses Chroma's default embedding function; Mem0 still needs OpenAI or local embeddings. Never commit API keys.

3. Start the API in one terminal:

   ```bash
   python main.py
   ```

   It listens on `http://localhost:8000` by default. The SQLite database and Chroma stores are created under `data/` on first startup.

4. In another terminal, start the dashboard:

   ```bash
   streamlit run app.py
   ```

   Open `http://localhost:8501`. The dashboard uses `API_BASE_URL` (default `http://localhost:8000`) to reach the API.

5. In the dashboard, use **Ingest Knowledge Base** to index the banking reference files before generating policy-based replies. Alternatively:

   ```bash
   curl -X POST http://localhost:8000/api/knowledge/ingest \
     -H "Content-Type: application/json" \
     -d '{"clear_existing": false}'
   ```

FastAPI's interactive API documentation is at `http://localhost:8000/docs`; the health check is `http://localhost:8000/health`.

## API overview

| Method | Endpoint | Purpose |
| --- | --- | --- |
| `GET` | `/health` | Health status |
| `POST` | `/api/tickets` | Create a ticket; `auto_generate` defaults to `true` |
| `GET` | `/api/tickets` | List recent tickets |
| `GET` | `/api/tickets/{ticket_id}` | Get a ticket |
| `POST` | `/api/tickets/{ticket_id}/generate-draft` | Generate a draft manually |
| `GET` | `/api/drafts/{ticket_id}` | Get the latest draft for a ticket |
| `PATCH` | `/api/drafts/{draft_id}` | Edit a draft and/or set status to `pending`, `accepted`, or `discarded` |
| `POST` | `/api/knowledge/ingest` | Index knowledge files; optionally clear the collection first |
| `GET` | `/api/customers/{customer_id}/memories` | List customer and company memories |
| `GET` | `/api/customers/{customer_id}/memory-search?query=...` | Search customer and company memories |

Ticket creation requires a valid `customer_email`, `subject` (at least 3 characters), and `description` (at least 10 characters). Priority can be `low`, `medium`, `high`, or `urgent`.

## Docker Compose

Create `.env` as described above, then run from the repository root:

```bash
docker compose up --build
```

The API is published on port `8000` and the dashboard on port `8501`. Compose persists database/vector data and the Hugging Face model cache in named volumes. Stop the services with `docker compose down`; named volumes remain available.

## Project layout

```text
.
|-- app.py                         # Streamlit support dashboard
|-- main.py                        # FastAPI/Uvicorn entry point
|-- customer_support_agent/
|   |-- api/                       # App factory, dependencies, and HTTP routers
|   |-- core/settings.py           # Environment-backed configuration and paths
|   |-- integrations/              # Chroma RAG, Mem0 memory, support tools
|   |-- observability/             # JSONL tracing
|   |-- repo/sqlite/               # SQLite persistence for customers/tickets/drafts
|   |-- schema/                    # Pydantic API request/response models
|   `-- services/                  # Copilot, draft, knowledge, and guardrail logic
|-- knowledge_base/                # Banking policy and FAQ source documents
|-- data/                          # Runtime SQLite, Chroma, and trace data
|-- test/                          # Automated tests
|-- docs/deployment.md             # EC2 and GitHub Actions deployment guide
|-- Dockerfile
`-- docker-compose.yml
```

The knowledge collection reads top-level `.md` and `.txt` files from `knowledge_base/`. The current references cover savings-account rules, ATM cash withdrawals, account charges/minimum balances, and KYC/profile updates. Runtime files in `data/` are generated locally and should be treated as application data.

## Configuration

Settings are read from environment variables and `.env` by `customer_support_agent/core/settings.py`. Common options:

| Variable | Default | Purpose |
| --- | --- | --- |
| `GROQ_API_KEY` | empty | Required to initialize the draft copilot |
| `GROQ_MODEL` | `openai/gpt-oss-20b` | Groq chat model |
| `GOOGLE_API_KEY` | empty | Google embeddings for RAG and Mem0 |
| `OPENAI_API_KEY` | empty | Alternative Mem0 embedding provider |
| `ENABLE_LOCAL_EMBEDDINGS` | `false` | Use local Hugging Face embeddings for Mem0 |
| `API_HOST` / `API_PORT` | `0.0.0.0` / `8000` | API bind address and port |
| `API_BASE_URL` | `http://localhost:8000` | Dashboard's API address |
| `GUARDRAILS_ENABLED` | `true` | Enable input/output safety checks and PII handling |
| `TRACE_ENABLED` | `true` | Write request traces under `data/traces/` |
| `RAG_TOP_K` / `MEM0_TOP_K` | `4` / `5` | Retrieval limits used during draft generation |

SQLite defaults to `data/support.db`; Chroma stores default to `data/chroma_rag/` and `data/chroma_mem0/`. Paths are relative to the project root unless absolute paths are supplied.

## Tests

Install the runtime requirements and pytest, then run the test suite:

```bash
python -m pip install -r requirements.txt pytest
python -m pytest -q
```

GitHub Actions also defines CI and EC2 deployment workflows. See [docs/deployment.md](docs/deployment.md) for the EC2 setup, required GitHub secrets, and deployment steps.
