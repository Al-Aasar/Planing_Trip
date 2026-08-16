# ✈️ Planing_Trip AI — Multi-Agent Travel Planner

An AI-powered travel planning system built on multiple **agents** orchestrated with **LangGraph**, served through a **FastAPI** web app. The system pulls together flight, hotel, weather, and budget information, builds a full trip itinerary, and lets the user review, approve, or request changes before the final plan is delivered (Human-in-the-Loop).

---

## 🧠 How It Works

The user types a travel request in natural language (e.g. "Plan me a 5-day trip to Istanbul on a tight budget"). Then:

1. The **Supervisor Agent** checks the request (guardrail) to confirm it's actually travel-related, and decides which agents are needed plus extracts trip details (destination, duration, budget, etc.) from the message.
2. Based on the supervisor's decision, one or more of the following agents run in order:
   - **Flight Agent** – airport info, airlines, and typical flight duration (via the AviationStack MCP server).
   - **Hotel Agent** – hotel suggestions via web search (Tavily MCP).
   - **Weather Agent** – current weather and forecast (a custom MCP server on top of the OpenWeather API).
   - **Budget Agent** – analyzes whether the trip is realistic for the given budget.
3. The **Itinerary Agent** combines all results into a draft itinerary.
4. **Human Approval** — execution pauses (interrupt) and waits for the user to approve the draft or send revision feedback through the UI.
5. The **Final Agent** produces the final response after approval or revision, making sure the current weather data is reported accurately.

The whole flow is built as a **graph** with LangGraph, with each conversation's state persisted to **PostgreSQL** (`PostgresSaver`), so a session can be resumed later using the same `thread_id`.

---

## 📁 Project Structure

```
New folder/
├── app.py                       # FastAPI app (API + web page)
├── backend.py                   # Builds the LangGraph, routing logic, and DB connection
├── state.py                     # Trip state definition (TravelState)
├── mcp_client.py                # Connects to MCP servers (Tavily / AviationStack / Weather)
├── custom_weather_mcp_server.py # Custom MCP server for weather (OpenWeather API)
├── utils.py                     # Shared helper functions (LLM helpers, JSON parsing)
├── .env.example                 # Template for required environment variables
├── Agents/
│   ├── supervisor_agent.py      # Request validation + agent routing
│   ├── flight_agent.py
│   ├── hotel_agent.py
│   ├── weather_agent.py
│   ├── budget_agent.py
│   ├── itinerary_agent.py
│   └── final_agent.py
├── templates/
│   └── index.html               # Web UI (Planing_Trip AI)
└── static/
    ├── style.css
    └── script.js
```

---

## 🛠️ Tech Stack

| Technology | Purpose |
|---|---|
| **FastAPI** | Backend API and serving the web page |
| **LangGraph** | Building the multi-agent graph and controlling flow (including the human-in-the-loop step) |
| **LangChain / langchain-groq** | Communicating with the LLM |
| **Groq (`llama-3.3-70b-versatile`)** | The LLM used across all agents |
| **MCP (Model Context Protocol)** via `langchain-mcp-adapters` | Connecting external tools as agent tools |
| **Tavily MCP** | Web search (used by the Hotel Agent) |
| **AviationStack MCP** | Flight data (Flight Agent) |
| **OpenWeather API** | Via a custom MCP server (`custom_weather_mcp_server.py`) for current weather and forecast |
| **PostgreSQL** (`psycopg` + `PostgresSaver`) | Persisting each conversation's state (checkpointing) |
| **Jinja2** | Rendering the `index.html` page |
| **nest_asyncio** | Allows calling async functions from sync code inside FastAPI |

---

## ⚙️ Setup & Running

### 1. Requirements
- Python 3.12+
- A PostgreSQL database (e.g. hosted on Render or any other provider)
- `uv` / `uvx` installed (used to run the AviationStack MCP server)

### 2. Install dependencies
There's no `requirements.txt` included in the project. Based on the imports in the code, the main dependencies are:

```bash
pip install fastapi uvicorn jinja2 python-multipart \
    langgraph langchain-core langchain-groq langchain-mcp-adapters \
    psycopg[binary] python-dotenv certifi requests nest_asyncio \
    "mcp[cli]"
```

### 3. Environment variables
Copy `.env.example` to `.env` and fill in the values:

```env
GROQ_API_KEY=your_groq_api_key
DATABASE_URL=your_postgresql_database_url
TAVILY_API_KEY=your_tavily_api_key
AVIATION_STACK_API_KEY=your_aviationstack_api_key
OPENWEATHER_API_KEY=your_openweather_api_key
```

### 4. Run the server

```bash
python app.py
```

or:

```bash
uvicorn app:app --host 127.0.0.1 --port 8000 --reload
```

Then open your browser at: `http://127.0.0.1:8000`

### 5. Health check
```
GET /health
```

---

## 🔌 API Endpoints

| Method | Endpoint | Purpose |
|---|---|---|
| `GET` | `/` | Main web UI page |
| `POST` | `/api/travel` | Submit a new travel request and start planning |
| `POST` | `/api/travel/approve` | Approve the draft itinerary or send revision feedback (human-in-the-loop) |
| `GET` | `/health` | Service health check |

**Example request to `/api/travel`:**
```json
{
  "message": "Plan me a week-long trip to Dubai with a $2000 budget",
  "thread_id": null
}
```

**Example request to `/api/travel/approve`:**
```json
{
  "thread_id": "user_xxxxxxxx",
  "approved": false,
  "feedback": "I want cheaper hotels closer to downtown"
}
```

---

## 🧩 Technical Notes

- The **guardrail** in `supervisor_agent.py` blocks requests unrelated to travel or harmful/illegal requests, while still allowing valid requests that are missing some details.
- Each agent is designed so a failing external MCP tool doesn't break the whole flow (e.g., if the hotel search fails, a general fallback response is used instead of crashing).
- Each conversation's state (constraints, per-agent results, approval, messages, etc.) is tracked in `TravelState` (see `state.py`) and persisted via `PostgresSaver`.
- The `__pycache__` folders inside `Agents/` are compiled Python cache files and should be deleted or added to `.gitignore`.

---

## 📌 Project Status

This is a demo/learning project showcasing how to build a multi-agent system with LangGraph + MCP + FastAPI. Before real-world use, it needs:
- Valid API keys for each service (Groq, Tavily, AviationStack, OpenWeather).
- A working PostgreSQL database for checkpointing.
- `uv`/`uvx` installed to run the AviationStack MCP server.
