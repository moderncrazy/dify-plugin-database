# Repository Guidelines

## Project Structure & Module Organization

This repository is a Python Dify database plugin. The runtime entry point is `main.py`. Plugin metadata lives in `manifest.yaml`, with provider and endpoint declarations in `provider/*.yaml` and `endpoints/*.yaml`.

Core implementation files are organized by capability:

- `provider/database.py`: credential validation for database connections.
- `tools/*.py`: tool implementations such as SQL execution, table schema lookup, text-to-SQL, and CSV querying.
- `tools/*.yaml`: Dify tool schemas paired with the matching Python module.
- `endpoints/sql_execute.py`: HTTP endpoint behavior.
- `_assets/`: marketplace icons and README images.
- `test.db`: SQLite sample database for manual checks.

## Build, Test, and Development Commands

- `pip install -r requirements.txt`: install plugin dependencies.
- `cp .env.example .env`: create local debug configuration.
- `python -m main`: run the plugin locally for Dify remote debugging.
- `dify-plugin plugin package .`: build a `.difypkg` package.

Use Python 3.10+ for local work; the plugin manifest targets the Python runner used by Dify.

## Coding Style & Naming Conventions

Follow the existing Python style: 4-space indentation, type hints on public tool/provider methods, and concise exceptions for user-facing validation errors. Keep tool class names descriptive, for example `SQLExecuteTool`, and align schema filenames with implementations, such as `tools/sql_execute.py` and `tools/sql_execute.yaml`.

When adding parameters, update both the Python implementation and the corresponding YAML schema. Avoid logging or committing database credentials, connection strings, or generated result files.

## Testing Guidelines

There is no dedicated automated test suite in this repository. For changes, run `python -m main` and validate through a Dify debug install. Prefer `sqlite:///test.db` for smoke tests, and manually verify affected tools with simple queries such as `SELECT 1`.

For URI parsing or utility changes, add doctest-style examples or a focused test file if introducing a test framework.

## Commit & Pull Request Guidelines

Recent commits use short, imperative, lowercase messages, for example `fix db uri special char encoding` or `add config options to all tools`. Keep commits focused on one behavior change.

Pull requests should include a brief description, affected tools or endpoints, manual verification steps, and any relevant issue link. Include screenshots only when README assets, marketplace metadata, or visible Dify UI behavior changes.

## Security & Configuration Tips

The SQL execution tool can run arbitrary SQL. Test with read-only database accounts whenever possible. Keep `.env` local, and use `.env.example` only for non-secret configuration names.
