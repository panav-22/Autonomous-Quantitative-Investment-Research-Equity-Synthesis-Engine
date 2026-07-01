# Autonomous Quantitative Investment Research & Equity Synthesis Engine

An autonomous, production-grade financial analysis platform that acts as a self-contained equity research desk. Using a multi-agent orchestration pattern built on **CrewAI**, **FastAPI**, and **Streamlit**, the engine automatically aggregates financial statements, conducts quantitative performance benchmarking against market indices, scrapes qualitative news sentiment, and synthesizes investment recommendations (BUY, SELL, or HOLD). All artifacts are durably archived in **Azure Blob Storage** and logged in **Azure Database for PostgreSQL**.

---

## Key Features

- **Decoupled Service Architecture**: A lightweight Streamlit UI communicates with a FastAPI backend, separating the user interface from agent orchestration.
- **Sequential Multi-Agent Pipeline (CrewAI)**:
  - **Senior Quantitative Analyst**: Extracts 11 core balance sheet metrics (P/E ratio, market cap, volatility, EPS, etc.) and evaluates historical S&P 500 (`SPY`) benchmark returns.
  - **Chief Investment Strategist**: Ingests the quantitative analyst's metrics and synthesizes them with real-time news headlines to produce a unified recommendation, eliminating token hallucinations.
- **Qualitative News Scraping (Firecrawl)**: Crawls and parses search-relevant financial news directly into clean Markdown, bypassing messy HTML formatting to minimize prompt token usage.
- **Relational & Object Cloud Persistence**: Saves generated markdown reports to Azure Blob Storage and logs query metadata in Azure PostgreSQL via SQLAlchemy ORM.
- **observability & Telemetry**: Integrated with OpenTelemetry for Azure Monitor/Application Insights and LangSmith for detailed LLM trace debugging.
- **Configuration Cache**: Optimizes environment loading by caching validated configurations through Pydantic Settings and LRU caching.

---

## System Architecture & Data Flow

The diagram below outlines how a user request flows through the frontend, API gateway, agent reasoning workspace, and cloud storage systems:

```mermaid
graph TD
    UI[Streamlit Analyst Portal] <-->|HTTP POST /api/v1/analyze| API[FastAPI Gateway]
    
    subgraph Quant-Sentiment Agent Workspace (CrewAI)
        API -->|Orchestrate Crew| Crew[Financial Research Crew]
        Crew -->|Task 1: Fundamental Analysis| Agent1[Senior Quantitative Analyst]
        Agent1 -->|yfinance API| Tool1[Fundamental Metrics & Benchmarking Tools]
        
        Agent1 -->|Inject Quantitative Context| Agent2[Chief Investment Strategist]
        Crew -->|Task 2: News Sentiment & Verdict| Agent2
        Agent2 -->|Firecrawl API| Tool2[Sentiment Web Search Tool]
    end

    Crew -->|Generate Report.md| API
    
    subgraph Data Archival & Persistence
        API -->|Durable upload| Blob[Azure Blob Storage]
        API -->|Log report metadata| DB[Azure Database for PostgreSQL]
    end
    
    subgraph Enterprise Observability
        Crew -.->|Prompt Tracing| LS[LangSmith]
        API -.->|Telemetry & Spans| AZM[Azure Monitor via OpenTelemetry]
    end
```

### Execution Flow:
1. **User Request**: The research analyst inputs a stock ticker (e.g., `AMZN`) into the **Streamlit UI**. Streamlit triggers an HTTP POST request to the **FastAPI `/api/v1/analyze`** endpoint.
2. **Quantitative Extraction**: The FastAPI controller invokes `run_financial_crew`. The **Quantitative Analyst** agent retrieves current valuation statistics from Yahoo Finance (`yfinance`) and compares the 365-day historical performance against `SPY`.
3. **Sentiment Synthesis**: The Quantitative Analyst's output is passed directly into the **Investment Strategist's** task context. The Strategist uses **Firecrawl** to pull the top 3 news articles matching recent search terms, matches the qualitative sentiment with quantitative values, and issues a BUY, SELL, or HOLD verdict.
4. **Cloud Persistence**: The FastAPI gateway uploads the resulting Markdown file to **Azure Blob Storage** and commits a metadata record to **Azure PostgreSQL** using **SQLAlchemy ORM**.
5. **UI Rendering**: The JSON response (containing the report text, execution logs, and Blob URLs) is returned to Streamlit for presentation.

