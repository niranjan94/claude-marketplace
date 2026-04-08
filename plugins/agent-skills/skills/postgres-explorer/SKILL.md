---
name: postgres-explorer
description: Use psql to explore and query PostgreSQL databases in strict read-only mode. Invoke this skill when the user explicitly asks to explore a database, query tables, inspect schema, check data, debug data issues, or investigate database performance -- anything that requires connecting to a PostgreSQL database via psql. Do NOT use for writing migrations, inserting data, or any write operations.
---

# PostgreSQL Database Explorer (Read-Only)

This skill connects to PostgreSQL databases via `psql` to help explore schema, query data, and investigate database structure and performance. Every interaction enforces strict read-only access -- no exceptions, no workarounds, no "just this once."

## Safety: Read-Only Enforcement

This is non-negotiable. Every single psql session MUST enforce read-only mode. The reason is simple: this skill exists to look at data, never to change it. A single accidental write in production can cause real damage that no amount of clever queries can undo.

### Connection-Level Read-Only

Always set the transaction to read-only immediately after connecting:

```bash
psql "<connection_string>" -c "SET default_transaction_read_only = ON;" -c "<your_query>"
```

Or when running multiple queries in a session:

```bash
psql "<connection_string>" <<'SQL'
SET default_transaction_read_only = ON;
-- all queries here
SQL
```

### Forbidden Operations -- Always, Without Exception

Never generate or execute any of the following, even if the user asks:

- `INSERT`, `UPDATE`, `DELETE`, `TRUNCATE`
- `CREATE`, `ALTER`, `DROP` (any DDL)
- `GRANT`, `REVOKE`
- `COPY ... FROM` (COPY ... TO is fine for exporting query results)
- Any function or procedure that modifies data
- `SET default_transaction_read_only = OFF` or any attempt to disable read-only mode

If the user asks for a write operation, decline and explain that this skill is read-only by design. Suggest they perform writes manually or through their application's migration/ORM tooling.

### EXPLAIN ANALYZE

`EXPLAIN ANALYZE` is permitted only for `SELECT` statements. It executes the query to collect actual timing data, but since the query is a SELECT and the transaction is read-only, no data is modified. Never use `EXPLAIN ANALYZE` on INSERT/UPDATE/DELETE -- use plain `EXPLAIN` instead if the user wants to see the plan for a write query.

## Connecting to the Database

### Step 1: Find the connection string

Check the project's `.env` file (or `.env.local`, `.env.development`, etc.) for a `DATABASE_URL` variable. Common variable names to look for:

- `DATABASE_URL`
- `POSTGRES_URL`
- `DB_URL`
- `PG_CONNECTION_STRING`

If no connection string is found, ask the user to provide one. Do not guess or construct connection strings from partial information.

### Step 2: Verify connectivity

Before doing anything else, verify the connection works and confirm read-only mode:

```bash
psql "<connection_string>" -c "SET default_transaction_read_only = ON; SELECT current_database(), current_user, version();"
```

This confirms: the connection works, which database and user you're connected as, and that the PostgreSQL version is known (useful for feature availability).

## Querying Patterns

### Token-Efficient Output

psql can be verbose. Use these flags to keep output compact and context-window-friendly:

- `-t` (tuples only) -- omit column headers and row count footers when you just need values
- `-A` (unaligned) -- no padding/alignment whitespace
- `-F','` -- comma-separated (useful with `-A` for CSV-like output)
- `--pset=format=wrapped` -- wrap long columns instead of truncating
- `\x` or `--expanded` -- vertical format for wide tables (one field per line)
- `LIMIT` -- always limit result sets. Default to `LIMIT 20` unless the user asks for more

Prefer compact output. For example, when listing tables:

```bash
psql "$DATABASE_URL" -t -A -c "SET default_transaction_read_only = ON; SELECT tablename FROM pg_tables WHERE schemaname = 'public' ORDER BY tablename;"
```

### Common Exploration Queries

**List all tables with row counts (estimates):**
```sql
SET default_transaction_read_only = ON;
SELECT schemaname, relname AS table_name,
       n_live_tup AS estimated_rows
FROM pg_stat_user_tables
ORDER BY n_live_tup DESC;
```

**Describe a table's columns:**
```sql
SET default_transaction_read_only = ON;
SELECT column_name, data_type, is_nullable, column_default
FROM information_schema.columns
WHERE table_schema = 'public' AND table_name = '<table>'
ORDER BY ordinal_position;
```

**Show indexes on a table:**
```sql
SET default_transaction_read_only = ON;
SELECT indexname, indexdef
FROM pg_indexes
WHERE tablename = '<table>';
```

**Show foreign key relationships:**
```sql
SET default_transaction_read_only = ON;
SELECT
    tc.table_name, kcu.column_name,
    ccu.table_name AS foreign_table,
    ccu.column_name AS foreign_column
FROM information_schema.table_constraints tc
JOIN information_schema.key_column_usage kcu
    ON tc.constraint_name = kcu.constraint_name
JOIN information_schema.constraint_column_usage ccu
    ON tc.constraint_name = ccu.constraint_name
WHERE tc.constraint_type = 'FOREIGN KEY'
    AND tc.table_schema = 'public'
ORDER BY tc.table_name;
```

**Sample rows from a table:**
```sql
SET default_transaction_read_only = ON;
SELECT * FROM <table> LIMIT 10;
```

**Table sizes on disk:**
```sql
SET default_transaction_read_only = ON;
SELECT relname AS table_name,
       pg_size_pretty(pg_total_relation_size(relid)) AS total_size
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC;
```

### Performance Investigation

When investigating slow queries or performance issues:

```sql
SET default_transaction_read_only = ON;
-- Check for long-running queries
SELECT pid, now() - pg_stat_activity.query_start AS duration,
       query, state
FROM pg_stat_activity
WHERE (now() - pg_stat_activity.query_start) > interval '5 seconds'
      AND state != 'idle'
ORDER BY duration DESC;
```

```sql
SET default_transaction_read_only = ON;
-- Table bloat and vacuum stats
SELECT relname, last_vacuum, last_autovacuum,
       n_dead_tup, n_live_tup
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

For query plans, always use EXPLAIN on SELECT statements first, then EXPLAIN ANALYZE only if actual timings are needed:

```sql
SET default_transaction_read_only = ON;
EXPLAIN (FORMAT TEXT) SELECT ...;
-- If actual timings needed:
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) SELECT ...;
```

## Workflow

1. **Find connection string** from `.env` or ask the user
2. **Verify connectivity** and confirm read-only mode
3. **Explore schema** -- start broad (list tables), then narrow down based on what the user needs
4. **Query data** -- always with `LIMIT`, always read-only
5. **Summarize findings** -- present results concisely, highlight what's relevant to the user's question

When the user's question is vague (e.g., "show me what's in the database"), start with the schema overview (tables, row counts, relationships) and let them guide you deeper. Don't dump everything at once -- explore incrementally.
