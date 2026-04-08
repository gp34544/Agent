# SAP Datasphere MCP — Complete System Architecture Specification

> Use this document as input to any diagramming tool (draw.io, Lucidchart, Miro, PlantUML, Structurizr, or AI diagram generators) to produce a professional E2E architecture diagram.

---

## 1. HIGH-LEVEL SYSTEM OVERVIEW

```
[User / Developer]
       |
       v
[React Frontend (Vite, port 5173)]
       |
       v
[FastAPI Orchestrator Backend (Uvicorn, port 8000)]
       |
       +---> [Google ADK Agent Framework]
       |         |
       |         +---> [Orchestrator Agent (Claude Sonnet via AWS Bedrock)]
       |         |         |
       |         |         +---> [Builder Agent] ---> [MCP Servers (stdio)]
       |         |         |                              |
       |         |         +---> [Tester Agent]  ---> [MCP Servers (stdio)]
       |         |
       |         +---> [LiteLLM Adapter] ---> [AWS Bedrock] ---> [Claude Sonnet 4]
       |
       +---> [Shared Memory (File-based JSON)]
       +---> [Observer (Audit Log, JSONL)]
       +---> [Test Case Store (File-based JSON)]
       |
       +---> [SAP Datasphere CLI (@sap/datasphere-cli, Node.js)]
       |         |
       |         +---> [SAP Datasphere (HANA Cloud)]
       |
       +---> [SAP S/4HANA Cloud (OData/ADT APIs)]
       +---> [SAP HANA Cloud (Direct SQL via hdbcli)]
```

---

## 2. COMPONENT DETAILS (for diagram boxes)

### 2.1 PRESENTATION LAYER

| Component | Technology | Details |
|-----------|-----------|---------|
| **React Frontend** | React 18 + TypeScript + Vite | Port 5173, SPA with routing |
| Pages: Dashboard | `frontend/src/pages/Dashboard.tsx` | Space overview, object counts, test health |
| Pages: Chat | `frontend/src/pages/Chat.tsx` | Conversational agent UI with SSE streaming |
| Pages: Tests | `frontend/src/pages/Tests.tsx` | 3-tab UI: Test Cases, Executions, Reports |
| Pages: Wizard | `frontend/src/pages/Wizard.tsx` | Step-by-step replication flow creation (no LLM) |
| Pages: ViewBuilder | `frontend/src/pages/ViewBuilder.tsx` | View creation from specs/requirements |
| Pages: Monitor | `frontend/src/pages/Monitor.tsx` | Real-time flow monitoring |
| Pages: Trace | `frontend/src/pages/Trace.tsx` | Audit trail / observer events |
| Pages: Login | `frontend/src/pages/Login.tsx` | User authentication |
| Components: ChatBubble | `frontend/src/components/ChatBubble.tsx` | Agent message rendering |
| Components: ToolCallCard | `frontend/src/components/ToolCallCard.tsx` | MCP tool call visualization |
| Components: GenerateTestsModal | `frontend/src/components/GenerateTestsModal.tsx` | LLM/CSN test generation dialog |
| Components: ManualTestCaseModal | `frontend/src/components/ManualTestCaseModal.tsx` | Manual test case creation |
| Components: TestCaseDetail | `frontend/src/components/TestCaseDetail.tsx` | Test case inspection with artifacts |
| Components: Layout | `frontend/src/components/Layout.tsx` | App shell with navigation |

### 2.2 API LAYER