---

## Directory Structure

```text
├── Dockerfile                         # Deployment container configuration
├── pyproject.toml                     # Project packaging and dependencies
├── uv.lock                            # Lockfile for dependency integrity
├── main.py                            # CLI-based execution entry point
├── README.md                          # Documentation
├── frontend/
│   ├── app.py                         # Streamlit UI dashboard
│   └── requirements.txt               # Frontend dependency manifest
├── src/
│   ├── agents/
│   │   ├── agents.py                  # Agent persona configurations
│   │   ├── tasks.py                   # Sequential Task configurations
│   │   ├── crew.py                    # CrewAI team orchestration
│   │   └── tools/
│   │       ├── financial.py           # yfinance fundamental extraction tool
│   │       └── scraper.py             # Firecrawl web scraping tool
│   ├── api/
│   │   ├── main.py                    # FastAPI server entry point
│   │   ├── models.py                  # Pydantic validation schemas
│   │   └── routes.py                  # REST API routes
│   └── shared/
│       ├── config.py                  # Pydantic Settings configuration cache
│       ├── database.py                # SQLAlchemy PostgreSQL service
│       └── storage.py                 # Azure Blob Storage service
```

---

## Tech Stack

- **Frontend**: Streamlit
- **Backend API**: FastAPI, Uvicorn
- **AI/ML Orchestration**: CrewAI, OpenAI GPT-4o
- **APIs**: yfinance (Yahoo Finance), firecrawl-py (Firecrawl Search SDK)
- **Databases**: PostgreSQL (via SQLAlchemy ORM & psycopg2)
- **Cloud Infrastructure**: Azure Blob Storage, Azure Database for PostgreSQL
- **Observability**: OpenTelemetry, Azure Monitor Opentelemetry, LangSmith
- **Environment & Packager**: Python 3.12, Pydantic, Pydantic Settings, `uv`

---

## Installation & Setup

### Prerequisites
- Python 3.12+
- `uv` package manager (recommended: `curl -LsSf https://astral.sh/uv/install.sh | sh`)
- An OpenAI API Key
- A Firecrawl API Key
- Connection strings for Azure Blob Storage and Azure PostgreSQL

### 1. Clone & Navigate
```bash
git clone https://github.com/panav-22/Autonomous-Quantitative-Investment-Research-Equity-Synthesis-Engine.git
cd Autonomous-Quantitative-Investment-Research-Equity-Synthesis-Engine
```

### 2. Configure Environment Variables
Create a `.env` file in the root directory:
```env
# AI Brain
OPENAI_API_KEY="your_openai_api_key"
OPENAI_MODEL_NAME="gpt-4o"

# Web Scraping
FIRECRAWL_API_KEY="your_firecrawl_api_key"

# Database Persistence
AZURE_POSTGRES_CONNECTION_STRING="postgresql://[user]:[password]@[host]:5432/[db]?sslmode=require"

# Object Storage
AZURE_BLOB_STORAGE_CONNECTION_STRING="DefaultEndpointsProtocol=https;AccountName=[account];AccountKey=[key];EndpointSuffix=core.windows.net"

# Optional Observability Tracing
LANGCHAIN_TRACING_V2=false
LANGCHAIN_API_KEY="your_langsmith_api_key"
```

---

## Running the Application

To run the complete platform locally, you must launch the backend FastAPI gateway and frontend Streamlit dashboard in separate terminal windows:

### Terminal 1: Launch Backend API
```bash
uv run uvicorn src.api.main:app --reload
```
Ensure the terminal prints: `Application startup complete` and `Uvicorn running on http://127.0.0.1:8000`.

### Terminal 2: Launch Streamlit Portal
```bash
uv run streamlit run frontend/app.py
```
This automatically opens your default browser at `http://localhost:8501`. Enter a ticker like `MSFT` or `NVDA` in the sidebar and click **Run Full Analysis**.
