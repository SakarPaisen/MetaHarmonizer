# Contributing to MetaHarmonizer

This is the contributor-facing half of the docs. Installation (conda env, the
FAISS-from-conda-forge rule on macOS, the extras matrix) is README
§[2. Installation](README.md#2-installation), and every environment variable
plus the config precedence chain is README
§[3. Environment variables](README.md#3-environment-variables). Both are
linked here, not restated. This is a library, not a deployment.

## Workflow

1. Branch from the latest `main`.
2. Set up (README §2), plus the dev extra:

   ```bash
   pip install -e ".[dev]"     # pytest & pytest-cov
   pip install faiss-cpu       # Linux / Windows / CI; on macOS use conda-forge (README §2)
   ```

3. Make your change inside the module that owns it (see the map below).
4. `pytest tests/` green before you push.
5. Open a PR against `main`.

## Running the suite

`pytest tests/` (24 test files) is green on a clean clone with **no API key,
no network, and no corpus data on disk** — that is exactly how CI runs it
(`.github/workflows/ci.yml`: Python 3.10 and 3.11, `pip install -e ".[dev]"`,
`pip install faiss-cpu`, `pytest tests/ -v`). Where the boundaries are:

| needs | in the default suite | where it actually matters |
|---|---|---|
| API keys (`GEMINI_API_KEY`, `UMLS_API_KEY`, OpenAI) | **nothing** — every LLM/API boundary is stubbed | real engine runs |
| network | **nothing** | corpus builds, OLS/NCI/UMLS lookups |
| corpus data on disk | **nothing** — path resolution is tested, not corpus contents | end-to-end runs, example notebooks |
| `faiss` importable | some ontology-mapper / stage tests import it | install `faiss-cpu` first (macOS rule in README §2) |

Nothing is conditionally skipped: tests that touch key-gated code construct
around the gate. Examples: `tests/test_sm_stage4_matchers.py` builds
`OntoMapLLM` via `__new__` + stub injection to bypass the `GEMINI_API_KEY`
check in `__init__`; `tests/test_knowledge_db_portability.py` exercises archive
export/import with stub blobs so no `UMLS_API_KEY` or built corpus is required.
If a test fails for you on a clean clone with only `.[dev]` + faiss installed,
that is a bug in the suite or in this document — please report it.

What you *cannot* do without data on disk is run the engines end-to-end (see
below).

## Lint expectations

`ruff` is configured in `pyproject.toml` (`E,F,W`, `line-length = 99`,
per-file `E741` exemptions for the canonical FAISS `D, I` names) but is **not
run in CI**, and `ruff check .` currently reports on the order of a hundred
errors in existing code. The bar for a PR today:

- your diff must not add new lint errors — run `ruff check` on the files you
  touched;
- don't fix unrelated lint in the same PR (it buries the change);
- full-repo clean-up is welcome as its own, purely-lint PR.

## Data on disk

The full ontology corpus is **not bundled in the wheel** — only small reference
files ship in `metaharmonizer/_bundled_data/`, which surprises people on their
first end-to-end run. For local work:

- `METAHARMONIZER_DATA_DIR` (default `~/.metaharmonizer/data`) holds corpus +
  schema reference files; `KNOWLEDGE_DB_DIR` (default
  `~/.metaharmonizer/KnowledgeDb`) holds the SQLite vector DB + FAISS indexes.
- `python -m metaharmonizer.scripts.bootstrap_data` copies the repo's `data/`
  tree (e.g. the EFO phenotype corpus) into the active `METAHARMONIZER_DATA_DIR`;
  idempotent, `--force` to overwrite.
- `python -m metaharmonizer.scripts.knowledge_db export` / `import` moves a
  built knowledge base as one sha256-verified archive — build it once with a
  `UMLS_API_KEY`, or import one someone else built.
- The engines fetch or build what's missing on first run when keys are
  available; everything else resolves through `metaharmonizer._paths` and is
  overridable by env var (README §3).

You need a corpus on disk for end-to-end engine runs and the notebooks; you do
not need one to write or run tests.

## Module map

Where a change belongs, one line per package under `src/metaharmonizer/`:

- `engine/` — `OntoMapEngine`: wires the ontology-mapping stages (exact →
  semantic → synonym) into one pipeline.
- `models/` — the mapping methods: `ontology_mapper_*` (LM, ST, bi-encoder,
  RAG, LLM, synonym), `reranker.py`, and `schema_mapper/` (`SchemaMapEngine`
  and its stage 1–4 matchers, alias-dict generation).
- `knowledge_db/` — corpus and concept-table builders, OLS/NCI/UMLS clients,
  the FAISS + SQLite vector pipeline, synonym dict.
- `field_suggester/` — suggests new harmonized fields for unmapped columns
  (NER + embedding clustering).
- `utils/` — embeddings, vector store, model cache/loader, LLM client, shared
  match helpers.
- `scripts/` — data bootstrap and knowledge-DB archive export/import.
- `settings.py` — config resolution (`arg > env > project file > default`);
  `_paths.py` — all on-disk path resolution; `_bundled_data/` — the small
  reference files that ship in the wheel.

## Downstream breakage

This package is wrapped by two consumers: **MetaHarmonizerR** (through
reticulate) and **MetaHarmonizerApp** (only through
`backend/app/engine_adapter/`). A change to the public API of `OntoMapEngine` /
`SchemaMapEngine` — constructor signature, method names, returned DataFrame
columns — breaks both in ways this repo's tests will not catch. If your change
touches that surface, say so explicitly in the PR and expect extra review.

## PR conventions

- Branch from latest `main`; one topic per PR.
- Behavior changes come with tests; keep the diff inside the owning module.
- No unrelated refactors, reformatting, or dependency bumps in the same PR.
- `pytest tests/` green is the merge bar; the lint bar is as stated above.
- Bugs and genuinely broken things you hit along the way go in separate issues
  (and a friction-log comment on your PR is just as welcome).