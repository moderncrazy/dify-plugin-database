# SQL Execute Read Transaction Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ensure `sql_execute` read queries end their PostgreSQL transaction after result export.

**Architecture:** Keep the existing `SQLExecuteTool` structure and output formatting. Replace only the `SELECT/WITH` branch's `db.query()` shortcut with explicit records connection and transaction management.

**Tech Stack:** Python 3, Dify plugin SDK, records 0.6.0, SQLAlchemy.

---

## File Structure

- Modify: `tools/sql_execute.py`
  - Responsibility: execute configured SQL, format query results, and manage database connection lifecycle.
- No YAML, endpoint, or other tool changes.

### Task 1: Update Read Query Transaction Handling

**Files:**
- Modify: `tools/sql_execute.py`

- [ ] **Step 1: Inspect the current read branch**

Run:

```bash
sed -n '1,120p' tools/sql_execute.py
```

Expected: the `SELECT/WITH` branch calls `rows = db.query(query)` before formatting output.

- [ ] **Step 2: Replace `db.query()` with explicit transaction handling**

In `tools/sql_execute.py`, replace the read branch body with this structure:

```python
            if re.match(r'^\s*(SELECT|WITH)\s+', query, re.IGNORECASE):
                with db.get_connection() as conn:
                    trans = conn._conn.begin()
                    try:
                        rows = conn.query(query, fetchall=True)
                        if format == "json":
                            result = rows.as_dict()
                            message = self.create_json_message({"result": result})
                        elif format == "md":
                            result = str(rows.dataset)
                            message = self.create_text_message(result)
                        elif format == "csv":
                            result = rows.export("csv").encode()
                            message = self.create_blob_message(
                                result, meta={"mime_type": "text/csv", "filename": "result.csv"}
                            )
                        elif format == "yaml":
                            result = rows.export("yaml").encode()
                            message = self.create_blob_message(
                                result,
                                meta={"mime_type": "text/yaml", "filename": "result.yaml"},
                            )
                        elif format == "xlsx":
                            result = rows.export("xlsx")
                            message = self.create_blob_message(
                                result,
                                meta={
                                    "mime_type": "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
                                    "filename": "result.xlsx",
                                },
                            )
                        elif format == "html":
                            result = rows.export("html").encode()
                            message = self.create_blob_message(
                                result, meta={"mime_type": "text/html", "filename": "result.html"}
                            )
                        else:
                            raise ValueError(f"Unsupported format: {format}")
                        trans.commit()
                        yield message
                    except Exception:
                        trans.rollback()
                        raise
```

This keeps current output formats but commits before yielding the message.

- [ ] **Step 3: Run syntax verification**

Run:

```bash
.venv/bin/python -m py_compile tools/sql_execute.py
```

Expected: command exits successfully with no output.

- [ ] **Step 4: Verify no accidental scope expansion**

Run:

```bash
git diff -- tools/sql_execute.py
```

Expected: diff only changes the `SELECT/WITH` branch in `SQLExecuteTool._invoke`; non-read execution logic remains unchanged.

- [ ] **Step 5: Commit implementation**

Run:

```bash
git add tools/sql_execute.py
git commit -m "fix sql execute read transaction"
```

Expected: one commit containing only `tools/sql_execute.py`.

## Self-Review

- Spec coverage: the plan implements explicit read-query transaction handling, preserves output formats, and leaves non-read paths unchanged.
- Placeholder scan: no placeholder tasks remain.
- Type consistency: the plan uses existing `records.Connection.query`, `trans.commit()`, `trans.rollback()`, and existing Dify message constructors.