| Component | Technology | Details |
|-----------|-----------|---------|
| **FastAPI Backend** | FastAPI + Uvicorn | Port 8000, async Python |
| CORS | FastAPI CORSMiddleware | Allows localhost:3000, localhost:5173 |
| Auth Endpoints | `/api/auth/login`, `/api/auth/logout`, `/api/auth/me` | File-based user auth with tokens |
| Session Endpoints | `/api/sessions`, `/api/sessions/{id}/messages` | Agent chat sessions |
| SSE Streaming | `/api/sessions/{id}/stream` | Server-Sent Events with 15s keepalive |
| Trace Endpoints | `/api/trace/{space_id}`, `/api/trace/{space_id}/blame/{object}` | Audit log queries |
| Memory Endpoints | `/api/memory/{space_id}/ledger`, `/api/memory/{space_id}/lineage` | Shared memory reads |
| Test Endpoints | `/api/memory/{space_id}/test-cases`, `/api/memory/{space_id}/test-cases/generate` | CRUD + generation + execution |
| Test Execution | `/api/memory/{space_id}/test-cases/execute`, `/api/memory/{space_id}/test-executions` | Async execution with background tasks |
| Failure Analysis | `/api/memory/{space_id}/test-executions/{run_id}/analysis` | LLM-powered root cause analysis |
| DSP Proxy | `/api/spaces`, `/api/spaces/{id}/views`, `/api/spaces/{id}/views/{name}/csn` | Frontend proxy to Datasphere CLI |
| Direct Wizard | `/api/replication-flow` | No-LLM deterministic pipeline |
| Doc Extraction | `/api/extract-pdf`, `/api/extract-doc` | PDF/DOCX/TXT text extraction (PyPDF2, python-docx) |
| Test Report Export | `/api/memory/{space_id}/test-report/excel` | Excel download (openpyxl) |

### 2.3 AGENT ORCHESTRATION LAYER

| Component | Technology | Details |
|-----------|-----------|---------|
| **Google ADK** | `google-adk` Python SDK | Agent framework with native agent transfer |
| **LiteLLM Adapter** | `litellm` | Routes LLM calls to AWS Bedrock |
| **Orchestrator Agent** | Google ADK `Agent` | Routes user intent to Builder or Tester; does NOT call tools directly |
| **Builder Agent** | Google ADK `Agent` | Creates/manages Datasphere objects; has MCP tools for both DSP + S4 servers |
| **Tester Agent** | Google ADK `Agent` | Validates data quality; has MCP tools for both DSP + S4 servers |
| **ADK Runner** | `google.adk.runners.Runner` | Manages agent execution, session state, tool calls |
| **ADK InMemorySessionService** | `google.adk.sessions.InMemorySessionService` | Per-session conversation memory |
| **Before-Tool Callback** | Python function | Guards: blocks tools with missing space_id, stops tool chaining after errors, injects stored content |
| **Content Store** | Python module | Extracts large payloads (DDL, specs) from messages, sends summary to LLM, injects full content into tool args |
| **Session Manager** | `orchestrator/session_manager.py` | Wraps ADK Runner; records tool events in Observer + Memory; manages agent transfer detection |

### 2.4 LLM / AI LAYER

| Component | Technology | Details |
|-----------|-----------|---------|
| **AWS Bedrock** | AWS Bedrock Runtime API | Managed LLM inference |
| **Claude Sonnet 4** | `us.anthropic.claude-sonnet-4-20250514-v1:0` | Primary LLM for all agents (via LiteLLM → Bedrock) |
| **Claude Opus 4.6** | `us.anthropic.claude-opus-4-6-v1` | Used for test generation (`_call_bedrock_llm` in test_generator.py) |
| **boto3** | AWS SDK for Python | Bedrock API calls |
| **LLM Use Cases** | — | Agent reasoning, view spec generation, test case generation, failure root cause analysis |

### 2.5 MCP SERVER LAYER (Model Context Protocol)

| Component | Technology | Details |
|-----------|-----------|---------|
| **MCP Protocol** | `mcp` Python SDK | stdio-based server/client communication |
| **SAP Datasphere MCP Server** | `mcp_server_dsp/` | 16+ tools for Datasphere operations |
| **SAP S/4HANA MCP Server** | `mcp_server_s4hana/` | 7 tools for S/4HANA operations |
| **McpToolset** | `google.adk.tools.mcp_tool.McpToolset` | ADK adapter wrapping MCP servers as agent tools |
| **StdioConnectionParams** | MCP stdio transport | Each agent gets its own MCP server subprocess |
| **4 MCP Connections Total** | — | Builder: 1x DSP + 1x S4; Tester: 1x DSP + 1x S4 |

