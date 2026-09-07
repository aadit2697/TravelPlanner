# TravelPlanner — TripMate AI

A multi-agent AI travel planner built with **LangGraph**, **FastAPI**, and **Groq**. Give it a natural-language request (e.g. _"Plan a 7 day Tokyo trip from Mumbai"_) and it searches live flights, discovers hotels, and generates a complete, budget-aware, day-by-day itinerary.

## Features

- **Multi-agent LangGraph pipeline** — a linear graph of four specialized agents.
- **Live flight lookup** via the AviationStack API (`tools/flight_tool.py`).
- **Hotel discovery** via Tavily web search (`tools/tavily_tool.py`).
- **LLM itinerary + final response** generation using Groq.
- **Conversation persistence** using a PostgreSQL checkpointer (LangGraph `PostgresSaver`), keyed by `thread_id`.
- **Web UI** (FastAPI + Jinja2 + vanilla JS) with Markdown rendering, copy-to-clipboard, and PDF download.

## Architecture

```
START → flight_agent → hotel_agent → itinerary_agent → final_agent → END
```

| Agent | Role |
|-------|------|
| `flight_agent` | Fetches live flight data for the requested route (AviationStack). |
| `hotel_agent` | Searches for hotel options (Tavily). |
| `itinerary_agent` | Uses the LLM to build a practical, budget-aware itinerary. |
| `final_agent` | Uses the LLM to format the final response (summary, flights, hotels, day-by-day plan, budget, recommendations). |

State flows through a `TravelState` `TypedDict` and is checkpointed to PostgreSQL so each `thread_id` retains its history.

## Tech Stack

- **Orchestration:** LangGraph 1.x
- **LLM:** Groq — `openai/gpt-oss-120b`
- **Web framework:** FastAPI + Uvicorn
- **Templating / frontend:** Jinja2, vanilla JS, `marked` (Markdown), `html2pdf` (PDF export)
- **Database:** PostgreSQL (Render-hosted) via `psycopg`
- **Tools/APIs:** AviationStack (flights), Tavily (search)

## Project Structure

```
TravelPlanner/
├── app.py              # FastAPI app + REST endpoints
├── backend.py          # LangGraph agents, graph, PostgreSQL checkpointer
├── test.py             # CLI entry point for quick testing
├── tools/
│   ├── flight_tool.py  # AviationStack live flight search
│   └── tavily_tool.py  # Tavily web search
├── templates/
│   └── index.html      # Web UI
├── static/
│   ├── script.js       # Frontend logic (fetch → /api/travel, render, PDF)
│   └── style.css
├── requirements.txt
├── Dockerfile
└── .env                # API keys + DATABASE_URL (not committed)
```

## Setup

1. **Create and activate a virtual environment (Python 3.11):**

   ```bash
   python3.11 -m venv venv
   source venv/bin/activate
   ```

2. **Install dependencies:**

   ```bash
   pip install --upgrade pip
   pip install -r requirements.txt
   ```

3. **Configure environment variables** in a `.env` file:

   ```env
   DATABASE_URL=postgresql://<user>:<password>@<host>/<db>?sslmode=require
   GROQ_API_KEY=your_groq_key
   AVIATIONSTACK_API_KEY=your_aviationstack_key
   TAVILY_API_KEY=your_tavily_key
   DEFAULT_ORIGIN_IATA=BOM
   # Optional LangSmith tracing
   LANGSMITH_TRACING=true
   LANGSMITH_API_KEY=your_langsmith_key
   LANGSMITH_PROJECT=travel-agent
   ```

   > **Note:** The `DATABASE_URL` must include `?sslmode=require` (with the `?` separator) so the database name isn't merged with the SSL parameter. `backend.py` also appends it automatically if missing.

## Running

**Web app:**

```bash
python app.py
```

Then open http://127.0.0.1:8000, enter a travel request, and click **Generate Plan**.

**CLI (quick test):**

```bash
python test.py
```

## API

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET`  | `/` | Serves the web UI. |
| `POST` | `/api/travel` | Body: `{ "message": string, "thread_id": string \| null }` → returns the generated plan. |
| `GET`  | `/health` | Health check. |

## Docker

```bash
docker build -t travelplanner .
docker run -p 8000:8000 --env-file .env travelplanner
```

## Troubleshooting / Notes

- **Groq model:** `llama-3.3-70b-versatile` and `llama-3.1-8b-instant` were deprecated/shut down (Aug 16, 2026). This project uses `openai/gpt-oss-120b`; alternatives include `qwen/qwen3.6-27b` and `openai/gpt-oss-20b`.
- **Database URL:** A missing `?` before `sslmode=require` causes a `database "...sslmode=require" does not exist` error — make sure the separator is present.
- **Frontend not responding to "Generate Plan":** Ensure `static/script.js` is saved (non-empty) and hard-refresh the browser (`Cmd+Shift+R`) to clear cached assets.
- **Security:** Never commit `.env`. Rotate any keys that have been exposed and add `.env` to `.gitignore`.
