# Repository Guidelines

## Scope

This repository publishes the optional `jharness-tools` distribution, imported as
`jharness.tools`. It contains curated, replaceable tool implementations for JHarness
agents. The initial scaffold intentionally contains no preset implementations.

The stable runtime and extension contracts live in `Ezio2000/jharness-python`.
Portable behavior lives in the language-neutral `Ezio2000/jharness` specification.

## Architecture

- `jharness-tools` may depend on public `jharness.kernel` and `jharness.toolkit` APIs
  when source imports require them.
- Kernel, toolkit, and providers never depend on this project.
- Installing this distribution must not discover, register, or activate tools.
- Applications explicitly construct the presets they choose.
- Do not add `src/jharness/__init__.py`; `jharness` is an implicit PEP 420 namespace.
- Model-visible names, schemas, execution facts, and risk facts are public contracts.
- Security boundaries such as filesystem roots and network allowlists are Host-owned
  constructor configuration, not model-controlled arguments.

## Development

Python must be managed with `uv`; never use `pip`.

```bash
uv sync --locked
uv run pytest -c pyproject.toml -q -p no:cacheprovider
uv run ruff check --config pyproject.toml .
uv run ruff format --check --config pyproject.toml .
uv run pyright --project .
uv build
```

Do not claim completion until tests, lint, formatting, strict types, package builds,
metadata checks, and isolated wheel imports pass.
