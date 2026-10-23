# Agent Instructions: QABuddy.ai — multi-source hybrid RAG for QA

QABuddy answers QA questions (RCA, test design, triage, framework help, RTM) with
**cited** answers over 10 knowledge sources. One FastAPI process serves the API and the
built React UI on **port 8300**.

## Tech Stack (what is actually used)

- **Language & Runtime:** Python **3.13** (this machine installs it machine-wide; `.venv`
  is optional). Windows-first; `run.sh` is the bash/macOS/Linux path.
- **Package management:** `requirements.txt` (no `pyproject.toml`). `uv` is optional —
  never assume it is installed.
- **API + UI:** FastAPI + uvicorn (`qabuddy/api.py`, `qabuddy/__main__.py serve`) and a
  React 18 + Vite 6 app in `ui/` built to `ui/dist`, served by the same process.
- **Vector DB:** Qdrant (local `:6333` or Qdrant Cloud). One collection, two named vectors
  per point: `dense` (cosine, 1024-d) and `bm25` (sparse, IDF modifier).
- **Embeddings:** `qwen3-embedding:0.6b` via **Ollama** (`:11434`), instruction-aware
  query prefix, 1024 dims (`EMBED_DIM`).
- **Sparse retrieval:** in-process code-aware BM25 (`qabuddy/sparse.py`), not a library —
  it preserves `TimeoutException`, `VWO-26`, `WaitHelpers.java`, camelCase identifiers.
- **Reranker:** `sentence-transformers` cross-encoder
  `cross-encoder/ms-marco-MiniLM-L-6-v2`, in-process (`qabuddy/rerank.py`).
- **Answer LLM:** Groq `openai/gpt-oss-120b` by default; `LLM_PROVIDER` also supports
  `ollama` and `openai` (any OpenAI-compatible endpoint).
- **Parsing:** pypdf (layout mode), openpyxl, tree-sitter (Java/TS/JS), snowballstemmer.
- **Not used:** no LangChain, LlamaIndex, FastMCP, Instructor, or Pydantic-LLM structured
  outputs. Do not introduce them or add imports from them.

## Critical Architectural Constraints (Do Not Violate)

1. **Identical chunking for dense and sparse.** Both vectors are written on the *same*
   Qdrant point from the same chunk (`qabuddy/models.py` `Chunk.id`). Never build a
   separate keyword index with different text or IDs; never fuse unmatched chunks.
2. **RRF only for fusion.** Combine dense and BM25 with Qdrant's native
   `FusionQuery(Fusion.RRF)` in `store.hybrid`. Never add cosine similarity to BM25
   scores. The dense-only and BM25-only queries exist only to show per-side ranks in the
   trace.
3. **Strict context grounding.** `qabuddy/answer.py` modes must keep the zero-hallucination
   rule: answer *only* from provided context; if the answer is missing, say the context is
   insufficient. Do not weaken this prompt.
4. **Always citations.** Every answer source carries source metadata (source id, file
   path, locator/page/lines/build/test id). Inline citations use `[1]`, `[2][3]` only —
   never `【】` or line ranges.
5. **Preserve retrieval structure.** Per-source quotas, pinned inventory/summary chunks,
   exact-ID lookup (`VWO-26`, `LOGIN-002`), the relevance floor (`RERANK_FLOOR`) and the
   token budget in `qabuddy/retrieve.py` are deliberate. Do not remove them to "simplify"
   ranking.
6. **Secrets stay in `.env`.** Never commit keys or print them; ingest redacts
   password/token/API-key values. `.env` and `.index/` are git-ignored.

## Agent Automation & Command Mappings

Run from the repository root (`C:\PramodAcademy\QABuddy`). Use these exact commands.

- **Environment check:** `python -V` (expect 3.13.x) and `python -m pip check`
- **Install deps (only if missing):**
  - With `uv`: `uv venv --python 3.13 .venv` then
    `uv pip install --python .venv\Scripts\python.exe -r requirements.txt pytest`
  - Without: `python -m venv .venv`; activate; `pip install -r requirements.txt pytest`
  - Machine-wide installs need an elevated shell, or use `pip install --user <pkg>`.
