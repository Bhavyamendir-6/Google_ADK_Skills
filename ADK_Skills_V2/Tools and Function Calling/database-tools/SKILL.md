---
name: database-tools
description: >
  Use this skill when building agent tools that query, read, insert, update, or
  delete data from relational databases (PostgreSQL, MySQL, SQLite) or NoSQL
  databases (MongoDB, Firestore, Bigtable). Covers safe query construction,
  connection management, and result normalization.
compatibility:
  - Claude Code
  - Google ADK
  - OpenAI Agents
  - MCP-based agents
  - Tool-using LLM agents
metadata:
  author: agent-skills-generator
  version: "1.0"
---

## Purpose

This skill defines the standard pattern for building database-backed agent
tools. It ensures that SQL queries are parameterized (preventing injection),
connections are managed with a connection pool, results are serialized to
JSON-safe dicts, and write operations require explicit confirmation before
execution.

## When to Use

- When building a tool that reads from or writes to a relational or document
  database.
- When an agent needs to look up, filter, aggregate, or update records.
- When wrapping an ORM (SQLAlchemy, Tortoise) as an agent tool.

## When NOT to Use

- Do not use for external API calls — use `external-api-tools`.
- Do not use for vector database search — use a dedicated vector-search tool.
- Do not allow raw SQL strings passed by the LLM to be executed directly.
  Always use parameterized queries.

## Capabilities

- Implements parameterized SQL queries to prevent injection.
- Uses a connection pool (`psycopg2.pool` or `SQLAlchemy`) to reuse connections.
- Serializes `datetime`, `Decimal`, and other non-JSON types to strings.
- Limits result set size to prevent context overflow.
- Separates read and write tools — write tools require explicit confirmation.
- Applies `@tool_error_handler` for database-specific exceptions.

## Step-by-Step Instructions

1. **Create a connection pool** at module level (not inside the tool function):
   ```python
   from sqlalchemy import create_engine
   import os
   engine = create_engine(os.environ["DATABASE_URL"], pool_size=5, max_overflow=2)
   ```

2. **Separate read tools from write tools** — define them as distinct functions:
   - `query_<table_name>` for reads.
   - `insert_<table_name>`, `update_<table_name>`, `delete_<table_name>` for writes.

3. **Use parameterized queries exclusively**:
   ```python
   # CORRECT
   cursor.execute("SELECT * FROM users WHERE email = %s", (email,))
   # WRONG - never do this
   cursor.execute(f"SELECT * FROM users WHERE email = '{email}'")
   ```

4. **Limit result rows** with `LIMIT`:
   ```python
   cursor.execute("SELECT * FROM products WHERE category = %s LIMIT %s", (category, limit))
   ```
   Default `limit` parameter: 20. Maximum: 100.

5. **Serialize results** to JSON-safe dicts:
   ```python
   def _serialize_row(row: dict) -> dict:
       return {k: str(v) if not isinstance(v, (str, int, float, bool, type(None))) else v
               for k, v in row.items()}
   ```

6. **For write tools**: Return a confirmation preview before executing on the
   first call. Only execute when `confirm=True` is passed.

7. **Apply `@tool_error_handler`** and map database exceptions:
   - `OperationalError` → `transient`
   - `ProgrammingError` → `invalid_input`
   - `IntegrityError` → `permanent` (e.g., unique constraint)

## Input Format

Read tool:
```
table: <str>
filters: dict           # {column: value} — safe filter conditions
columns: list[str]      # Columns to retrieve; empty = all
limit: int              # Default: 20, max: 100
```

Write tool:
```
table: <str>
data: dict              # Row data to insert/update
condition: dict         # WHERE condition for update/delete
confirm: bool           # Must be True to execute
```

## Output Format

Read result:
```python
{
  "rows": [
    {"id": 1, "name": "Alice", "email": "alice@example.com"},
  ],
  "count": 1,
  "truncated": False
}
```

Write result:
```python
{
  "affected_rows": 1,
  "operation": "INSERT",
  "table": "users"
}
```

Confirmation preview (before write):
```python
{
  "preview": "INSERT INTO users (name, email) VALUES ('Alice', 'alice@example.com')",
  "requires_confirmation": True
}
```

## Error Handling

- `psycopg2.OperationalError` (connection lost) → `transient`, `recoverable: True`
- `psycopg2.IntegrityError` (constraint violation) → `permanent`, `recoverable: False`
- `sqlalchemy.exc.ProgrammingError` (invalid column) → `invalid_input`, `recoverable: False`
- Result set > 100 rows: Truncate and set `"truncated": True` in result.
- Empty result set: Return `{"rows": [], "count": 0}` — not an error.

## Examples

### Example 1 — Read tool with SQLAlchemy

```python
from sqlalchemy import text
from typing import Optional

@tool_error_handler
def query_users(email: Optional[str] = None, limit: int = 20) -> dict:
    """Query users from the database with optional email filter.

    Args:
        email (Optional[str]): Filter by exact email address. Defaults to None (all users).
        limit (int): Maximum number of rows to return. Defaults to 20.

    Returns:
        dict: Contains 'rows' (list of user dicts) and 'count' (int).
    """
    limit = min(limit, 100)
    with engine.connect() as conn:
        if email:
            result = conn.execute(text("SELECT id, name, email FROM users WHERE email = :email LIMIT :limit"),
                                  {"email": email, "limit": limit})
        else:
            result = conn.execute(text("SELECT id, name, email FROM users LIMIT :limit"), {"limit": limit})
        rows = [dict(row._mapping) for row in result]
    return {"rows": rows, "count": len(rows), "truncated": len(rows) == limit}
```

### Example 2 — Write tool with confirmation

```python
@tool_error_handler
def insert_user(name: str, email: str, confirm: bool = False) -> dict:
    """Insert a new user into the database.

    Args:
        name (str): Full name of the user.
        email (str): Email address (must be unique).
        confirm (bool): Set to True to execute the insert. Defaults to False (preview only).

    Returns:
        dict: Preview dict if confirm=False; affected_rows dict if confirm=True.
    """
    if not confirm:
        return {
            "preview": f"INSERT INTO users (name, email) VALUES ('{name}', '{email}')",
            "requires_confirmation": True,
        }
    with engine.connect() as conn:
        result = conn.execute(
            text("INSERT INTO users (name, email) VALUES (:name, :email)"),
            {"name": name, "email": email},
        )
        conn.commit()
    return {"affected_rows": result.rowcount, "operation": "INSERT", "table": "users"}
```

## Edge Cases

- **LLM attempts raw SQL input**: The LLM should never control raw SQL strings.
  Do not create a `run_sql(query: str)` tool in production.
- **Transactions spanning multiple tools**: Use a shared connection passed via
  context; do not open separate connections per tool in a transaction.
- **Large BLOB fields**: Never return binary BLOBs. Return a URL or file path.
- **Schema changes**: If a query fails due to a missing column, return
  `invalid_input` and prompt schema refresh.

## Integration Notes

- **Google ADK**: Register read tools freely; write tools should use
  `action_confirmation` callbacks to require user approval before execution.
- **MCP Toolbox for Databases**: Google's MCP Toolbox provides pre-built
  database tool servers. Use this skill when custom tools are needed beyond
  those provided.
- **Google Cloud Spanner**: Use the Spanner ADK integration for global-scale
  relational tools. Use this skill's patterns for custom Spanner queries.
- **tool-validation skill**: Apply input validation before executing any query
  to prevent SQL injection attempts via LLM-controlled filter values.