#### SAP Datasphere MCP Server Tools (16 tools)
```
Connection:     test_connection, test_cli
Spaces:         list_spaces, create_space
Tables:         create_local_table, read_local_table, list_local_tables
Views:          create_view, read_view, create_view_from_spec, create_bronze_view
Repl Flows:     create_replication_flow, read_replication_flow, deploy_replication_flow
Flow Tasks:     run_replication_flow, replication_flow_status
CDS Parsing:    parse_cds_view
Testing:        generate_tests, execute_approved_tests
Reconciliation: reconcile_data
```

#### SAP S/4HANA MCP Server Tools (7 tools)
```
Connection:     test_s4_connection
Read (LIVE):    list_cds_views, get_cds_metadata, get_cds_record_count, preview_cds_data
Create (LIVE):  create_cds_view
Parse (OFFLINE): parse_cds_ddl, generate_cds_ddl
```

### 2.6 DETERMINISTIC PYTHON LAYER (No LLM)

| Component | File | Details |
|-----------|------|---------|
| **CDS DDL Parser** | `mcp_server_dsp/cds_converter.py` | Regex-based ABAP CDS DDL parser; extracts view name, fields, keys, types, labels, CDC config |
| **Table CSN Generator** | `mcp_server_dsp/cds_converter.py` | Generates Datasphere local table CSN JSON from parsed CDS metadata |
| **Replication Flow Generator** | `mcp_server_dsp/cds_converter.py` | Generates `sap.dis.replicationflow` JSON with vTypes, column mapping, source/target config |
| **Dataflow Generator** | `mcp_server_dsp/cds_converter.py` | Generates `sap.dis.dataflow` JSON (source → projection → target) |
| **Bronze View Generator** | `mcp_server_dsp/cds_converter.py` | Generates 1:1 AV_ passthrough view CSN from local table |
| **Graphical View Generator** | `mcp_server_dsp/view_generator.py` | SQL parser + CSN builder + uiModel builder; produces Transport-ready JSON with graphical editor layout |
| **View Spec Generator** | `mcp_server_dsp/view_spec_generator.py` | End-to-end: fetches metadata, optionally calls LLM for spec, builds CSN + uiModel |
| **CSN Analyzer** | `orchestrator/csn_analyzer.py` | Extracts structured metadata from CSN (fields, joins, keys, layers) for test generation |
| **ABAP Type Mapping** | `mcp_server_dsp/cds_converter.py` | CHAR→cds.String, DEC→cds.Decimal, DATS→cds.Date, etc. (20+ type mappings) |
| **Auto-Fix Engine** | `mcp_server_dsp/datasphere_client.py` | Analyzes CLI validation errors, patches JSON (duplicate columns, missing measures, etc.), retries up to 3x |

### 2.7 SAP DATASPHERE CLI WRAPPER

| Component | Technology | Details |
|-----------|-----------|---------|
| **DatasphereClient** | `mcp_server_dsp/datasphere_client.py` | Python wrapper around `@sap/datasphere-cli` Node.js CLI |
| **OAuth2 Authentication** | Python `requests` | Supports client_credentials, authorization_code (browser flow), refresh tokens |
| **Token Management** | JWT decode + file cache | Auto-refresh before expiry (600s buffer); cached to `.token_cache.json` |
| **CLI Login** | `datasphere login --access-token` | Passes OAuth2 token to Node.js CLI |
| **CLI Execution** | `subprocess.run` | All DSP operations via CLI subprocess; 300s timeout |
| **Windows .cmd Handling** | Custom `_run_process` | Resolves .cmd → node.exe + terminal.js for Windows compatibility |
| **fnm Node Version** | Auto-detection | Prefers fnm-managed Node 20/22/24 over system Node |
| **Auth Retry** | Auto-retry on 401 | Clears token, re-authenticates, re-runs CLI command |
| **Validation Auto-Fix** | `_auto_fix_json()` | Fixes: duplicate columns, missing measures, fact view issues; up to 3 retry cycles |
| **Deploy Workaround** | `read → update` | CLI has no deploy command; deploy = read + update (triggers deploy side-effect) |

