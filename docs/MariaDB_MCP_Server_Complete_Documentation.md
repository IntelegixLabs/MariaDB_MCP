# MariaDB MCP Server - Complete Documentation

## Table of Contents
1. [Project Overview](#project-overview)
2. [Architecture](#architecture)
3. [Features & Capabilities](#features--capabilities)
4. [Installation & Setup](#installation--setup)
5. [Configuration](#configuration)
6. [API Reference](#api-reference)
7. [Security & Safety](#security--safety)
8. [Performance & Reliability](#performance--reliability)
9. [Testing](#testing)
10. [Deployment](#deployment)
11. [Hackathon Submission](#hackathon-submission)
12. [Contributing](#contributing)

---

## Project Overview

The MariaDB MCP Server is a Model Context Protocol (MCP) implementation that provides AI assistants and tools with secure, observable access to MariaDB databases. It enables natural language interactions with relational data while maintaining strict security guardrails and providing optional vector store capabilities for semantic search.

### Key Benefits
- **AI-Native Interface**: Direct integration with AI assistants via MCP protocol
- **Security First**: Read-only mode, write allow-lists, and input validation
- **Observability**: Correlation IDs, structured logging, and health monitoring
- **Reliability**: Connection pooling, query timeouts, and rate limiting
- **Extensibility**: Pluggable embedding providers and configurable operations

---

## Architecture

### System Components

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   AI Assistant  │───▶│   MCP Client     │───▶│  FastMCP Server │
└─────────────────┘    └──────────────────┘    └─────────────────┘
                                                         │
                                                         ▼
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│  Rate Limiter   │◀───│  Tool Handler    │───▶│  Connection Pool│
└─────────────────┘    └──────────────────┘    └─────────────────┘
                                                         │
                                                         ▼
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│  Write Guard    │    │  EmbeddingService│    │    MariaDB      │
└─────────────────┘    └──────────────────┘    └─────────────────┘
                                │
                                ▼
                       ┌─────────────────┐
                       │ Embedding APIs  │
                       │ (OpenAI/Gemini/ │
                       │  HuggingFace)   │
                       └─────────────────┘
```

### Data Flow

1. **Request Processing**:
   - Client sends MCP tool call
   - Rate limiter checks request frequency
   - Tool handler validates parameters
   - Write guard checks operation permissions

2. **Query Execution**:
   - Connection acquired from pool
   - Query executed with timeout protection
   - Results returned with correlation ID
   - Structured error handling

3. **Vector Operations** (Optional):
   - Document embedding generation
   - Vector similarity search
   - Metadata storage and retrieval

---

## Features & Capabilities

### Core Database Tools
- **`list_databases()`**: List all accessible databases
- **`list_tables(database_name)`**: List tables in a database
- **`get_table_schema(database_name, table_name)`**: Get column information
- **`get_table_schema_with_relations(database_name, table_name)`**: Include foreign key relationships
- **`execute_sql(sql_query, database_name, parameters?)`**: Execute read-only SQL queries
- **`create_database(database_name)`**: Create new database (when not read-only)
- **`explain_sql_plan(sql_query, database_name)`**: Get query execution plan

### Vector Store Tools (Optional)
- **`create_vector_store(database_name, vector_store_name, model_name?, distance_function?)`**: Create vector store table
- **`list_vector_stores(database_name)`**: List all vector stores
- **`delete_vector_store(database_name, vector_store_name)`**: Remove vector store
- **`insert_docs_vector_store(database_name, vector_store_name, documents, metadata?)`**: Insert documents with embeddings
- **`search_vector_store(user_query, database_name, vector_store_name, k?)`**: Semantic search

### System Tools
- **`ping()`**: Liveness check
- **`health()`**: System health with pool status and configuration
- **`version()`**: Server version information

### Embedding Providers
- **OpenAI**: text-embedding-3-small, text-embedding-3-large
- **Google Gemini**: text-embedding-004
- **Hugging Face**: BAAI/bge-m3, intfloat/multilingual-e5-large-instruct

---

## Installation & Setup

### Prerequisites
- Python 3.11+
- MariaDB server with VECTOR support (for vector stores)
- Optional: OpenAI API key, Gemini API key, or Hugging Face model

### Installation

```bash
# Clone the repository
git clone <repository-url>
cd MariaDB_MCP

# Install dependencies
pip install -e .

# Or using pip directly
pip install asyncmy fastmcp[websockets] python-dotenv
```

### Environment Configuration

Create a `.env` file:

```bash
# Database Configuration
DB_HOST=localhost
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=your_database
DB_CHARSET=utf8mb4

# MCP Server Configuration
MCP_READ_ONLY=true
MCP_MAX_POOL_SIZE=10
MCP_QUERY_TIMEOUT_SECS=30

# Rate Limiting
MCP_RATE_LIMIT_ENABLED=true
MCP_RATE_LIMIT_REQUESTS_PER_MINUTE=60
MCP_RATE_LIMIT_BURST=10

# Write Guardrails
MCP_ALLOWED_WRITE_OPERATIONS=CREATE_DATABASE,CREATE_TABLE,CREATE_INDEX

# Logging
LOG_LEVEL=INFO
LOG_FILE=logs/mcp_server.log
MCP_CORRELATION_LOGGING=true

# Embeddings (Optional)
EMBEDDING_PROVIDER=openai
OPENAI_API_KEY=your_openai_key
```

---

## Configuration

### Database Settings
| Variable | Default | Description |
|----------|---------|-------------|
| `DB_HOST` | localhost | MariaDB host |
| `DB_PORT` | 3306 | MariaDB port |
| `DB_USER` | - | Database username |
| `DB_PASSWORD` | - | Database password |
| `DB_NAME` | - | Default database |
| `DB_CHARSET` | - | Character set |

### MCP Server Settings
| Variable | Default | Description |
|----------|---------|-------------|
| `MCP_READ_ONLY` | true | Enable read-only mode |
| `MCP_MAX_POOL_SIZE` | 10 | Connection pool size |
| `MCP_QUERY_TIMEOUT_SECS` | 30 | Query timeout in seconds |

### Rate Limiting
| Variable | Default | Description |
|----------|---------|-------------|
| `MCP_RATE_LIMIT_ENABLED` | true | Enable rate limiting |
| `MCP_RATE_LIMIT_REQUESTS_PER_MINUTE` | 60 | Requests per minute |
| `MCP_RATE_LIMIT_BURST` | 10 | Burst capacity |

### Security
| Variable | Default | Description |
|----------|---------|-------------|
| `MCP_ALLOWED_WRITE_OPERATIONS` | CREATE_DATABASE,CREATE_TABLE,CREATE_INDEX | Allowed write operations |
| `ALLOWED_ORIGINS` | localhost,127.0.0.1 | CORS allowed origins |
| `ALLOWED_HOSTS` | localhost,127.0.0.1 | Trusted hosts |

---

## API Reference

### Tool Parameters

#### `execute_sql`
```python
execute_sql(
    sql_query: str,           # SQL query (SELECT, SHOW, DESCRIBE only)
    database_name: str,        # Target database
    parameters: List[Any]      # Optional query parameters
) -> List[Dict[str, Any]]
```

#### `search_vector_store`
```python
search_vector_store(
    user_query: str,           # Natural language query
    database_name: str,        # Database containing vector store
    vector_store_name: str,    # Vector store table name
    k: int = 7                # Number of results to return
) -> List[Dict[str, Any]]
```

### Response Format

#### Success Response
```json
[
    {
        "id": 1,
        "document": "Sample text",
        "metadata": {"source": "file1.txt"},
        "distance": 0.123
    }
]
```

#### Error Response
```json
{
    "error_code": "RATE_LIMIT_EXCEEDED",
    "message": "Request rate limit exceeded",
    "details": {
        "limit": 60,
        "burst": 10
    },
    "correlation_id": "abc12345"
}
```

---

## Security & Safety

### Read-Only Mode
- Blocks all non-SELECT operations when enabled
- Prevents accidental data modification
- Configurable via `MCP_READ_ONLY` environment variable

### Write Guardrails
- Allow-list of permitted write operations
- Configurable via `MCP_ALLOWED_WRITE_OPERATIONS`
- Default: CREATE_DATABASE, CREATE_TABLE, CREATE_INDEX

### Input Validation
- Safe identifier validation for database/table names
- SQL injection prevention through parameterized queries
- Query comment stripping and sanitization

### Rate Limiting
- Token bucket algorithm with configurable limits
- Per-client request tracking
- Prevents abuse and DoS attacks

---

## Performance & Reliability

### Connection Pooling
- Async connection pool using asyncmy
- Configurable pool size and recycling
- Automatic connection health checks

### Query Timeouts
- Configurable query timeout (default: 30 seconds)
- Prevents runaway queries
- Structured timeout error responses

### Observability
- Correlation IDs for request tracking
- Structured logging with configurable levels
- Health and version endpoints
- Rotating log files with size limits

### Error Handling
- Structured error responses with error codes
- Detailed error context and correlation IDs
- Graceful degradation and recovery

---

## Testing

### Running Tests
```bash
# Run all tests
python -m pytest src/tests/

# Run specific test file
python -m pytest src/tests/test_mariadb_mcp_tools.py

# Run with coverage
python -m pytest --cov=src src/tests/
```

### Test Categories
- **Unit Tests**: Individual tool functionality
- **Integration Tests**: End-to-end MCP server behavior
- **Error Handling**: Exception scenarios and edge cases
- **Security Tests**: Input validation and access control

---

## Deployment

### Docker Deployment
```bash
# Build image
docker build -t mariadb-mcp-server .

# Run container
docker run -d \
  --name mariadb-mcp \
  -p 9001:9001 \
  -e DB_HOST=your_host \
  -e DB_USER=your_user \
  -e DB_PASSWORD=your_password \
  mariadb-mcp-server
```

### Docker Compose
```yaml
version: '3.8'
services:
  mariadb-mcp:
    build: .
    ports:
      - "9001:9001"
    environment:
      - DB_HOST=mariadb
      - DB_USER=root
      - DB_PASSWORD=password
    depends_on:
      - mariadb
```

### Production Considerations
- Use environment variables for sensitive configuration
- Configure appropriate log levels and rotation
- Set up monitoring and alerting
- Consider load balancing for high availability

---

## Hackathon Submission

### Idea Title
**MariaDB MCP Server: AI-Native Interface for Relational + Vector Workloads**

### Problem Statement
Organizations want to leverage AI agents to interact with operational and analytical data safely. Traditional database access patterns are not AI-friendly, lack semantic capabilities, and risk unsafe writes or long-running queries. Teams need a secure, observable, AI-native interface to query structured data and perform semantic search with guardrails.

### Approach
Expose MariaDB through the Model Context Protocol (MCP) so AI assistants can call tools for schema discovery, read-only SQL, and vector search. Add reliability (timeouts, pooling), safety (read-only mode, write allow-list), abuse prevention (rate limiting), and observability (structured errors, correlation IDs). Optional embeddings support enables vector stores for semantic search at the database layer.

### Use Cases
- **Analysts**: Ask natural questions via AI to retrieve tables, schemas, and summaries
- **Developers**: Use MCP tools in IDE assistants to inspect DBs and run safe queries
- **LLM Apps**: Perform RAG against vector stores in MariaDB with consistent security controls

### LinkedIn Post Template
> Signed up for the #mariadb #python #hackathon with my idea "MariaDB MCP Server: AI-Native Interface for Relational + Vector Workloads". Excited to bring safe, observable, AI-powered data access to life! #LLM #RAG #MCP @MariaDB @Hackerearth

---

## Contributing

### Development Setup
1. Fork the repository
2. Create a feature branch
3. Make changes with tests
4. Submit a pull request

### Code Style
- Follow PEP 8 guidelines
- Use type hints
- Add docstrings for public methods
- Include tests for new features

### Testing Requirements
- All new features must include tests
- Maintain or improve test coverage
- Test both success and error scenarios

---

## License

This project is licensed under the MIT License - see the LICENSE file for details.

---

## Support

For questions, issues, or contributions:
- Create an issue in the repository
- Check the documentation
- Review the test examples

---

*This documentation covers the complete MariaDB MCP Server implementation, including all features, configuration options, security measures, and deployment instructions.*
