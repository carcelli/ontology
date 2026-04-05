# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

`ontology` is a standalone, domain-agnostic ontology kernel — a typed knowledge graph engine designed to power:

- **Polymarket agents + swarm intelligence** (primary consumer: `onto-market`)
- **ML inference** (feature extraction, priors, Brier scoring)
- **Multi-domain support** (prediction markets, crypto, NBA, politics via plugins)
- **OpenClaw integration** (typed KG with namespaced triples)

This is a **greenfield rebuild** — not a migration. The reference implementation lives at `/home/orson-dev/projects/onto-market/` (specifically `onto_market/ontology/graph.py`), but this repo fixes its architectural problems: hardcoded NetworkX backend, manual registry, domains mixed into the kernel, no thread safety.

## Python Environment

```bash
conda activate onto-market
pip install -e ".[dev]"
```

Requires Python >= 3.11.

## Commands

```bash
# Tests
python -m pytest tests/ -x -q

# Single test
python -m pytest tests/test_schema.py -x -q

# Type checking
mypy src/

# Lint (once configured)
ruff check src/ tests/
```

No build, test, or lint commands are configured yet — add them alongside the first toolchain commit and update this section.

## Architecture

### Design Principles

1. **Kernel purity** — domains consume the kernel, never modify it. Domain plugins live outside (e.g. in `onto-market/domains/`), discovered automatically via `importlib.metadata.entry_points(group="ontology.plugins")`.
2. **Pluggable backends** — Kuzu (embedded, default) → Neo4j (production scale). Swap engines without changing agent or swarm code.
3. **Pydantic v2 everywhere** — typed schemas with validation replace raw dataclasses. Prevents silent predicate coercion bugs from the original `graph.py`.
4. **Thread safety** — RLock on all mutations, not just persistence.
5. **Namespaced triples** — every entity is namespaced (`polymarket:Market`, `nba:Player`, `crypto:Token`).

### Target Directory Layout

```
src/ontology/
├── __init__.py
├── protocols.py          # Abstract interfaces (OntologyBackend, DomainPlugin, EnricherProtocol)
├── schema.py             # Pydantic v2 models: Triple, Entity, Relation, Namespace
├── namespace.py          # Namespace registry + validation (polymarket:*, nba:*, etc.)
├── graph.py              # Facade — delegates to current backend
├── registry.py           # Auto-discovery via entry_points, no manual edits
├── enricher.py           # Domain-agnostic enrichers only (HubDecomposer = topological)
├── events.py             # Lightweight pub/sub event bus for reactive enrichment
├── backends/
│   ├── __init__.py       # Default = Kuzu
│   ├── networkx.py       # Lightweight fallback / test backend
│   ├── kuzu.py           # Default backend (embedded, Python-native, fast)
│   └── neo4j.py          # Production scale (stub until needed)
└── config.py             # Kernel configuration
tests/                    # Mirrors src/ontology/ structure
```

### Key Interfaces (protocols.py)

All interfaces use `typing.Protocol` — plugins never need to import base classes:

- `OntologyBackend` — graph storage operations (add/query/delete triples)
- `DomainPlugin` — domain registration + enrichment hooks
- `EnricherProtocol` — triple enrichment pipeline

### Plugin System

Domains register via `pyproject.toml` entry points:

```toml
[project.entry-points."ontology.plugins"]
polymarket = "onto_market.domains.polymarket:PolymarketPlugin"
```

`pip install -e .` auto-registers — no touching kernel files.

### Event Bus (events.py)

When a market is upserted → kernel fires event → all registered domain enrichers can react automatically. Lightweight pub/sub, no external dependencies.

### Backend Selection

- **Kuzu** (default): embedded, Python-native, blazing fast for millions of triples
- **NetworkX**: in-memory fallback for tests and lightweight usage
- **Neo4j**: production cluster scale (stub, wire when needed)

## Relationship to onto-market

`onto-market` is the primary consumer. Its agents (`memory_agent`, `planning_agent`, swarm oracle) will import `ontology` as a dependency. The migration path:
1. Build kernel here with clean interfaces
2. `onto-market` adds `ontology` as a dependency in `pyproject.toml`
3. Replace `onto_market/ontology/graph.py` with calls to the kernel facade
4. Move domain-specific enrichers to `onto-market/domains/`

## Conventions

- Spaces, not tabs; lowercase filenames and directories
- All imports use the `ontology.*` prefix (e.g. `from ontology.schema import Triple`)
- Wrap HTTP calls with `@retry_with_backoff` (from shared utils)
- Commit messages: short, imperative (`Add schema validation`, `Wire Kuzu backend`)
- Keep commits focused on one change
