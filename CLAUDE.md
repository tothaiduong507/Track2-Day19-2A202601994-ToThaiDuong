# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A VinUni AICB course lab (Day 19, Track 2): a hybrid search API (BM25 + vector + RRF) backed by
Qdrant, plus a Feast feature store, taught through 8 Jupytext notebooks. The `app/` package is the
production-shaped code the notebooks import; the notebooks are the graded deliverable. Two
interchangeable runtime paths exist — **lite** (in-memory Qdrant + SQLite Feast, no Docker) and
**Docker** (Qdrant server + Redis + Postgres) — selected entirely through `.env`, with no code
branching beyond what already lives in `app/`.

## Commands

```bash
make setup-lite      # venv + deps + seed corpus + smoke test (~60s) — do this first
make api             # FastAPI /search on :8000 (uvicorn --reload)
make lab             # Jupyter Lab on :8888 (auto-converts notebooks/*.py -> .ipynb via jupytext)
make benchmark       # Precision@10 + P99 latency table (scripts/benchmark.py)
make test            # pytest -q (app/ + scripts/, ~2s, 34 tests)
make seed            # regenerate data/corpus_vn.jsonl + data/golden_set.jsonl (deterministic)
make gen-advanced    # regenerate NB6 agent-query data + NB8 spend parquet
make notebooks       # execute ALL notebooks headless — this is what the grader runs
make clean-lite      # wipe venv + data + Feast registry/online-store

make runtime-check   # report which of Docker/Podman/Apple-container is available
make setup-docker    # bring up docker-compose stack + venv + seed
make verify-docker   # confirm Qdrant/Redis/Postgres reachable + Feast wired
```

Run a single test: `.venv/bin/pytest tests/test_filters.py -q` (or `-k name` for one test).
There is no lint/format command wired into `make` — `ruff` config exists in `pyproject.toml`
(line-length 100, target py310) but is not invoked by any make target.

Notebooks are Jupytext `.py` files under `notebooks/` — **that `.py` file is the source of truth**;
the `.ipynb` is a generated artifact kept in sync by `jupytext --to notebook --update`. Edit the
`.py` (or edit the `.ipynb` in Jupyter, which jupytext syncs back).

## Architecture

### Two runtime paths, one codebase

`QDRANT_MODE` (`memory`|`server`) and `EMBEDDING_BACKEND` in `.env` are the only switches between
lite and Docker — `app/search.py` and `app/feast_repo/` read them at runtime rather than branching
in notebook code. Changing `EMBEDDING_BACKEND` changes the vector dimension (384/1024/1536), which
means the Qdrant collection must be rebuilt — there is no migration path, just re-run from NB1.

`app/embeddings.py`'s `Embedder` is the single seam for all four backends (`fastembed`,
`multilingual`, `bge-m3`, `openai`); `app/search.py` reads `Embedder().dim` rather than
hard-coding a dimension constant, specifically so switching backends doesn't require touching
`Searcher`.

### Core retrieval (`app/search.py`)

`Searcher` holds a BM25 index (`rank_bm25`) and a Qdrant collection side by side, built once at
`Searcher.from_corpus()` and reused (construction is expensive — model load + embedding 1000
docs). `mode="hybrid"` fuses BM25 and vector results with **Reciprocal Rank Fusion (RRF, k=60)**:
pull `top_k*5` (min 50) candidates from each retriever, score by `1/(k + rank)` with **1-based
rank**, sum across retrievers. This RRF formula and its 1-based-rank detail recur across
`app/search.py`, `app/agent.py`, and are the #1 NB2 debugging trap called out in the README.

`app/main.py` wraps `Searcher` in FastAPI with a lifespan hook that builds the searcher once at
startup (module-level `_searcher` global, intentional — rebuilding per-request would defeat the
P99 < 50ms rubric target).

### Advanced-mission layer (NB5–NB8), built on top of the same index

- **`app/metadata.py`** — deterministic per-doc metadata (`tenant`, `access`, `published_ts`)
  derived from `doc_id` via `blake2b` hash, *not* generated during corpus seeding. This is
  intentional: adding fields inside `seed_corpus.py`'s RNG loop would shift every downstream draw
  and invalidate the golden set. Same `doc_id` always yields the same metadata everywhere.
