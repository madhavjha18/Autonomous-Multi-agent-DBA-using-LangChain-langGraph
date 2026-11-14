<div align="center">

<h1>🤖 Autonomous Multi-Agent DBA</h1>

<p><strong>Talk to your MySQL database in plain English. Safely.</strong></p>

<p>
  <img src="https://img.shields.io/badge/Python-3.10%2B-blue?style=for-the-badge&logo=python&logoColor=white"/>
  <img src="https://img.shields.io/badge/LangGraph-Stateful%20Graph-purple?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Groq-LLaMA%203-orange?style=for-the-badge&logo=groq"/>
  <img src="https://img.shields.io/badge/MySQL-Database-4479A1?style=for-the-badge&logo=mysql&logoColor=white"/>
  <img src="https://img.shields.io/badge/SQLAlchemy-ORM-red?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/License-MIT-green?style=for-the-badge"/>
</p>

<p>
  <a href="#-overview">Overview</a> •
  <a href="#-architecture">Architecture</a> •
  <a href="#-features">Features</a> •
  <a href="#-quick-start">Quick Start</a> •
  <a href="#-how-it-works">How It Works</a> •
  <a href="#-security-model">Security</a> •
  <a href="#-sample-queries">Sample Queries</a>
</p>

</div>

---

## 📌 Overview

**Autonomous Multi-Agent DBA** is a production-grade, supervisor-routed multi-agent system that lets you interact with a MySQL database using plain English — safely, intelligently, and autonomously.

