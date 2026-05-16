# AI Data Explorer — Complete Project Overview

## What Is This Project?

**AI Data Explorer** (branded as **"Nexus"** in the UI) is a full-stack **natural language data querying application**. You upload a data file (CSV, Excel, JSON, TSV) or connect to a MySQL database, then ask questions in plain English. An LLM (Llama 3 via Ollama) translates your question into SQL, executes it, and returns both a human-readable answer and interactive data visualizations.

**GitHub**: https://github.com/sohamkotapally/AI_DataExplorer.git

---

## High-Level Architecture

```mermaid
graph LR
    subgraph Frontend ["Frontend (React + Vite)"]
        UI[React UI :5173]
    end
    subgraph Backend ["Backend (FastAPI)"]
        API[FastAPI Server :8000]
        LLM[LangChain + Ollama Llama3]
        DB[(SQLite / MySQL)]
    end
    UI -- "POST /ask, /upload, /clear-upload" --> API
    API -- "create_sql_query_chain" --> LLM
    LLM -- "Generated SQL" --> API
    API -- "Execute SQL" --> DB
    DB -- "Raw Results" --> API
    API -- "Natural Language + Table Data" --> UI
```

---

## Project Structure

```
AI_DataExplorer/
├── backEnd/
│   ├── main.py              ← Entire backend (single file, 224 lines)
│   └── venv/                ← Python virtual environment
├── frontEnd/
│   ├── index.html           ← HTML entry point
│   ├── vite.config.js       ← Vite configuration
│   ├── package.json         ← NPM dependencies
│   ├── tailwind.config.js   ← TailwindCSS config (present but CSS uses vanilla)
│   ├── postcss.config.js    ← PostCSS config
│   ├── src/
│   │   ├── main.jsx         ← React entry point
│   │   ├── App.jsx          ← Root component (146 lines)
│   │   ├── App.css          ← All styles (1210 lines)
│   │   ├── index.css        ← Minimal reset
│   │   ├── components/
│   │   │   ├── Sidebar.jsx          ← Navigation sidebar (104 lines)
│   │   │   ├── ChatArea.jsx         ← Main chat area + input (131 lines)
│   │   │   ├── MessageBubble.jsx    ← Individual messages (91 lines)
│   │   │   ├── EmptyState.jsx       ← Welcome screen (84 lines)
│   │   │   ├── FileUpload.jsx       ← Drag & drop uploader (137 lines)
│   │   │   └── DataVisualization.jsx← Charts & tables (417 lines)
│   │   └── utils/
│   │       └── chartUtils.js        ← Chart analysis engine (219 lines)
│   └── dist/                ← Production build output
├── diagrams/                ← PlantUML documentation
│   ├── sequence_diagram.puml
│   ├── user_activity_diagram.puml
│   ├── system_architecture.puml
│   ├── usecase_diagram.puml
│   └── class_diagram.puml
├── test_data.csv            ← Sample test data (10 employees)
├── TODO.md                  ← Git push log
├── README.md                ← Minimal readme
└── .gitignore
```

---

## Technology Stack

### Backend
| Technology | Version / Detail | Purpose |
|---|---|---|
| **Python** | 3.x | Runtime |
| **FastAPI** | — | REST API framework |
| **Uvicorn** | — | ASGI server (port 8000) |
| **LangChain** | `langchain`, `langchain_ollama`, `langchain_community` | LLM orchestration, SQL chain generation |
| **Ollama** | Llama 3 model | Local LLM inference (temperature=0) |
| **SQLAlchemy** | via LangChain's `SQLDatabase` | Database abstraction |
| **SQLite** | Built-in | For uploaded file queries |
| **MySQL** | `mysql+mysqlconnector` | Default database (`retail_db` on localhost:3306) |
| **Pandas** | — | File parsing (CSV, Excel, TSV, JSON) |
| **Pydantic** | — | Request validation |

### Frontend
| Technology | Version | Purpose |
|---|---|---|
| **React** | 19.2.0 | UI framework |
| **Vite** | 7.2.4 | Build tool & dev server (port 5173) |
| **Axios** | 1.13.2 | HTTP client for API calls |
| **Framer Motion** | 12.38.0 | Animations & transitions |
| **Recharts** | 3.8.1 | Data visualization (charts) |
| **Lucide React** | 0.562.0 | Icon library |
| **TailwindCSS** | 3.4.17 | Installed but **not actively used** — all styles are in vanilla CSS (`App.css`) |

