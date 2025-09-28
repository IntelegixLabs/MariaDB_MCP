## MariaDB MCP Server — Project Documentation

### Overview
The MariaDB MCP Server exposes MariaDB database operations via the Model Context Protocol (MCP), enabling AI assistants and tools to safely query relational data and optional vector stores. It provides secure read-only SQL execution, schema discovery, and optional embeddings-based semantic search, while enforcing guardrails, timeouts, and rate limits for reliability and safety.

### Key Features
- Read-only SQL tools: list databases/tables, get schema, execute SELECT/SHOW/DESCRIBE
- Optional vector store: create/list/delete stores, insert documents, semantic search
- Embedding providers: OpenAI, Gemini, and Hugging Face (configurable)
- Security & safety: read-only mode, write allow-list, identifier validation
- Reliability: connection pooling, query timeouts
- Abuse prevention: configurable rate limiting
- Observability: health/version tools, correlation IDs, structured errors, logging

### Architecture
- `src/server.py`: FastMCP server, connection pool (asyncmy), MCP tool registration
- `src/config.py`: environment-driven configuration and logging setup
- `src/embeddings.py`: pluggable embedding service for vector store workflows
- `src/tests/…`: unit and integration tests for tools and server behavior

Data flow (read):
Client → MCP tool (e.g., `execute_sql`) → validation & rate-limit → `_execute_query` (timeout, pool) → MariaDB → results → structured response

Data flow (vector search):
Client → `search_vector_store` → embed query → similarity search (COSINE/EUCLIDEAN) → results with metadata

### Configuration (env/.env)
- Database: `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD`, `DB_NAME`, `DB_CHARSET`
- MCP: `MCP_READ_ONLY`, `MCP_MAX_POOL_SIZE`, `MCP_QUERY_TIMEOUT_SECS`
- Rate limiting: `MCP_RATE_LIMIT_ENABLED`, `MCP_RATE_LIMIT_REQUESTS_PER_MINUTE`, `MCP_RATE_LIMIT_BURST`
- Write guardrails: `MCP_ALLOWED_WRITE_OPERATIONS`
- Logging: `LOG_LEVEL`, `LOG_FILE`, `LOG_MAX_BYTES`, `LOG_BACKUP_COUNT`, `MCP_CORRELATION_LOGGING`, `ALLOWED_ORIGINS`, `ALLOWED_HOSTS`
- Embeddings: `EMBEDDING_PROVIDER`, `OPENAI_API_KEY`, `GEMINI_API_KEY`, `HF_MODEL`

### Tools (selected)
- `list_databases()` → List accessible DBs
- `list_tables(database_name)` → List tables
- `get_table_schema(database_name, table_name)` → Column info
- `get_table_schema_with_relations(database_name, table_name)` → Foreign keys included
- `execute_sql(sql_query, database_name, parameters?)` → Read-only queries
- `create_database(database_name)` → When not read-only, subject to allow-list
- `ping()` / `health()` / `version()`
- Vector tools (when embeddings enabled):
  - `create_vector_store(database_name, vector_store_name, model_name?, distance_function?)`
  - `list_vector_stores(database_name)`
  - `delete_vector_store(database_name, vector_store_name)`
  - `insert_docs_vector_store(database_name, vector_store_name, documents, metadata?)`
  - `search_vector_store(user_query, database_name, vector_store_name, k?)`
  - `explain_sql_plan(sql_query, database_name)`

### Safety & Guardrails
- Read-only mode blocks non-SELECT statements
- Allowed write operations configurable when not read-only
- Safe identifier validation for DB/table names
- Query timeouts (fail fast) with structured timeout errors
- Token-bucket rate limiter per client

### Run Modes (transport)
- stdio (default), sse, http with Starlette middleware (CORS, Trusted Host)

### Development
- Python ≥ 3.11. Dependencies in `pyproject.toml`
- Tests in `src/tests`
- Logging to console and rotating file (`logs/mcp_server.log` by default)

### Deployment
- Dockerfile and docker-compose provided
- Configure env via `.env` or environment variables

### Extensibility
- Add new MCP tools by registering methods in `register_tools`
- Extend embeddings by adding providers/models in `embeddings.py`