### 2.8 PERSISTENCE / MEMORY LAYER

| Component | Storage | Path | Details |
|-----------|---------|------|---------|
| **Shared Memory — Ledger** | JSON file | `memory/spaces/{space}/ledger.json` | Current state of all objects (name, type, status, test_status, created_by) |
| **Shared Memory — Test Results** | JSON file | `memory/spaces/{space}/test_results.json` | Validation history per object |
| **Shared Memory — Lineage** | JSON file | `memory/spaces/{space}/lineage.json` | Requirement → object tracing |
| **Observer — Audit Log** | JSONL file | `memory/spaces/{space}/events.jsonl` | Append-only event log (who, what, when, status) |
| **Test Case Store** | JSON file | `memory/spaces/{space}/test_cases.json` | Test cases with lifecycle: draft → approved/rejected → executed |
| **Test Executions** | JSON file | `memory/spaces/{space}/test_executions.json` | Execution runs with per-case results, artifacts, duration |
| **OAuth Token Cache** | JSON file | `mcp_server_dsp/.token_cache.json` | Cached access/refresh tokens |
| **Claude Code Memory** | Markdown files | `~/.claude/projects/.../memory/` | Cross-conversation agent memory (feedback, project, user, reference types) |

### 2.9 DATA RECONCILIATION LAYER

| Component | Technology | Details |
|-----------|-----------|---------|
| **HanaConnector** | `hdbcli` (SAP HANA Python driver) | Direct SQL to HANA Cloud via Database Access user |
| **Schema Check** | Python | Compares source CDS fields vs target table CSN elements |
| **Row Count Check** | SQL `SELECT COUNT(*)` | Source count (from S/4HANA) vs target count (from HANA) |
| **Sample Data Check** | SQL `SELECT TOP N` | Row-by-row comparison of first N rows |
| **Checksum Check** | SQL `SUM()` | Numeric column checksums: source vs target |
| **Replication Status Check** | CLI `tasks replication-flows status` | Flow completed successfully? |
| **Retry Logic** | Python | Up to 3 retries on transient HANA connection errors |
| **PDF Report** | `fpdf2` + `matplotlib` | Professional PDF with charts, scorecard, tables |

### 2.10 TEST PIPELINE

| Component | Technology | Details |
|-----------|-----------|---------|
| **Test Generation (CSN-only)** | Python deterministic | NULL checks, JOIN integrity, row count, key uniqueness from CSN structure |
| **Test Generation (LLM)** | Claude via AWS Bedrock | Business-logic tests from spec + CSN; returns structured JSON with SQL scripts |
| **Test Case Lifecycle** | State machine | `draft` → `approved` (human) → `executed` → `pass`/`fail`/`error` |
| **Human-in-the-Loop** | Tests UI + API | Human reviews generated cases, approves/rejects individually or in bulk |
| **Test Execution** | hdbcli SQL | Runs SQL scripts against HANA Cloud; captures artifacts (columns, rows, duration) |
| **Result Evaluation** | Python | Pattern matching: "= 0", "> 0", exact match; determines pass/fail |
| **Failure Analysis** | Claude via AWS Bedrock | LLM explains root cause + suggests fixes for each failed test |
| **Simulation Mode** | Python fallback | When hdbcli not installed, returns simulated pass results |
| **Excel Export** | openpyxl | Downloadable test report with all cases + execution results |

---

## 3. EXTERNAL SYSTEMS & INTEGRATIONS

### 3.1 SAP Systems