### Fonts
- **Inter** (weights 300–700) — UI text
- **JetBrains Mono** — Code/SQL blocks and table data

---

## Backend Deep Dive ([main.py](file:///d:/projects/AI_DataExplorer/backEnd/main.py))

### API Endpoints

| Endpoint | Method | Purpose | Request | Response |
|---|---|---|---|---|
| `/upload` | POST | Upload a data file | `multipart/form-data` (file) | `{ filename, rows, columns, column_names, message }` |
| `/upload-status` | GET | Check if a file is uploaded | — | `{ loaded, filename?, rows?, columns?, column_names? }` |
| `/clear-upload` | POST | Remove uploaded file, revert to default DB | — | `{ message }` |
| `/ask` | POST | Ask a natural language question | `{ question: string }` | `{ query, raw, answer, table_data }` |

### How `/ask` Works (The Core Flow)

1. **Determine active database**: If a file has been uploaded → use the SQLite-backed `uploaded_chain`. Otherwise → use the MySQL `retail_db` default chain.
2. **Generate SQL**: `create_sql_query_chain(llm, db).invoke({ question })` sends the question + DB schema to Llama 3, which returns SQL.
3. **Clean the SQL**: 
   - Strip `SQLQuery:` prefix and markdown code fences
   - Run [`clean_sql_quotes()`](file:///d:/projects/AI_DataExplorer/backEnd/main.py#L17-L42) — removes backtick/double-quote wrapping, fixes multi-word aliases (replaces spaces with underscores), preserves function string arguments like `strftime('%Y-%m')`.
4. **Validate**: Reject anything that doesn't start with `SELECT`.
5. **Execute**: Run the cleaned SQL against the active database using SQLAlchemy.
6. **Extract table data**: Re-execute the same SQL to get structured `{ columns, rows }` data (capped at 100 rows).
7. **Summarize**: Send the raw results back to Llama 3 with a prompt to produce a friendly, natural-language sentence.
8. **Return**: `{ query, raw, answer, table_data }`.

### File Upload Pipeline

1. Accept `.csv`, `.xlsx`, `.xls`, `.tsv`, `.json`
2. Parse with Pandas into a DataFrame
3. Sanitize column names (strip spaces, remove parens/hyphens, replace spaces with `_`)
4. Write to a temp SQLite database file (table name: `data`)
5. Create a new LangChain SQL query chain against this SQLite DB
6. Store in global state (`uploaded_db`, `uploaded_chain`, `uploaded_file_info`)

### CORS Configuration
- Only allows `http://localhost:5173` (the Vite dev server)

### Default Database
- MySQL: `mysql+mysqlconnector://root:admin@localhost:3306/retail_db`
- Credentials are **hardcoded** (user: `root`, password: `admin`)

---

## Frontend Deep Dive

### Component Hierarchy

```mermaid
graph TD
    App --> Sidebar
    App --> ChatArea
    ChatArea --> EmptyState
    ChatArea --> MessageBubble
    EmptyState --> FileUpload
    MessageBubble --> DataVisualization
    MessageBubble --> DataTable
```

### [App.jsx](file:///d:/projects/AI_DataExplorer/frontEnd/src/App.jsx) — Root Component

**State Management** (all via `useState`):
- `chats` — Array of chat objects `{ id, title, messages[] }`, persisted to `localStorage` under key `retail-chat-history`
- `activeId` — Currently selected chat ID
- `input` — Current text input value
- `loading` — Whether a query is in progress
- `uploadedFile` — Info about the uploaded file (or null)
- `visualize` — Boolean toggle for chart rendering

**Key Functions**:
- `handleSend(textOverride?)` — Sends a question to `/ask`, appends user message immediately, then appends AI response (with `sql`, `tableData`, and `visualize` flag)
- `startNewChat()` — Creates a fresh chat, prepends to list
- `deleteChat(e, id)` — Removes a chat; creates a fresh one if list becomes empty
- `handleClearFile()` — Calls `/clear-upload` and clears local state

### [Sidebar.jsx](file:///d:/projects/AI_DataExplorer/frontEnd/src/components/Sidebar.jsx)

- **Brand logo** ("Nexus" with Zap icon)
- **New Chat button** (dashed border style)
- **Uploaded file badge** — Shows filename, row/col count, with an ✕ to remove
- **Chat history list** — Animated list of recent chats with active indicator bar and delete button

### [ChatArea.jsx](file:///d:/projects/AI_DataExplorer/frontEnd/src/components/ChatArea.jsx)

- **Chat display**: Shows `EmptyState` when no messages, otherwise renders `MessageBubble` for each message
- **Loading indicator**: Animated dots with glowing AI avatar
- **Input area**: Contains:
  - **Visualize toggle** — Glassmorphism pill button with animated checkbox, enables/disables chart rendering for subsequent queries
  - **Text input** — Placeholder: "Ask Nexus anything about your data..."
  - **Send button** — Gradient accent button, disabled when loading or empty input

### [MessageBubble.jsx](file:///d:/projects/AI_DataExplorer/frontEnd/src/components/MessageBubble.jsx)

Each message shows:
- **Avatar**: Zap icon (AI) or User icon (human)
- **Author label**: "Nexus" or "You"
- **Content**:
  - Error messages → styled error card with `AlertTriangle` icon
  - Normal text → plain message
  - If `tableData` exists AND `visualize === true` → full `DataVisualization` component
  - If `tableData` exists AND `visualize !== true` → `DataTable` only (table view)
  - If `sql` exists → collapsible SQL code block with syntax highlighting

### [EmptyState.jsx](file:///d:/projects/AI_DataExplorer/frontEnd/src/components/EmptyState.jsx)

Welcome screen with:
- Animated hero icon + "Nexus" title with shimmer gradient effect
- Subtitle: "Your AI-powered data companion"
- `FileUpload` component (drag & drop zone)
- Divider: "or try a sample question"
- 4 suggestion cards (Pricing Analysis, Top Rankings, Inventory Stats, Search)

### [FileUpload.jsx](file:///d:/projects/AI_DataExplorer/frontEnd/src/components/FileUpload.jsx)

- Drag & drop zone with visual feedback
- Accepts: `.csv`, `.xlsx`, `.xls`, `.tsv`, `.json`
- 120-second upload timeout
- Three visual states: idle → uploading (spinner) → success/error result
- Posts to `http://localhost:8000/upload` as `multipart/form-data`

### [DataVisualization.jsx](file:///d:/projects/AI_DataExplorer/frontEnd/src/components/DataVisualization.jsx) — The Visualization Engine

This is the most complex component (417 lines). It contains:

**7 Chart Types** (all from Recharts):

| Chart Type | Component | When Used |
|---|---|---|
| Bar | `BarChartView` | Categorical + numeric data (default for most queries) |
| Horizontal Bar | `HorizontalBarView` | When labels are long (avgLength > 12) |
| Line | `LineChartView` | Date/time series data |
| Area | `AreaChartView` | Alternative to line charts |
| Donut/Pie | `PieChartView` | ≤8 categories with ≤8 rows |
| Scatter | `ScatterChartView` | 2+ numeric columns, no categorical |
| Histogram | `HistogramView` | Single numeric column, no categorical |

**Plus**:
- `KpiCard` — Large single-number display (1 row, 1 numeric col)
- `DataTable` — Styled HTML table with sticky headers
- `CustomTooltip` — Glassmorphism-styled chart tooltip

**Chart Type Switcher**: A toolbar at the top lets users toggle between all available chart types for any given result set.

### [chartUtils.js](file:///d:/projects/AI_DataExplorer/frontEnd/src/utils/chartUtils.js) — Data Analysis Engine

| Function | Purpose |
|---|---|
| `analyzeTableData(tableData)` | Classifies each column as `numeric`, `categorical`, or `date`. Detects year-like values (1900–2100) as categorical. |
| `detectChartType(analysis)` | Auto-selects the best chart type based on column types and row count |
| `getAvailableChartTypes(analysis)` | Returns all compatible chart types for the data |
| `transformForChart(tableData, analysis, chartType)` | Transforms raw table data into the format each Recharts component expects |
| `generateHistogramBins(tableData, colIndex, binCount=10)` | Creates frequency distribution bins for histogram rendering |

**Color Palette**: 12 curated colors + 8 gradient pairs in a purple/teal/pink/gold theme.

---

## Design System (from [App.css](file:///d:/projects/AI_DataExplorer/frontEnd/src/App.css))

### Theme
- **Dark mode only** — Background: `#1a1a1a`, Sidebar: `#0f0f0f`
- **Accent**: Purple `#6c5ce7` → Light purple `#a29bfe`
- **Success**: `#00b894` (teal green)
- **Error**: `#ff6b6b` (coral red)

### Design Patterns
- **Glassmorphism**: `backdrop-filter: blur()` on tooltips, viz wrapper, input area
- **Gradient accents**: Linear gradients on buttons, avatars, chart fills
- **Micro-animations**: Framer Motion throughout — spring physics on buttons, staggered list animations, blur transitions between views
- **Custom scrollbar**: Thin 5px track, semi-transparent thumb

### Responsive
- `@media (max-width: 768px)`: Narrower sidebar, single-column suggestion grid, hidden toggle labels

---

## Data Flow — Complete Request Lifecycle

```mermaid
sequenceDiagram
    participant U as User
    participant FE as React Frontend
    participant BE as FastAPI Backend
    participant LLM as Ollama (Llama 3)
    participant DB as SQLite/MySQL

    U->>FE: Types question + clicks Send
    FE->>FE: Append user message to chat
    FE->>BE: POST /ask { question }
    BE->>BE: Select active chain (uploaded vs default)
    BE->>LLM: Generate SQL from question + schema
    LLM-->>BE: Raw SQL response
    BE->>BE: Clean SQL (strip quotes, fix aliases)
    BE->>BE: Validate starts with SELECT
    BE->>DB: Execute cleaned SQL
    DB-->>BE: Raw results + columns/rows
    BE->>LLM: Summarize results in natural language
    LLM-->>BE: Friendly answer string
    BE-->>FE: { query, raw, answer, table_data }
    FE->>FE: Append AI message with tableData & visualize flag
    FE->>U: Render answer + table/charts + SQL block
```

---

## How to Run

### Prerequisites
- **Node.js** (for frontend)
- **Python 3.x** (for backend)
- **Ollama** running locally with the `llama3` model pulled
- **MySQL** (optional — only for default `retail_db`; file uploads use SQLite)

### Backend
```bash
cd backEnd
python -m venv venv
venv\Scripts\activate       # Windows
pip install fastapi uvicorn langchain langchain-ollama langchain-community pandas sqlalchemy mysql-connector-python
python main.py              # Starts on http://localhost:8000
```

### Frontend
```bash
cd frontEnd
npm install
npm run dev                 # Starts on http://localhost:5173
```

---

## Test Data

[test_data.csv](file:///d:/projects/AI_DataExplorer/test_data.csv) — 10 rows, 4 columns:
| name | department | salary | city |
|---|---|---|---|
| Alice | Engineering | 95000 | New York |
| Bob | Marketing | 72000 | San Francisco |
| Charlie | Engineering | 105000 | New York |
| ... | ... | ... | ... |

---

## UML Diagrams

The `diagrams/` folder contains 5 PlantUML diagrams with a "cyborg" theme:
1. **Sequence Diagram** — Query flow between frontend → backend → LLM → DB
2. **User Activity Diagram** — Full user journey from upload to visualization
3. **System Architecture** — Component overview
4. **Use-Case Diagram** — Actor interactions
5. **Class Diagram** — Frontend components + backend services

---

## Known Patterns & Quirks

1. **No authentication** — Open to anyone on localhost
2. **Hardcoded credentials** — MySQL password `admin` in source code
3. **Single-file backend** — Everything in `main.py`, no separation
4. **Global state** — Backend uses global variables for DB connections
5. **No database connection pooling** — New connections on each request
6. **Chat persistence** — Only in browser `localStorage`, not server-side
7. **CORS** — Locked to `http://localhost:5173` only
8. **SQL injection risk** — Relies on LLM output validation (only `SELECT` allowed) but no parameterized queries
9. **TailwindCSS** is installed but the actual styling is entirely vanilla CSS
10. **No environment variables** — All config (ports, DB URI, model name) is hardcoded