Instead of a single LLM juggling every type of database task, this system delegates work to **three specialized sub-agents**, each compiled as an independent LangGraph subgraph with its own tools, loop caps, and privilege scope. A central **Supervisor LLM** (powered by Groq's ultra-fast inference) reads your intent and routes it to the right specialist — just like dispatching a ticket to the right engineer.

> Built with LangGraph · LangChain · Groq API · MySQL · SQLAlchemy · Python 3.10+

---

## 🏛️ Architecture

```
  ┌─────────────────────────────────────────────────────────────┐
  │                       User Request                          │
  │            "Show total sales by region last month"          │
  └────────────────────────────┬────────────────────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │   🛡️  PII Redactor   │  ← Masks emails, phones,
                    │   (Middleware)       │    card numbers BEFORE LLM
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  🤖  Supervisor LLM  │  ← Groq (LLaMA 3)
                    │  (Router Node)       │    Reads intent → routes
                    └──┬───────┬───────┬──┘
                       │       │       │
           ┌───────────┘   ┌───┘   └───────────┐
           ▼               ▼                   ▼
  ┌────────────────┐ ┌──────────────┐ ┌────────────────────┐
  │ 📊 SQL Analyst │ │ ⚡ Optimizer  │ │ 🔧 Schema Admin     │
  │  (Subgraph)    │ │  (Subgraph)  │ │  (Subgraph)        │
  │                │ │              │ │                    │
  │ • NL → SQL     │ │ • EXPLAIN    │ │ • ALTER / CREATE   │
  │ • SELECT only  │ │ • Index tips │ │ • Human approval ⏸ │
  │ • Self-correct │ │ • Rewrites   │ │ • DROP = blocked   │
  │ • Loop cap: N  │ │ • Loop cap:N │ │ • Loop cap: N      │
  └───────┬────────┘ └──────┬───────┘ └────────┬───────────┘
          │                 │                  │
          └─────────────────┴──────────────────┘
                            │
                 ┌──────────▼──────────┐
                 │   🗄️  MySQL via      │
                 │   SQLAlchemy ORM    │
                 │   (Connection Pool) │
                 └─────────────────────┘

  ══════════════════════════════════════════════════════════════
  📋 Shared Graph State  →  messages | next_agent | loop_count
  ══════════════════════════════════════════════════════════════
```

---

## ✨ Features

### 🤖 Multi-Agent Orchestration
| Feature | Description |
|---|---|
| **Supervisor Routing** | A central LLM node reads intent and dynamically delegates to the correct specialist using structured tool calls, not hardcoded if/else |
| **Independent Subgraphs** | Each agent is a fully compiled `StateGraph` — encapsulated tools, internal ReAct loop, and loop cap |
| **Conditional Edges** | LangGraph edges read the state's `next_agent` field — the LLM's own decision controls routing and termination |
| **FINISH Signal** | Supervisor emits `FINISH` when done; the graph terminates cleanly via the `END` node |

### 🧠 Stateful Context Management
| Feature | Description |
|---|---|
| **Typed Shared State** | `GraphState` TypedDict carries `messages`, `next_agent`, `status`, and `loop_count` across every node |
| **Message Accumulation** | `add_messages` reducer appends to history — never overwrites — giving the LLM full multi-turn context |
| **Loop Cap Safety** | Each agent tracks `loop_count`; when it hits the cap the graph hard-stops, preventing runaway LLM loops and token blow-up |
| **Result Truncation** | Large SQL result sets are capped at N rows; a note is appended to protect the context window |

### 🔐 Privilege Separation & Human-in-the-Loop
| Agent | Allowed | Blocked |
|---|---|---|
| **SQL Analyst** | `SELECT` | Everything else — keyword blocked before execution |
| **Query Optimizer** | `SELECT`, `EXPLAIN ANALYZE` | All writes |
| **Schema Admin** | `ALTER`, `CREATE` with approval | `DROP DATABASE`, `TRUNCATE` — permanently blocked |

- **Human-in-the-Loop (HITL)**: All DDL from the Schema Admin triggers `langgraph.types.interrupt()` — the graph pauses, shows the human a preview, and only resumes on explicit `"approve"` input
- **Keyword Blocklist**: SQL Analyst raises a `PermissionError` before any write keyword reaches the database engine — a defense-in-depth layer independent of DB-level permissions

### 🛡️ PII Redaction & Self-Correction
| Feature | Description |
|---|---|
| **PII Middleware** | Regex-based redactor runs on every message *before* it reaches Groq — masks emails → `[EMAIL]`, phones → `[PHONE]`, card patterns → `[CARD]` |
| **Error-Driven Self-Correction** | On SQL failure, the exact error message is injected into the message history as a `ToolMessage`; the LLM reads its own mistake and autonomously rewrites the query — no hardcoded retry loops |

---

## 📁 Project Structure

```
autonomous-multi-agent-dba/
├── src/
│   ├── state.py                 # GraphState TypedDict — shared across all nodes
│   ├── supervisor.py            # Supervisor LLM node + routing logic
│   ├── middleware/
│   │   └── pii_redactor.py      # Regex PII masking (email, phone, card)
│   ├── subgraphs/
│   │   ├── sql_analyst.py       # SELECT-only agent + self-correction loop
│   │   ├── optimizer.py         # EXPLAIN ANALYZE + index suggestion agent
│   │   └── schema_admin.py      # DDL agent + human-in-the-loop interrupt
│   ├── tools/
│   │   ├── db_connector.py      # SQLAlchemy MySQL connection pool
│   │   ├── sql_tools.py         # execute_select, describe_table, list_tables
│   │   ├── optimizer_tools.py   # explain_query, suggest_index
│   │   └── ddl_tools.py         # preview_ddl, apply_ddl (gated)
│   └── graph.py                 # Assembles the full supervisor graph
├── main.py                      # Entry point — CLI interaction loop
├── setup.py                     # Initializes MySQL schema + seed data
├── .env.example                 # Environment variables template
├── requirements.txt
└── README.md
```

---

## 🚀 Quick Start

### Prerequisites

- Python 3.10+
- A running MySQL instance (local or cloud)
- A [Groq API key](https://console.groq.com/) (free tier available)

### 1. Clone & Install

```bash
git clone https://github.com/your-username/autonomous-multi-agent-dba.git
cd autonomous-multi-agent-dba
python -m venv venv && source venv/bin/activate   # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

### 2. Configure Environment

```bash
cp .env.example .env
```

Open `.env` and fill in your values:

```env
# Groq API
GROQ_API_KEY=your_groq_api_key_here
GROQ_MODEL=llama-3.3-70b-versatile

# MySQL Connection
DB_HOST=localhost
DB_PORT=3306
DB_NAME=your_database
DB_USER=your_username
DB_PASSWORD=your_password

# Safety Settings
MAX_LOOP_COUNT=10
MAX_RESULT_ROWS=100
```

### 3. Set Up the Database

```bash
python setup.py
```

This creates the schema and populates sample data for testing.

### 4. Run the Agent

```bash
python main.py
```

---

## 🎮 How It Works

### The State Machine

Every user request flows through a typed, shared `GraphState`:

```python
from typing import Annotated, TypedDict
from langgraph.graph.message import add_messages

class GraphState(TypedDict):
    messages:   Annotated[list, add_messages]  # Full conversation history
    next_agent: str                             # Routing signal for edges
    status:     str                             # "running" | "waiting" | "done"
    loop_count: int                             # Safety fuse — stops runaway loops
```

### Step-by-Step Flow

```
1. User input arrives
       ↓
2. PII Redactor masks sensitive data (emails, phones, cards)
       ↓
3. Supervisor LLM analyses sanitized intent
       ↓
4. Conditional edge routes to the correct subgraph:
   • "Show me users who..." → SQL Analyst
   • "Why is this query slow?" → Query Optimizer
   • "Add a column to users" → Schema Admin (+ human gate)
       ↓
5. Agent runs its internal ReAct loop (think → tool call → observe)
       ↓
6a. SUCCESS: Result returned, Supervisor emits FINISH → END
6b. FAILURE: SQL error injected into messages → agent self-corrects
```

### Supervisor Routing (Simplified)

```python
class RouteDecision(BaseModel):
    next: Literal["sql_analyst", "optimizer", "schema_admin", "FINISH"]

def supervisor_node(state: GraphState) -> GraphState:
    response = llm.with_structured_output(RouteDecision).invoke(state["messages"])
    return {"next_agent": response.next}

graph.add_conditional_edges("supervisor", lambda s: s["next_agent"], {
    "sql_analyst":  "sql_analyst_subgraph",
    "optimizer":    "optimizer_subgraph",
    "schema_admin": "schema_admin_subgraph",
    "FINISH":        END,
})
```

### Human-in-the-Loop DDL Gate

```python
from langgraph.types import interrupt

def apply_ddl_tool(sql: str):
    preview = format_ddl_preview(sql)
    # Graph pauses here — human must respond before execution continues
    approval = interrupt({"preview": preview, "prompt": "Type 'approve' to execute:"})
    if approval.lower() != "approve":
        return "DDL aborted by user."
    return execute_ddl(sql)
```

---

## 🔐 Security Model

### Privilege Separation

The system enforces two independent layers of protection — both must pass:

```
Layer 1: Keyword Blocklist (code-level)
   → SQL Analyst raises PermissionError for any non-SELECT keyword
   → Schema Admin permanently rejects DROP DATABASE and TRUNCATE

Layer 2: DB-Level Permissions (MySQL user)
   → Analyst DB user: GRANT SELECT ON database.* TO 'analyst_user'
   → Admin DB user:   GRANT ALTER, CREATE ON database.* TO 'admin_user'
```

### PII Redaction

```python
PII_PATTERNS = {
    "EMAIL": r"[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}",
    "PHONE": r"(\+?\d[\d\-\s]{7,}\d)",
    "CARD":  r"\b(?:\d[ -]?){13,16}\b",
}

# Applied as a LangChain runnable — wraps every LLM call
pii_safe_llm = RunnableLambda(redact_messages) | llm
```

> Real PII **never reaches the Groq API**. Masking happens before every external call.

---

## 🎯 Sample Queries

### SQL Analyst — Read-Only Queries

```
You: Show me total revenue grouped by product category for last month
You: Which users signed up in the last 7 days?
You: List the top 5 customers by total order value
You: How many orders are in "pending" status?
```

### Query Optimizer — Performance Analysis

```
You: Why is this query slow: SELECT * FROM orders WHERE created_at > '2024-01-01'
You: What indexes should I add to the users table?
You: Show me the execution plan for fetching all orders with joins
```

### Schema Admin — Structural Changes (Requires Approval)

```
You: Add a "phone_number" column to the users table
You: Create an index on orders.status
You: Rename the column "usr_name" to "username" in users

[System]: Preview: ALTER TABLE users ADD COLUMN phone_number VARCHAR(20);
[System]: Type 'approve' to execute or anything else to cancel:
You: approve
[System]: ✅ Schema updated successfully.
```

### Off-Topic (Handled Gracefully)

```
You: What's the weather today?
Agent: I'm your database assistant — I only speak SQL! Ask me about your data. 😄
```

---

## ⚙️ Configuration Reference

| Variable | Default | Description |
|---|---|---|
| `GROQ_API_KEY` | — | Your Groq API key (required) |
| `GROQ_MODEL` | `llama-3.3-70b-versatile` | Groq model to use |
| `DB_HOST` | `localhost` | MySQL host |
| `DB_PORT` | `3306` | MySQL port |
| `DB_NAME` | — | Target database name |
| `DB_USER` | — | MySQL username |
| `DB_PASSWORD` | — | MySQL password |
| `MAX_LOOP_COUNT` | `10` | Max agent iterations before hard-stop |
| `MAX_RESULT_ROWS` | `100` | Max rows returned per query (truncation cap) |

### Swapping the LLM

The system is LLM-agnostic. Any LangChain-compatible model works:

```python
# Groq (default — fastest)
from langchain_groq import ChatGroq
llm = ChatGroq(model="llama-3.3-70b-versatile", temperature=0)

# OpenAI
from langchain_openai import ChatOpenAI
llm = ChatOpenAI(model="gpt-4o", temperature=0)

# Local (Ollama)
from langchain_ollama import ChatOllama
llm = ChatOllama(model="llama3.2:latest", temperature=0)
```

### Switching the Database

SQLAlchemy supports any SQL database — just change the connection string:

```python
# PostgreSQL
DATABASE_URL = "postgresql+psycopg2://user:password@host:5432/dbname"

# SQLite (for local dev/testing)
DATABASE_URL = "sqlite:///./local.db"
```

---

## 🛠️ Requirements

```txt
langgraph>=0.2.0
langchain>=0.3.0
langchain-core>=0.3.0
langchain-community>=0.3.0
langchain-groq>=0.2.0
sqlalchemy>=2.0.0
pymysql>=1.1.0
pydantic>=2.0.0
python-dotenv>=1.0.0
cryptography>=41.0.0
```

Install all at once:

```bash
pip install -r requirements.txt
```

---

## 🔍 Troubleshooting

**Groq API errors**
> Verify your `GROQ_API_KEY` in `.env` and check your rate limits at [console.groq.com](https://console.groq.com).

**MySQL connection refused**
> Confirm MySQL is running (`mysql -u root -p`), and that `DB_HOST`, `DB_PORT`, `DB_USER`, and `DB_PASSWORD` in `.env` are correct.

**Agent loops without finishing**
> Lower `MAX_LOOP_COUNT` in `.env`. If the agent consistently fails, refine the system prompt in `supervisor.py` to give the LLM clearer schema context.

**PII not being masked**
> Extend `PII_PATTERNS` in `src/middleware/pii_redactor.py` with additional regex patterns for your data format.

**DDL changes not requiring approval**
> Ensure your `apply_ddl_tool` function uses `interrupt()` from `langgraph.types`. If the graph is compiled without a `checkpointer`, interrupts are silently skipped.

---

## 🤝 Contributing

Contributions are welcome! Here's how to get involved:

1. **Fork** the repository and create your branch: `git checkout -b feature/my-feature`
2. **Commit** your changes: `git commit -m 'Add: my feature description'`
3. **Push** and open a **Pull Request**

Suggested contributions:
- 📊 Streamlit / Gradio UI wrapper
- 🔌 Additional agent types (Backup Agent, Monitoring Agent)
- 🌐 REST API layer via FastAPI
- 🧪 Test suite for each subgraph
- 📈 LangSmith tracing integration

---

## 📄 License

This project is licensed under the **MIT License** — see the [LICENSE](LICENSE) file for details.

---

## 🙏 Acknowledgments

- **[LangChain](https://www.langchain.com/) & [LangGraph](https://langchain-ai.github.io/langgraph/)** — for the agent orchestration framework
- **[Groq](https://groq.com/)** — for blazing-fast LLM inference that makes real-time agentic loops practical
- **[SQLAlchemy](https://www.sqlalchemy.org/)** — for robust, dialect-agnostic database access

---

<div align="center">

**⭐ Star this repo if it helped you — it motivates continued development!**

*Built with 🧠 LangGraph · ⚡ Groq · 🗄️ MySQL · 🐍 Python*

</div>