| System | Connection | Protocol | Auth |
|--------|-----------|----------|------|
| **SAP Datasphere** | Via `@sap/datasphere-cli` (Node.js) | CLI → REST API (internal) | OAuth2 (client_credentials or authorization_code) |
| **SAP HANA Cloud** | Direct SQL via `hdbcli` driver | TCP/443 (encrypted) | Database Access user (username/password) |
| **SAP S/4HANA Cloud** | OData v2/v4 + ADT REST APIs | HTTPS | Basic Auth or OAuth2 |
| **SAP BTP Cockpit** | OAuth2 service key source | — | Service key JSON provides token_url, client_id, client_secret |

### 3.2 AWS Services

| Service | Purpose | Details |
|---------|---------|---------|
| **AWS Bedrock** | LLM inference | Claude Sonnet 4 for agents; Claude Opus 4.6 for test generation |
| **AWS IAM** | Authentication | AWS_ACCESS_KEY_ID + AWS_SECRET_ACCESS_KEY for Bedrock access |
| **Region** | us-east-1 (default) | Configurable via AWS_DEFAULT_REGION |

### 3.3 Node.js Dependency

| Component | Version | Purpose |
|-----------|---------|---------|
| **Node.js** | 20-24 (via fnm or system) | Runtime for @sap/datasphere-cli |
| **@sap/datasphere-cli** | npm global install | All Datasphere CRUD operations |

---

## 4. DATA FLOW DIAGRAMS

### 4.1 Replication Flow Creation (E2E)

```
User provides CDS DDL text + space + connection name
    |
    v
[1. CDS DDL Parser] --- Python regex, no LLM
    | extracts: view_name, fields, keys, types, labels, CDC
    v
[2. Table CSN Generator] --- all cds.String for replication
    | generates: local table JSON definition
    v
[3. Datasphere CLI: create local-table] --- subprocess → Node.js CLI → DSP REST API
    | creates table in Datasphere space
    v
[4. Replication Flow JSON Generator] --- vTypes, column mapping, source/target
    | generates: replication flow JSON
    v
[5. Datasphere CLI: create replication-flow] --- with -N (no deploy)
    | saves flow (not deployed)
    v
[6. User confirms deploy]
    v
[7. Datasphere CLI: update replication-flow] --- triggers deploy
    v
[8. User confirms run]
    v
[9. Datasphere CLI: tasks run] --- starts data load
    v
[10. Poll status until COMPLETED]
    v
[11. Bronze View Generator] --- AV_ passthrough view
    v
[12. Datasphere CLI: create view]
    v
[13. Test Generation] --- CSN analysis + optional LLM
    v
[14. Human approves tests]
    v
[15. Test Execution] --- SQL against HANA Cloud
    v
[16. Results + Failure Analysis]
```

### 4.2 Agent Message Flow

```
[React Frontend]
    | HTTP POST /api/sessions/{id}/stream (SSE)
    v
[FastAPI] → [SessionManager]
    | extract_and_summarize(message) — stores large content separately
    v
[ADK Runner.run_async()]
    | sends to Orchestrator Agent
    v
[Orchestrator Agent (Claude Sonnet via Bedrock)]
    | analyzes intent → transfers to sub-agent
    v
[Builder Agent] or [Tester Agent]
    |
    +---> [before_tool_callback] — validates args, injects content
    |
    +---> [MCP Tool Call via stdio] → [MCP Server subprocess]
    |         |
    |         +---> [Datasphere CLI] → [SAP Datasphere]
    |         +---> [S/4HANA OData] → [SAP S/4HANA]
    |         +---> [hdbcli SQL]    → [SAP HANA Cloud]
    |
    +---> [Tool Result]
    |         |
    |         +---> [Observer.record()] → events.jsonl
    |         +---> [SharedMemory.register_object()] → ledger.json
    |
    v
[SSE Events streamed to Frontend]
    types: status, agent, text, tool_call, tool_result, done
```

### 4.3 Test Pipeline Flow

