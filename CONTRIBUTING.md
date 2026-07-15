# Contributing

Changes must preserve the dependency direction and explicit activation rules in
`AGENTS.md`. Python is managed exclusively with `uv`.

```bash
uv sync --locked
uv run pytest -c pyproject.toml -q -p no:cacheprovider
uv run ruff check --config pyproject.toml .
uv run ruff format --check --config pyproject.toml .
uv run pyright --project .
uv build
```

Tool names, schemas, result shapes, execution facts, and risk facts are public API.
Add or change them together with focused tests and documentation. Portable runtime
contract changes belong in [`Ezio2000/jharness`](https://github.com/Ezio2000/jharness)
before implementation projects consume them.
