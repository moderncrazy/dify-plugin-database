# SQL Execute Read Transaction Design

## Problem

`tools/sql_execute.py` uses `records.Database.query()` for `SELECT` and `WITH` queries. In `records 0.6.0`, that helper opens a SQLAlchemy connection with `close_with_result=True`; the context manager exits without closing the underlying connection and relies on result consumption or garbage collection. With PostgreSQL this can leave sessions in `idle in transaction` after read queries, holding pooled connections longer than intended.

An observed session showed a completed read query stuck as `idle in transaction` for several minutes.

## Scope

Only the `SQLExecuteTool` read-query branch should change:

- Queries matching `^\s*(SELECT|WITH)\s+`
- Existing output formats: `json`, `md`, `csv`, `yaml`, `xlsx`, `html`

The non-read execution branch, endpoint code, YAML schemas, and other tools stay unchanged.

## Design

Replace the read branch's direct `db.query(query)` call with explicit connection and transaction management:

1. Open a records connection with `with db.get_connection() as conn:`.
2. Start a transaction using the underlying SQLAlchemy connection.
3. Execute the query through `conn.query(query, fetchall=True)` so the result is fully consumed before the connection exits.
4. Convert the result to the requested output format while still inside the transaction.
5. Commit after successful conversion, then yield the Dify message.
6. Roll back and re-raise on query or conversion errors.
7. Keep `db.close()` in the existing `finally` block so the engine is disposed.

This preserves the existing records result formatting while avoiding the `Database.query()` shortcut that can leave the connection open.

## Connection Pool Context

`records.Database(db_uri, **config_options)` passes options directly to `sqlalchemy.create_engine()`. With default PostgreSQL/psycopg2 settings, SQLAlchemy uses `QueuePool` with `pool_size=5`, `max_overflow=10`, `pool_timeout=30.0`, and `pool_recycle=-1`.

## Risks

Large result sets still keep a transaction open while the output is exported, especially for file formats such as `xlsx`. This is already true for the current implementation because exports require consuming the full result. The fix ensures the transaction ends immediately after export instead of relying on result cleanup.

The change must preserve current output shapes exactly. Using `records.Connection.query()` rather than raw SQLAlchemy rows reduces that compatibility risk.

## Verification

- Run a focused syntax/import check for `tools/sql_execute.py`.
- If a PostgreSQL test database is available, execute a `SELECT` through the tool and confirm `pg_stat_activity` no longer shows the session as `idle in transaction` after the response is produced.
- Smoke-test at least one structured output (`json`) and one exported output (`csv` or `xlsx`).