```
[Generate Tests]
    |
    +---> [read_view CSN from Datasphere]
    +---> [csn_analyzer.py: extract fields, joins, keys, layer]
    +---> (if business_spec) [Claude via Bedrock: generate test scenarios]
    +---> (if no spec) [Python: structural tests only]
    |
    v
[Test Cases stored as DRAFT] → test_cases.json
    |
    v
[Human reviews in Tests UI]
    | approve / reject per case or bulk
    v
[Execute Approved Tests]
    |
    +---> [hdbcli: run SQL against HANA Cloud]
    +---> [evaluate: pass/fail based on expected vs actual]
    +---> [store results + artifacts] → test_executions.json
    |
    v
[If failures exist]
    |
    +---> [Claude via Bedrock: analyze root causes]
    +---> [Return: root_cause + suggestion + severity per failure]
    v
[Results displayed in Tests UI]
    +---> [Excel export available]
```

---

## 5. LAYER ARCHITECTURE (Data Modeling)

```
┌─────────────────────────────────────────────────────────────┐
│  GOLD LAYER (Consumption)                                    │
│  DIM_Customer, FACT_Sales                                    │
│  Star schema: dimensions + facts for BI/analytics            │
├─────────────────────────────────────────────────────────────┤
│  SILVER LAYER (Business Logic)                               │
│  CV_SalesByRegion, CV_OrderSummary                           │
│  Joins, calculations, filters, aggregations                  │
├─────────────────────────────────────────────────────────────┤
│  BRONZE LAYER (Access Views)                                 │
│  AV_I_SalesOrder, AV_I_Product                               │
│  1:1 passthrough views with labels on replicated tables      │
├─────────────────────────────────────────────────────────────┤
│  REPLICATION LAYER (Raw Landing)                             │
│  T_I_SalesOrder, T_I_Product                                 │
│  Local tables populated by Replication Flows (RF_)           │
│  All columns as cds.String (CDS extraction sends strings)    │
├─────────────────────────────────────────────────────────────┤
│  SOURCE: SAP S/4HANA Cloud                                   │
│  CDS Views: I_SalesOrder, I_Product, Z_I_T001_CDC           │
│  Connected via S/4HANA connection in Datasphere              │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. DEPLOYMENT TOPOLOGY

```
┌─────────────────────────────────────────────────┐
│  Developer Workstation (Windows 11)              │
│                                                  │
│  ┌──────────────┐  ┌─────────────────────────┐  │
│  │ React Dev    │  │ FastAPI + Uvicorn        │  │
│  │ Server       │  │ (port 8000)             │  │
│  │ (port 5173)  │  │                         │  │
│  │              │  │ ┌─────────────────────┐ │  │
│  │ Vite HMR     │  │ │ Google ADK Runner   │ │  │
│  │              │  │ │                     │ │  │
│  └──────┬───────┘  │ │ Orchestrator Agent  │ │  │
│         │ HTTP     │ │   ├─ Builder Agent  │ │  │
│         │          │ │   └─ Tester Agent   │ │  │
│         └─────────►│ └─────────┬───────────┘ │  │
│                    │           │ stdio        │  │
│                    │ ┌─────────▼───────────┐ │  │
│                    │ │ MCP Server Process 1 │ │  │
│                    │ │ (python -m           │ │  │
│                    │ │  mcp_server_dsp)     │ │  │
│                    │ ├─────────────────────┤ │  │
│                    │ │ MCP Server Process 2 │ │  │
│                    │ │ (python -m           │ │  │
│                    │ │  mcp_server_s4hana)  │ │  │
│                    │ └─────────┬───────────┘ │  │
│                    │           │ subprocess   │  │
│                    │ ┌─────────▼───────────┐ │  │
│                    │ │ Node.js CLI Process  │ │  │
│                    │ │ (@sap/datasphere-cli)│ │  │
│                    │ └─────────────────────┘ │  │
│                    └─────────────────────────┘  │
│                                                  │
│  ┌──────────────────────────┐                    │
│  │ File-based Persistence   │                    │
│  │ memory/spaces/{space}/   │                    │
│  │  ├─ ledger.json          │                    │
│  │  ├─ test_cases.json      │                    │
│  │  ├─ test_executions.json │                    │
│  │  ├─ test_results.json    │                    │
│  │  ├─ lineage.json         │                    │
│  │  └─ events.jsonl         │                    │
│  └──────────────────────────┘                    │
└──────────────────────┬───────────────────────────┘
                       │ HTTPS (outbound)
          ┌────────────┼────────────────┐
          │            │                │
          v            v                v