- **Build the UI (only when `ui/` changed):** `cd ui; npm install; npm run build`
- **Ingest / index:** `python -m qabuddy ingest` (incremental; `--full` rebuilds;
  `--source <id>` for one source; `--force` re-embeds unchanged files)
- **Serve API + UI:** `python -m qabuddy serve` → http://localhost:8300
- **Health check:** `curl http://localhost:8300/api/health` (want `qdrant`, `ollama`,
  `embed_model_ready`, `llm_configured` true and non-zero `points`)
- **Retrieval eval:** `python -m qabuddy eval` (ablation table on `eval/golden.yaml`)
- **Regression tests:** `python -m pytest -q tests` (headless; no Qdrant/Ollama/LLM needed)
- **CLI ask/search:** `python -m qabuddy ask "..." --mode rca`,
  `python -m qabuddy search "..."`
- **Jira sync:** `python -m qabuddy sync-jira --jql "project = VWO AND type = Bug"`
- **UI hot reload (optional):** `cd ui; npm run dev` → `:5300`, proxies `/api` to `:8300`
- **Bash helper (mac/linux/WSL):** `./run.sh`, `./run.sh ingest`, `./run.sh test`,
  `./run.sh eval`, `./run.sh ask "..."`

Service prerequisites: Ollama running with `qwen3-embedding:0.6b` pulled, and Qdrant
reachable (`QDRANT_URL`). On Windows, `./run.sh` fails (no Windows Qdrant branch) — use the
`python -m qabuddy` commands, or run it under WSL/Git Bash.

## Project Layout

```
qabuddy/
  __main__.py   CLI dispatch (ingest, ask, search, eval, sync-jira, export-demo, serve)
  api.py        FastAPI routes + static UI on one port
  config.py     .env settings, sources.yaml, glossary.yaml
  ingest.py     source-aware chunking, sha256 manifest, incremental upsert
  chunkers/     testcases, jira, docs, transcript, diagram, logs, code, figma
  embed.py      Ollama Qwen3 embeddings        sparse.py  code-aware BM25
  store.py      Qdrant collection + hybrid RRF  retrieve.py  pipeline + selection
  rerank.py     cross-encoder                   answer.py  modes, prompts, citations
  llm.py        Groq/Ollama/OpenAI providers    evaluate.py, demo_export.py, jira_sync.py
ui/             React + Vite app (dist is built output served by FastAPI)
data/           10 source folders (sources.yaml maps folders -> chunkers)
eval/           golden.yaml + last_run.json
tests/          test_pipeline.py (pytest, no external services)
deploy/         Dockerfile, compose, DEPLOY.md
```

## Operational Guardrails & Preferences

- **Before a fix:** run `python -m pytest -q tests` to establish a green baseline.
- **After a change to chunkers, embedding, or retrieval:** re-run `python -m qabuddy ingest`
  (or `--full` if `EMBED_MODEL`/`EMBED_DIM` changed) and check `/api/health` `points`.
- **Before editing DB code:** verify the collection schema in `qabuddy/store.py`
  (`create_collection`, `PAYLOAD_INDEXES`); dense is 1024-d cosine, `bm25` is sparse/IDF.
- **Dependency management:** prefer `requirements.txt`; with uv use `uv pip install` /
  `uv add` only if a `pyproject.toml` exists (currently it does not). Never use a bare
  `pip install` that would modify the machine-wide install without elevation/`--user`.
- **Windows:** use `python`/`py -3.13`, not `python3`; avoid bash-only commands in edits.
- **Logging:** prefer the `logging` module over `print()` in library code (the CLI may
  print). Keep terminal output dense and minimal.
- **Tests:** add regression tests to `tests/` for any bug fixed; they must run without
  Qdrant, Ollama, or an LLM key.
- **Do not** rename ports casually: `PORT` default 8300 is load-bearing for the UI and docs.