- **`app/filters.py`** — `FilteredIndex` clones the base Qdrant collection into a richer one
  (payload includes metadata.py's fields) and implements three filtered-search strategies side by
  side: `post_filter` (ANN then discard — recall cliff under selective filters), `pre_filter`
  (exact brute-force scan over the matching subset), `filtered_ANN` (filter pushed into the
  engine). It reuses vectors already computed by `Searcher` via `client.scroll(with_vectors=True)`
  rather than re-embedding — embedding is the expensive step in this lab.
- **`app/agent.py`** — retrieval-as-a-tool. `RuleBasedPlanner` decomposes multi-intent Vietnamese
  questions (split on "và"/"hoặc"/"so với"/etc.), infers topic/year filters from keyword hints,
  and `Agent.answer()` retries once with filters relaxed if a filtered call returns too few
  results ("a filter that returns almost nothing is a bad filter, not an empty corpus"). The
  planner is deliberately rule-based, not an LLM call, so the lab needs no API key.
  `build_context()` is where retrieval and Feast meet: it pulls `topic_affinity` from Feast online
  features to bias the search filter, degrading gracefully if Feast hasn't been applied yet.
- **`app/cache.py`** — semantic cache over a separate Qdrant collection with three independently
  tunable failure modes: `threshold` too low → false hit (wrong answer served), missing `ttl_s` →
  stale hit, `namespaced=False` → cross-tenant leak (a *security* bug, framed per OWASP LLM08).
  Uses a virtual clock (`advance()`) so TTL expiry is testable without real sleeps.
- **`app/features.py`** — synthetic event-log generator plus the six feature-engineering families
  (windowed aggregation, ratio-to-baseline, lag/delta, recency, categorical encoding, embedding-as-
  feature) and two intentional leakage demos: `target_encode_naive` (leaks) vs
  `target_encode_in_fold` (correct), and `latest_join` (wrong — ignores time) vs `pit_join`
  (point-in-time correct, via `pd.merge_asof`). `auc()` is a pure-numpy rank-based implementation
  so no sklearn dependency is needed.

### Feast feature store (`app/feast_repo/`, `app/feast_repo_ondemand/`)

Two separate Feast repos: `feast_repo` has three standard `FeatureView`s (`user_profile_features`,
`item_popularity_features`, `query_velocity_features`) sourced from Parquet files students
generate in NB4; `feast_repo_ondemand` has an `OnDemandFeatureView` (`amount_vs_avg`) combining a
stored feature with request-time data. Known gotcha documented inline in
`feast_repo_ondemand/definitions.py`: `from feast import on_demand_feature_view` imports the
*module*, not the decorator — import from `feast.on_demand_feature_view` explicitly. Each
`feature_store.yaml` has the Docker-mode (Redis/Postgres) config commented out alongside the
active lite-mode (SQLite/file) config — switching paths means uncommenting, not writing new config.
If `feast apply` errors, the standard fix is deleting `app/feast_repo/registry.db` and re-applying.

### Corpus & determinism

`scripts/seed_corpus.py` generates `data/corpus_vn.jsonl` (1000 Vietnamese docs across 10 topics)
and `data/golden_set.jsonl` (50 labeled queries) from a seeded RNG — fully reproducible, nothing
committed as static data. Tests rely on this: `tests/conftest.py`'s `mini_corpus` fixture skips
(doesn't fail) if `data/corpus_vn.jsonl` is missing, and slices out whole topic clusters (docs with
index < 6 per topic) so filter-selectivity tests still have matches. Run `make seed` before running
tests fresh.

## Conventions specific to this repo

- Every non-obvious design choice in `app/*.py` is explained in a module or function docstring —
  read them before changing behavior; several encode fixes for real bugs found while building the
  lab (e.g. `embeddings.py`'s docstring explains why `EMBEDDING_BACKEND` used to be silently
  ignored). Preserve this pattern: when you fix a subtle bug here, say why in a comment, since the
  whole repo is pedagogical and students read this source.
- Tokenization for BM25 is naive whitespace-lowercase (`text.lower().split()`) — flagged in
  `search.py` as a known simplification for mixed VI/EN text, not a bug to silently fix.
  `VIBE-CODING.md` explicitly calls out which subtasks are meant to be delegated to AI vs.
  hand-designed by the student; don't casually "improve" things flagged as intentional teaching
  simplifications.
- Notebooks NB1–NB4 are graded core (100 pts); NB5–NB8 are graded advanced (50 pts). Changes to
  `app/` that alter numeric outputs (RRF scores, recall values, latency) can invalidate rubric
  thresholds described in `rubric.md` and the per-notebook "Pass when…" table in `README.md` —
  check both before changing scoring-relevant logic.