┌─────────────┐ ┌────────────┐ ┌──────────────┐
│ SAP         │ │ SAP S/4HANA│ │ AWS Bedrock  │
│ Datasphere  │ │ Cloud      │ │ (us-east-1)  │
│ (HANA Cloud)│ │            │ │              │
│             │ │ OData v2/v4│ │ Claude       │
│ CLI → REST  │ │ ADT REST   │ │ Sonnet 4     │
│ hdbcli → SQL│ │ Basic Auth │ │ Opus 4.6     │
│ OAuth2      │ │ or OAuth2  │ │              │
└─────────────┘ └────────────┘ └──────────────┘
```

---

## 7. TECHNOLOGY STACK SUMMARY

### Backend (Python 3.11+)
| Package | Version | Purpose |
|---------|---------|---------|
| `fastapi` | >=0.115.0 | REST API framework |
| `uvicorn` | >=0.30.0 | ASGI server |
| `google-adk` | >=1.0.0 | Agent Development Kit (multi-agent orchestration) |
| `litellm` | >=1.0.0 | LLM routing adapter (Bedrock, OpenAI, etc.) |
| `mcp` | >=1.0.0 | Model Context Protocol SDK |
| `boto3` | >=1.34.0 | AWS SDK (Bedrock API calls) |
| `requests` | >=2.31.0 | HTTP client (OAuth2, S/4HANA OData) |
| `hdbcli` | >=2.20.0 | SAP HANA Cloud Python driver |
| `fpdf2` | >=2.8.0 | PDF report generation |
| `matplotlib` | >=3.9.0 | Charts in reconciliation reports |
| `PyPDF2` | >=3.0.0 | PDF text extraction |
| `sqlglot` | >=26.0.0 | SQL parsing utilities |
| `pydantic` | (FastAPI dep) | Request/response validation |
| `python-dotenv` | (implicit) | .env file loading |

### Frontend (Node.js)
| Package | Purpose |
|---------|---------|
| React 18 | UI framework |
| TypeScript | Type safety |
| Vite | Build tool + dev server |
| React Router | SPA routing |

### Infrastructure
| Component | Purpose |
|-----------|---------|
| Node.js 20-24 | Runtime for @sap/datasphere-cli |
| @sap/datasphere-cli | Official SAP CLI for all Datasphere operations |
| fnm (optional) | Node version manager (Windows) |

---

## 8. ENVIRONMENT VARIABLES

### AWS Bedrock
```
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
AWS_DEFAULT_REGION=us-east-1
BEDROCK_MODEL_ID=us.anthropic.claude-sonnet-4-20250514-v1:0
```

### SAP Datasphere
```
DSP_HOST=https://<tenant>.us10.hcs.cloud.sap/
DSP_TOKEN_URL=https://<tenant>.authentication.us10.hana.ondemand.com/oauth/token
DSP_CLIENT_ID=sb-<uuid>|client!b655
DSP_CLIENT_SECRET=<secret>
DSP_AUTH_TYPE=client_credentials | authorization_code
DSP_AUTH_URL=https://<tenant>.authentication.us10.hana.ondemand.com
```

### SAP HANA Cloud (Direct SQL)
```
DSP_HANA_HOST=<host>.hanacloud.ondemand.com
DSP_HANA_PORT=443
DSP_HANA_USER=<database_access_user>
DSP_HANA_PASSWORD=<password>
```

### SAP S/4HANA
```
S4_HOST=https://<s4-host>:<port>/
S4_AUTH_TYPE=basic | oauth2
S4_USERNAME=...
S4_PASSWORD=...
S4_TOKEN_URL=...     (for OAuth2)
S4_CLIENT_ID=...     (for OAuth2)
S4_CLIENT_SECRET=... (for OAuth2)
```

---

## 9. SECURITY BOUNDARIES

```
┌─────────────────────────────────────────────────────┐
│  TRUST ZONE: Local Workstation                       │
│                                                      │
│  [Frontend] ←→ [FastAPI] ←→ [MCP Servers] ←→ [CLI] │
│                                                      │
│  File-based persistence (no encryption at rest)      │
│  Token cache in plaintext (.token_cache.json)        │
│  Credentials in .env / .mcp.json                     │
└──────────────────────┬───────────────────────────────┘
                       │ TLS (outbound only)
                       │ NOTE: SSL verify=False currently
          ┌────────────┼────────────────┐
          v            v                v
  [SAP Datasphere] [S/4HANA]    [AWS Bedrock]
  OAuth2 tokens    Basic/OAuth2  IAM credentials
  CLI manages      Session auth  SigV4 signing
