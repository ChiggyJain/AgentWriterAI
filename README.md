# AgentWriter AI

AgentWriter AI is a web application that turns a topic into a structured technical article. It uses a LangGraph workflow to decide whether research is needed, plan sections, write those sections in parallel, and optionally add technical images. The browser shows workflow progress as it runs and lets you preview or download the resulting Markdown.

## Features

- Classifies topics as evergreen or research dependent.
- Searches the web with Tavily when current information is useful.
- Plans an article and writes its sections concurrently with a Groq-hosted language model.
- Optionally creates and inserts technical images with Google Gemini.
- Streams workflow updates to the browser using Server-Sent Events (SSE).
- Stores LangGraph checkpoints in PostgreSQL and saves downloadable Markdown under `outputs/`.

## Architecture

```text
Browser (HTML, CSS, JavaScript)
        │ POST /api/run; SSE progress
        ▼
FastAPI app (app.py)
  ├── Jinja2 page and static assets
  ├── health and Markdown download endpoints
  └── LangGraph workflow (backend.py)
        ├── Router ── optional Tavily research
        ├── Article planner
        ├── Parallel section writers (Groq)
        └── Reducer ── optional Gemini images ── final Markdown
                         │
                         └── PostgreSQL checkpointer
```

### Main workflow

1. The router chooses whether the requested topic needs web research.
2. For research topics, Tavily results are collected and summarized as evidence.
3. The planner creates an article outline with section goals and writing requirements.
4. LangGraph fans out section-writing tasks and merges the finished sections in order.
5. The reducer may plan up to three explanatory images and asks Gemini to generate them. If image generation fails, the Markdown includes an explanatory fallback.
6. FastAPI streams progress to the UI and saves a run-specific copy at `outputs/<run-id>/blog.md`.

## Tech stack

- **Backend:** Python, FastAPI, Uvicorn, Pydantic, Jinja2
- **Workflow and LLM:** LangGraph, LangChain, Groq (`openai/gpt-oss-120b`)
- **Research:** Tavily via LangChain Community
- **Image generation:** Google GenAI SDK, Gemini (`gemini-2.5-flash-image`)
- **Persistence:** PostgreSQL, Psycopg, LangGraph PostgreSQL checkpointer
- **Frontend:** HTML, CSS, vanilla JavaScript

## Prerequisites

- Python 3.10 or newer (3.11 is a good default).
- PostgreSQL 14 or newer, either installed locally or run in Docker.
- API credentials:
  - `GROQ_API_KEY` is required to start the application.
  - `DATABASE_URL` is required to connect the PostgreSQL checkpointer.
  - `TAVILY_API_KEY` is needed for runs where the router chooses research.
  - `GOOGLE_API_KEY` is needed only when images are generated.
- Git and pip.

## Local installation

1. Clone the repository and change into the project directory.

   ```bash
   git clone <repository-url>
   cd AgentWriterAI
   ```

2. Create and activate a virtual environment.

   ```bash
   python -m venv .venv
   # macOS / Linux
   source .venv/bin/activate
   # Windows PowerShell
   .venv\Scripts\Activate.ps1
   ```

3. Install the pinned Python dependencies.

   ```bash
   python -m pip install --upgrade pip
   pip install -r requirements.txt
   ```

4. Start PostgreSQL. To use the included Docker Compose service:

   ```bash
   docker compose up -d postgres
   ```

   The compose file creates a database named `agent_writer_db` with user `db_user` and password `db_password` on `localhost:5432`.

5. Create a `.env` file in the project root:

   ```dotenv
   GROQ_API_KEY=your_groq_api_key
   DATABASE_URL=postgresql://db_user:db_password@localhost:5432/agent_writer_db?sslmode=disable
   TAVILY_API_KEY=your_tavily_api_key
   GOOGLE_API_KEY=your_google_api_key
   ```

   `DATABASE_URL` must include `sslmode=disable` for the local Compose database. The backend otherwise appends `sslmode=require`, which is generally intended for hosted PostgreSQL. For a hosted database, use its provided connection string and TLS settings. Omit the Tavily or Google key if you do not plan to use research or image generation, respectively; those features may fail or fall back when their credentials are unavailable.

## Run the application

From the project root, with the virtual environment active and `.env` configured:

```bash
python app.py
```

The development server runs at <http://127.0.0.1:8000> with Uvicorn reload enabled. Open that URL, enter a topic, and select **Run agent**. The page displays workflow progress and the generated article. Use the preview/Markdown controls to inspect it or download the Markdown file.

You can also start Uvicorn directly:

```bash
uvicorn app:app --host 127.0.0.1 --port 8000 --reload
```

## HTTP endpoints

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/` | Web application |
| `GET` | `/api/health` | Basic application health status |
| `POST` | `/api/run` | Start a workflow and stream events as `text/event-stream` |
| `GET` | `/api/runs/{run_id}/download` | Download the completed run's Markdown |

Example run request:

```bash
curl -N http://127.0.0.1:8000/api/run \
  -H 'Content-Type: application/json' \
  -d '{"topic":"Explain LangGraph subgraphs with a practical example"}'
```

## Project layout

```text
app.py                 FastAPI routes, SSE workflow streaming, output downloads
backend.py             LangGraph nodes, LLM integrations, and PostgreSQL checkpointer
requirements.txt       Pinned Python dependencies
docker-compose.yml     Local PostgreSQL service
templates/index.html   Browser application template
static/css/style.css   Application styling
static/js/app.js       Browser interaction and SSE client
outputs/               Per-run Markdown outputs
images/                Generated image assets
```

`outputs/` and `images/` are created automatically if missing. Generated files and `.env` credentials should not be committed.

## Troubleshooting

- **Startup reports a missing `GROQ_API_KEY` or `DATABASE_URL`:** check that `.env` is in the project root and contains both values. The workflow module loads configuration during import.
- **PostgreSQL connection fails:** confirm the database is running and the URL credentials/database match it. For local Compose, set `sslmode=disable` explicitly as shown above.
- **Research fails:** provide a valid `TAVILY_API_KEY`; research is only called for topics routed to that step.
- **Image generation fails:** provide a valid `GOOGLE_API_KEY` and check Gemini API access/quota. The workflow is designed to leave an explanatory note in the article when image generation fails.
- **Port 8000 is occupied:** choose another port in the Uvicorn command, for example `--port 8001`.
