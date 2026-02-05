---
received: 2026-02-04T00:00:00Z
source:
  - inputs/data-codex/BRIEF.md
  - inputs/data-codex/ROADMAP.md
domain: system-design
---

# Data Codex

## Vision

A desktop analytics application that functions as a "serverless data warehouse on a small device." Built on Electron, Data Codex combines embedded high-performance databases (LanceDB, DuckDB, RxDB) with AI-powered chat and GPU-accelerated data visualization to let a single user ingest, explore, search, and publish insights from heterogeneous data sources -- local files, local databases, and remote cloud platforms -- all without a server.

The core architectural insight: by treating Apache Arrow as a universal memory format and using embedded analytical engines (DuckDB as compute plane, LanceDB as storage/vector plane, RxDB as reactive state plane), a single desktop app can achieve warehouse-grade analytics with zero infrastructure. Data flows between layers via zero-copy Arrow transfers, keeping performance high and memory overhead low.

## Value Proposition

**For the user (data analyst/creator):**
- Point at a folder of CSV/JSON/Parquet files and immediately explore with interactive visualizations
- Connect remote sources (BigQuery, Snowflake, S3, Mixpanel, PostHog) alongside local data
- Vector search + full-text search + SQL in one unified interface
- AI chat contextualized to your data (Claude Agent SDK, MCP, ACP-compliant agents)
- Publish findings as interactive web apps with built-in chat UI

**For consumers of published outputs:**
- Beautiful, interactive data experiences (not static PDFs)
- Contextualized AI chat for self-service exploration
- Accessible without technical setup

## Success Criteria

- [ ] App can ingest and query local files (CSV, TSV, JSON, YAML, Parquet) with sub-second response
- [ ] DuckDB + LanceDB integration works via zero-copy Arrow (no JSON serialization overhead)
- [ ] Vector similarity search, full-text search, and SQL all work from a single query interface
- [ ] At least 3 remote data source connectors functional (e.g., S3, BigQuery, ClickHouse)
- [ ] Perspective.js renders 1M+ row datasets without freezing the UI
- [ ] Observable Plot and Apache ECharts produce publication-quality charts from DuckDB queries
- [ ] AI chat can answer questions about loaded datasets using Claude Agent SDK
- [ ] User can publish an interactive web app from their analysis session

## Key Systems

1. **Embedded Database Trio**
   - **LanceDB** -- Multimodal lakehouse: vector embeddings, full-text search, blob storage, S3-native
   - **DuckDB** -- Analytical SQL engine: complex joins, window functions, cross-format queries (Lance/CSV/Parquet/JSON)
   - **RxDB** -- Reactive NoSQL: application state, user preferences, real-time UI updates (SQLite storage adapter under the hood)

2. **Apache Arrow Data Plane**
   - Zero-copy data transfer between all database layers
   - MessagePort IPC for Electron (avoids JSON serialization)
   - Worker Threads for heavy database operations (prevents UI freezing)
   - Predicate pushdown via DuckDB for minimal data transfer

3. **Data Source Connectors**
   - Local: folder scanning (CSV, TSV, JSON, YAML, Parquet, text), SQLite, DuckDB files
   - Remote: ClickHouse, BigQuery, Snowflake, S3, Mixpanel, PostHog, Google Sheets, Airtable
   - Clone/freeze/cache: capture external tables to embedded DuckDB/LanceDB instances

4. **Visualization Layer**
   - **Perspective.js** (w/ DuckDBHandler) -- GPU-accelerated data grids, pivot tables, streaming; native Arrow consumption
   - **Observable Plot** (w/ Arquero & DuckDBClient) -- Exploratory charts, first-class DuckDB integration
   - **Apache ECharts** -- Polished dashboard reports for end-users (consumes DuckDB-aggregated summaries)

5. **AI Integration**
   - Claude Agent SDK for conversational data exploration
   - GitHub Copilot CLI SDK compatibility
   - Agent Client Protocol (ACP) compliance -- any ACP agent works as a plugin
   - MCP built-in for tool/resource access
   - Contextualized chat UI in published deliverables

6. **Publishing Engine**
   - Export analysis sessions as standalone interactive web apps
   - Embedded chat UI for consumer self-service
   - Observable Plot + Perspective.js + ECharts for visual richness

## Constraints & Assumptions

**Constraints:**
- macOS-first (Electron cross-platform later)
- Single-user desktop app (no multi-user sync required)
- Must work offline for local data (remote connectors require network)
- Electron bundle size matters -- minimize dependencies
- No standalone SQLite -- RxDB may use it internally as a storage adapter

**Assumptions:**
- LanceDB Node.js SDK is stable enough for production Electron use
- DuckDB Lance extension enables zero-copy querying of LanceDB data
- Perspective.js DuckDBHandler works in Electron renderer process
- Apache Arrow interop between DuckDB, LanceDB, and visualization libraries is seamless
- ACP specification is stable enough to build against

**Architecture Principles:**
- "Stream and Scan" not "Load then Process" -- never load full datasets into JS memory
- LanceDB = Data Plane (storage/retrieval), DuckDB = Compute Plane (analysis/logic)
- Arrow is the universal language -- every component speaks it
- Main Process runs databases, Renderer Process runs visualizations
- Worker Threads isolate heavy computation from UI responsiveness

## Roadmap

### Phase 1: Electron Shell + Embedded Database Foundation
Establish the core Electron app with all three embedded databases wired together and communicating via Apache Arrow.

### Phase 2: Local Data Ingestion + File Explorer
Enable users to point at a local folder and automatically discover, catalog, and query its data files.

### Phase 3: Visualization Engine
Wire up Perspective.js, Observable Plot, and Apache ECharts to consume DuckDB query results via Arrow.

### Phase 4: Vector Search + Full-Text Search
Enable semantic search, full-text search, and hybrid search across ingested data via LanceDB.

### Phase 5: Remote Data Source Connectors
Connect the app to external cloud data platforms alongside local data.

### Phase 6: AI Chat + Agent Integration
Conversational AI that understands loaded datasets for query generation and visualization.

### Phase 7: Publishing Engine
Export analysis sessions as standalone interactive web applications.
