# Data Codex Roadmap

## Phase 1: Electron Shell + Embedded Database Foundation
**Status**: Not Started
**Goal**: Establish the core Electron app with all three embedded databases wired together and communicating via Apache Arrow
**Deliverables**:
- Electron app scaffold (main process, renderer process, preload scripts)
- DuckDB Node.js integration in main process with Worker Thread isolation
- LanceDB Node.js SDK integration in main process with Worker Thread isolation
- RxDB setup with SQLite storage adapter for application state
- Apache Arrow IPC layer using MessagePort (main-to-renderer zero-copy transfer)
- Proof-of-concept: load a Parquet file into DuckDB, query it, send Arrow result to renderer, display in console

---

## Phase 2: Local Data Ingestion + File Explorer
**Status**: Not Started
**Goal**: Enable users to point at a local folder and automatically discover, catalog, and query its data files
**Deliverables**:
- Folder scanner that detects CSV, TSV, JSON, YAML, Parquet, and text files
- Auto-registration of discovered files as DuckDB external tables
- File metadata catalog stored in RxDB (file path, schema, row count, last modified)
- Basic data preview UI (first N rows of any detected file)
- LanceDB table creation from ingested text/documents (for vector search readiness)
- Clone/cache mechanism to import external file data into embedded DuckDB

---

## Phase 3: Visualization Engine
**Status**: Not Started
**Goal**: Wire up the three visualization libraries to consume DuckDB query results via Arrow
**Deliverables**:
- Perspective.js integration with DuckDBHandler adapter (DuckDB-WASM in renderer or native via IPC)
- Observable Plot integration with DuckDBClient and Arquero for exploratory charting
- Apache ECharts integration consuming DuckDB-aggregated JSON summaries
- Unified "view switcher" UI: toggle between data grid (Perspective), chart (Plot/ECharts), and raw SQL
- Demonstration: same dataset rendered across all three visualization modes

---

## Phase 4: Vector Search + Full-Text Search
**Status**: Not Started
**Goal**: Enable semantic search, full-text search, and hybrid search across ingested data via LanceDB
**Deliverables**:
- Embedding pipeline: configurable model (local via Transformers.js or API-based via OpenAI/Claude)
- LanceDB vector index creation from text columns of ingested datasets
- Full-text search index creation in LanceDB
- Hybrid search UI: combined vector similarity + full-text + SQL metadata filtering
- DuckDB-to-LanceDB bridge: query LanceDB .lance files from DuckDB via the Lance extension
- Search results displayed in Perspective.js grid with relevance scoring

---

## Phase 5: Remote Data Source Connectors
**Status**: Not Started
**Goal**: Connect the app to external cloud data platforms, treating remote data as queryable alongside local data
**Deliverables**:
- S3/GCS connector via LanceDB's native remote storage support
- BigQuery connector (DuckDB's BigQuery extension or dedicated adapter)
- ClickHouse connector (DuckDB's ClickHouse scanner or HTTP API)
- Mixpanel/PostHog connector (REST API adapters with DuckDB table registration)
- Google Sheets and Airtable connectors (API-based with local caching)
- Source management UI: add/remove/test connections, view schema, preview data
- Freeze/snapshot: capture remote table state to local DuckDB/LanceDB for offline access

---

## Phase 6: AI Chat + Agent Integration
**Status**: Not Started
**Goal**: Add conversational AI that understands loaded datasets and can answer questions, generate queries, and produce visualizations
**Deliverables**:
- Claude Agent SDK integration for natural language data exploration
- MCP server exposing loaded datasets as resources and DuckDB/LanceDB as tools
- Agent Client Protocol (ACP) compliance layer -- third-party agents can plug in
- Chat UI panel with conversation history (stored in RxDB)
- Query generation: user describes what they want, agent writes and executes DuckDB SQL
- Visualization generation: agent selects appropriate chart type and configures it
- RAG pipeline: chat uses LanceDB vector search for context retrieval from large datasets

---

## Phase 7: Publishing Engine
**Status**: Not Started
**Goal**: Enable users to export their analysis sessions as standalone interactive web applications
**Deliverables**:
- Export pipeline: package current views/charts/data as a self-contained web app
- Embedded Perspective.js and Observable Plot viewers in exported app
- Contextualized chat UI in published output (connects to hosted AI endpoint)
- Static hosting support (Vercel, Netlify, S3 static site)
- Share link generation with optional access controls
- Template system for common report types (dashboard, exploration notebook, presentation)

---

## Meta-Notes

**Execution order**: Phases 1-3 are sequential dependencies (need the database layer before visualization). Phase 4 can begin in parallel with Phase 3. Phases 5-6 can partially parallelize after Phase 3. Phase 7 requires Phases 3 and 6.

**Relationship to Data Agent project**: Data Codex is the GUI counterpart to the terminal-based Data Agent. They share the philosophical goal of "AI-powered data exploration" but differ in interface (desktop app vs CLI), architecture (embedded databases vs MCP connectors), and output (published web apps vs terminal insights). They may eventually share connectors or agent protocols.

**Key risk**: The DuckDB Lance extension and Perspective.js DuckDBHandler are relatively new integrations. Phase 1 should validate these work correctly in Electron before building on top of them.

**"Big Data on a Small Device" principle**: Every architectural decision should prioritize streaming/scanning over loading, Arrow over JSON, and predicate pushdown over full-table transfer. If a feature requires loading an entire dataset into JS memory, it needs redesign.