```

---

## 10. DIAGRAM LABELS & ANNOTATIONS

Use these labels on the diagram arrows/connections:

| From | To | Label | Protocol |
|------|----|-------|----------|
| Frontend | FastAPI | REST + SSE | HTTP |
| FastAPI | ADK Runner | Python async | In-process |
| ADK Runner | Orchestrator Agent | Agent execution | In-process |
| Orchestrator | Builder/Tester | Agent transfer | ADK native |
| Builder/Tester | MCP Server | Tool calls | stdio (JSON-RPC) |
| MCP DSP Server | Datasphere CLI | CLI commands | subprocess |
| Datasphere CLI | SAP Datasphere | CRUD operations | HTTPS (REST) |
| MCP DSP Server | HANA Cloud | SQL queries | TCP/443 (hdbcli) |
| MCP S4 Server | S/4HANA | CDS view read/create | HTTPS (OData/ADT) |
| FastAPI | AWS Bedrock | LLM inference | HTTPS (boto3) |
| FastAPI | File System | Persistence | File I/O |
| Agents | Shared Memory | Object ledger, lineage | File I/O (via Python) |
| Agents | Observer | Audit events | File I/O (JSONL append) |
| Agents | Test Store | Test cases + executions | File I/O |

---

## 11. COLOR CODING SUGGESTION

| Color | Components |
|-------|-----------|
| **Blue** | SAP systems (Datasphere, S/4HANA, HANA Cloud, BTP) |
| **Orange** | AWS services (Bedrock, IAM) |
| **Green** | Agent layer (Orchestrator, Builder, Tester, ADK, LiteLLM) |
| **Purple** | MCP layer (MCP servers, stdio transport, tool definitions) |
| **Gray** | Deterministic Python (CDS parser, CSN generators, reconciliation) |
| **Teal** | Frontend (React, Vite) |
| **Yellow** | Persistence (file-based memory, observer, test store) |
| **Red** | Security boundaries |

---

## 12. ICON SUGGESTIONS

| Component | Icon/Logo |
|-----------|-----------|
| AWS Bedrock | AWS Bedrock service icon |
| Claude | Anthropic logo |
| SAP Datasphere | SAP Datasphere product icon |
| SAP S/4HANA | SAP S/4HANA product icon |
| SAP HANA Cloud | SAP HANA icon |
| SAP BTP | SAP BTP icon |
| FastAPI | FastAPI lightning bolt logo |
| React | React atom logo |
| Python | Python logo |
| Node.js | Node.js logo |
| Google ADK | Google Cloud AI icon or generic agent icon |
| MCP | Anthropic MCP icon or plug/socket icon |
| File Storage | Folder/disk icon |
| OAuth2 | Lock/key icon |
| SSE Streaming | Arrow stream icon |
