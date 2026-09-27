# Stash Competitive Teardown — vs. Hyperspell

> Written from source inspection of `~/dev/stash-competitor/`. Claims are grounded in
> specific files; conjecture is flagged. Last read: July 10, 2026.

---

## 1. TL;DR

- **Stash is an agent-session archive + writable Markdown Drive** with connected-source connectors
  bolted on; the knowledge base is a secondary artifact built by a scheduled curator agent, not the
  primary product surface. (`ARCHITECTURE.md`, `backend/services/prompts.py → render_curator_prompt`)
- **Their core architectural bet is the VFS shell**: exposing the entire knowledge base as a
  POSIX-like filesystem (`ls`, `cat`, `find`, `grep`, pipes) that any coding agent can use without
  learning a new API. (`stashvfs/`, `backend/routers/vfs.py`)
- **Their biggest single difference from Hyperspell**: Stash does NOT have a pre-synthesized,
  continuously-maintained knowledge tree. The "brain" is assembled on demand by a scheduled
  Claude-agent curator that reads raw session events + pages and writes wiki pages into a special
  Memory folder on a daily cadence. Hyperspell's Temporal tree-generation is a durable background
  workflow that runs continuously and produces a richly-structured OKF-conformant document tree with
  per-sentence citations. Stash's wiki is Markdown pages in a folder, linked by regex-parsed
  `[text](/p/<uuid>)` markdown links. (`backend/services/curation_service.py`,
  `backend/services/prompts.py → render_curator_prompt`)
- **They are ahead on surface area**: multi-agent-host plugins (Claude Code, Cursor, Codex,
  OpenCode, Gemini CLI, Openclaw), a writable collaborative Markdown editor with real-time Yjs sync,
  an operable VFS shell, a Skills publish/fork system, and a cloud VM "sprite" per user for running
  agent workloads. (`README.md`, `plugins/`, `backend/routers/collab.py`, `backend/services/sprite_service.py`)
- **Hyperspell is ahead on retrieval quality and provenance**. Stash retrieval is FTS-only or
  vector-only (never fused in the code I can find); no reranking layer exists; no recency half-life;
  no MMR diversity; no per-sentence citation provenance; no redaction-on-read. Its knowledge
  synthesis is a prompt instructed to "maintain the wiki" rather than a deterministic structured
  workflow.

---

## 2. Their Stack

**Language / runtime**
- Backend: Python 3.12, FastAPI (`backend/`), asyncpg for DB, Alembic migrations.
- Frontend: Next.js (TypeScript) on port 3457. (`CLAUDE.md`)
- CLI + MCP: Python, Typer, distributed as `stashai` on PyPI. (`cli/main.py`, `pyproject.toml`)
- VFS: Pure-Python `stashvfs` package (included in-repo) exposed both in CLI and via REST.
  (`stashvfs/`, `backend/routers/vfs.py`)
- Collab: Yjs WebSocket sidecar (TypeScript, `collab/`), backed by the FastAPI auth layer.

**Datastores**
- PostgreSQL 16 with `pgvector` extension for all vector storage. Images: `pgvector/pgvector:pg16`.
  (`docker-compose.yml`)
- Redis: task queue (Celery broker + result backend) and presence/pubsub. (`backend/celery_app.py`,
  `docker-compose.yml`)
- S3-compatible object store (AWS S3, Cloudflare R2, or MinIO) for file binaries. Optional — uploads
  disabled if unset. (`.env.example`)
- No dedicated vector DB (Qdrant, Weaviate, etc.); everything lives in Postgres pgvector columns
  on `pages.embedding`, `history_events.embedding`, `files.embedding`, `table_rows.embedding`, and
  per-source-type content tables. (`backend/tasks/embeddings.py`, `backend/migrations/versions/0001_initial_schema.py`)

**Embedding / ML**
- Pluggable provider with auto-detection order: OpenAI-compat API → Hugging Face Inference API →
  local `sentence-transformers`. (`backend/services/embeddings/auto.py`)
- Default local model: `all-MiniLM-L6-v2` (384-dim). OpenAI default when key is set:
  `text-embedding-3-small`. HF default: `BAAI/bge-small-en-v1.5`. (`.env.example`)
- Text clipped to 6,000 chars before embedding; no chunking layer — whole documents are embedded
  as one unit. (`backend/services/embeddings/__init__.py`, line `MAX_TEXT_CHARS = 6_000`)
- **No reranking model exists** in the codebase. Not found anywhere.

**LLM**
- Anthropic-only: Claude Sonnet (quality tier) + Claude Haiku (fast tier). `claude-agent-sdk` is
  used for the sprite/cloud-VM agent harness; the ask-the-stash loop bypasses it and calls Anthropic
  directly via the native tool-use API. (`backend/services/llm.py`, `backend/services/tool_loop.py`)
- Default models: `claude-sonnet-4-6` / `claude-haiku-4-5`. (`.env.example`, `docker-compose.yml`)
- LLM is optional. Without `ANTHROPIC_API_KEY`, ask-the-stash and session-title generation are
  disabled; the rest of the product works. (`README.md`)

**Worker / queue**
- Celery with Redis broker. Beat scheduler for periodic tasks (embedding reconciler every 60s,
  extraction queue scan every 60s, viz precompute every 300s, source sync every 120s, etc.).
  (`backend/celery_app.py`)
- File extraction isolated in subprocess (`python -m backend.workers.extract_one`) with
  `RLIMIT_AS` to bound OOM impact. (`backend/tasks/extraction.py`, `backend/workers/extract_one.py`)
- Cloud-agent execution via "Sprites" (sprites.dev persistent Linux VMs, one per user) or local
  subprocess for dev. (`backend/services/sprite_service.py`)

**Integrations (connected sources)**
- GitHub, Google Drive, Gmail, Notion, Slack, Granola, Jira, Asana, Linear, Gong, Twitter.
  (`backend/integrations/`, `.env.example`)
- Two source patterns: *copy* (github/slack/granola/gong/notion — content indexed in Postgres
  FTS+vector) and *index-only* (Drive/Gmail/Jira/Asana — metadata stored, body fetched lazily
  from provider). (`backend/services/source_service.py`, docstring top)

---

## 3. How They Do Retrieval & Synthesis

Stash has three separate retrieval modes that never merge:

### 3a. Full-Text Search (FTS)
- PostgreSQL `websearch_to_tsquery` with `ts_rank`. Implemented independently for each content
  type: `search_scope_events` (history events), `search_pages_fts` (pages), `search_documents`
  (connected source content tables). (`backend/services/memory_service.py → search_scope_events`,
  `backend/services/files_tree_service.py → search_pages_fts`)
- Page FTS weights metadata `keywords` (A weight) above body text (B weight) via `setweight +
  to_tsvector`. (`backend/services/files_tree_service.py → search_pages_fts`)
- **No hybrid fusion anywhere in the code.** FTS and vector endpoints are separate REST routes
  (`GET /pages/search` vs `GET /pages/semantic-search`) with no code path that calls both and merges.

### 3b. Vector Search
- `pgvector` cosine distance (`<=>`) operator. Separate functions for each content type:
  `search_pages_vector`, `search_scope_events_vector`, `search_personal_events_vector`.
  (`backend/services/files_tree_service.py → search_pages_vector`,
  `backend/services/memory_service.py → search_scope_events_vector`)
- Whole-document embeddings (up to 6k chars). No chunking, no sliding window, no late-interaction.
- **No reranking step** anywhere (confirmed by `grep -rn "rerank"` returning zero hits in backend/).

### 3c. Search-all (the agent's `search` tool)
- Dispatched by `source_service.search_all`. Fans out to: FTS on history events, FTS on pages,
  FTS on copied-source content tables, external-ref exact match, and federated live-search for
  sources that support it (Gmail, Slack, Gong, Jira, Asana via their provider APIs). Results are
  concatenated with no scoring merge or deduplication. (`backend/services/source_service.py →
  search_all`)
- No vector search path in `search_all` — the agent tool only sees FTS results.

### 3d. Synthesis ("ask-the-stash")
- One-shot Anthropic tool-use loop (up to 8 turns, `max_turns=8`). Model calls tools from
  `ASK_TOOL_SET`: `search_history`, `read_page`, `grep_pages`, `list_files`, `read_file`,
  `query_table`, `list_skills`, `read_skill`, `list_sources`, `list_source`, `read_source`,
  `search`, `fetch_history`. (`backend/services/tool_loop.py`, `backend/services/prompts.py →
  ASK_TOOL_SET`)
- System prompt: "Answer questions by calling tools to ground every claim. Reference what you
  found by name (e.g., the page name, session id, skill title, or table)." That's the full citation
  contract — a name reference, not a sentence-level provenance span. (`backend/services/prompts.py →
  render_ask_system`)
- **No per-sentence citations.** The `tool_result` SSE event (`{"type": "tool_result", "id":...,
  "name":..., "ok":...}`) lets the UI render a "citations strip" attached to a tool call, but
  this is tool-call level (i.e. "this tool ran"), not sentence-level. (`backend/services/tool_loop.py`,
  `backend/routers/agent_chat.py`)
- **No permission-aware synthesis / redaction-on-read.** Permissions are enforced at query time
  (which rows the user can see) but the synthesis layer has no per-sentence redaction gate.

### 3e. Memory Wiki (the "context graph" / knowledge tree)
- A scheduled curator agent (`render_curator_prompt`) runs nightly on a Sprite VM, reads the
  incremental change feed (`curation_service.changes_since`), and writes/updates pages under a
  special `is_memory` folder using `stash files add-page / edit-page` CLI commands. It is an
  LLM-driven agent prompted to maintain a Karpathy-style wiki: entity pages, concept pages,
  categories as subfolders, a root index, an append-only Log. (`backend/services/prompts.py →
  render_curator_prompt`, `backend/tasks/agent_schedules.py`)
- The "context graph" visual in the dashboard is a link graph derived by regex-scanning page bodies
  for `[text](/p/<uuid>)` markdown links. Not an actual graph DB or entity-resolution layer.
  (`backend/services/files_tree_service.py → memory_wiki_graph`)
- **Not a structured schema.** The wiki's structure is entirely at the discretion of the curator
  LLM. There is no OKF schema, no conformance validation, no diff-determinism guarantee.

---

## 4. Where Stash Appears AHEAD of Hyperspell

**Agent-host coverage**
Stash ships working plugins for 6 coding agents out of the box: Claude Code, Cursor, Codex,
OpenCode, Gemini CLI, and Openclaw. Each plugin pushes session transcripts automatically with no
manual steps from the user. (`plugins/`, `README.md`) Hyperspell's equivalent (the OpenClaw plugin)
is still in active development. Stash's install path (`stash signin` wires hooks automatically)
is genuinely frictionless.

**Writable Markdown Drive**
Stash's Files surface is a collaborative writable Markdown editor with real-time Yjs sync — agents
and humans write to the same pages and see changes live. (`backend/routers/collab.py`, `collab/`)
This is a real product surface, not just retrieval. Hyperspell's brain tree is read-mostly (agents
read it; writing back is manual or via the curator).

**VFS shell surface**
`stash vfs "ls / -L 2"`, `cat`, `find`, `grep`, pipes — any agent that knows bash can navigate a
Stash without learning a new SDK. This is usable by agents that have zero Stash-specific knowledge.
(`stashvfs/`, `backend/routers/vfs.py`) Hyperspell has `hyperbrain` CLI but not a POSIX VFS.

**Skills: the shareable/publishable bundle**
A Skill is a special folder with a `SKILL.md` — it has a public URL, a Discover page, and can be
forked into another user's account. The agent can create, publish, and fork Skills via the CLI.
(`ARCHITECTURE.md`, `backend/routers/skills.py`) This gives Stash a network-effect flywheel
(shared knowledge bundles) that Hyperspell doesn't have yet.

**Federated live-search for external sources**
For Gmail, Slack, Gong, Jira, Asana: Stash searches the provider's live API at query time rather
than relying solely on what's indexed. (`backend/services/source_service.py → _federated_search`)
For time-sensitive inboxes this returns fresher results than a stale local index.

**Self-hostable with Docker Compose in ~5 minutes**
`docker compose up` brings up postgres, redis, backend, frontend, worker, beat, and collab in a
single command with pre-built GHCR images. `stash signin` handles auth. (`README.md`,
`docker-compose.prod.yml`) Hyperspell's self-hosting story is not this simple.

**Open source (MIT)**
Stash is fully open-source. Engineers can read, fork, and audit every line. (`LICENSE`)
Hyperspell is not open-source; the context-retrieval plugin is proprietary. That matters for
enterprise buyers who want audit rights.

---

## 5. Where Hyperspell Appears AHEAD of Stash

**Hybrid retrieval (vector + full-text, fused)**
Hyperspell fuses vector similarity and BM25-style full-text scores before ranking. Stash has both
search paths but they are **never merged in a single query path** — the agent tool `search`
uses only FTS; the `/pages/semantic-search` endpoint uses only vector; no code calls both.
A query that's strong on semantics but weak on keywords (or vice versa) silently degrades in Stash.

**Reranking**
Hyperspell uses ZeroEntropy zerank-2 cross-encoder reranking as a post-retrieval step. Stash has
**no reranker** — results come back in raw `ts_rank` or cosine order. Cross-encoders typically
improve MRR by 15-30% over bi-encoder retrieval alone, and the delta on long documents with
partial keyword overlap is larger.

**Recency half-life + MMR diversity**
Hyperspell applies a recency decay (half-life scoring) and MMR to push stale or redundant results
down. Stash returns raw rank order from postgres. For a knowledge base that accretes over years,
this matters: Stash will surface an old session from 18 months ago at the same rank as last week's,
if the embedding similarity is equal. Stash's migration schema does have a `recency_intervals`
column (`backend/migrations/versions/0001_initial_schema.py:178`) but no code in the backend
applies it to retrieval scoring — it appears unused.

**Per-sentence citation provenance**
Hyperspell attaches per-sentence source IDs throughout the synthesis pipeline and enforces
redaction-on-read: if a sentence's source is not readable by the caller, it is suppressed before
output. Stash's citation model is tool-call level: the UI shows "search_history was called" but
cannot attribute individual claims in the response to specific source documents. A multi-source
synthesis in Stash is unauditable at the sentence level.

**Structured, deterministic knowledge tree (Temporal workflow)**
Hyperspell's tree is produced by a durable Temporal workflow that generates an OKF-conformant
markdown tree — structured schema, multiple passes, deterministic output. Stash's Memory wiki is an
LLM agent instructed to "maintain the wiki" — it can hallucinate cross-links, skip categories,
and produce structurally inconsistent output across runs. The Karpathy-inspired prompt in
`render_curator_prompt` is well-designed but fundamentally relies on LLM compliance, not code
guarantees.

**No embedding cap at whole-document level**
Stash clips everything to 6,000 chars (`MAX_TEXT_CHARS = 6_000`) before embedding the full
document as one unit. For a 20k-word design document, only the first ~900 words are semantically
represented. Hyperspell uses per-sentence provenance which implies finer-grained embedding units.
This is a real retrieval quality gap for long documents.

**Permission-aware synthesis / redaction-on-read**
Stash enforces permissions at the row-query level (who can see which sessions/pages). But there is
no per-sentence redaction step in the synthesis layer — once a document is in the tool result,
all of its content goes into the model context regardless of sub-document ACLs. Hyperspell's
redaction-on-read design is architecturally more defensible for enterprise contexts.

---

## 6. What's Worth Stealing

**The VFS shell as agent interface**
`stash vfs "grep 'database migration' /"` is a genuinely elegant idea: agents know bash, POSIX
paths are universal, and grep/find/cat compose into complex retrieval without a query language.
`stashvfs/` is a clean ~300-line Python package. Worth evaluating as an alternative or complement
to `hyperbrain search`. Cost: maintaining path-namespace coherence as the source tree grows.

**Skills as shareable/publishable bundles**
The Skills model (folder + `SKILL.md` → public URL, fork-into-account) creates a distribution
mechanism and network effect. Useful for sharing prompt sets, playbooks, and research threads
across teams. Hyperspell has skills as a local concept (OpenClaw skill files) but no publish/fork
network layer.

**Federated live-search alongside indexed content**
The `_federated_search` pattern — query the provider's live API at search time for sources where
freshness trumps coverage (Gmail, Slack) — is worth adopting when indexed content is stale.
Hyperspell currently only retrieves from the local index.

**Confidence tagging in curator prompt**
The curator prompt's instruction to mark facts `(extracted)` / `(inferred)` / `(ambiguous)` and to
never create pages from ambiguous-only material is a rigorous epistemic discipline. Worth encoding
as a hard rule in Hyperspell's synthesis prompts too. (`backend/services/prompts.py`)

**Incremental delta-only curation**
`curation_service.changes_since` gates the curator to only the delta since the last watermark,
not the full corpus. Hyperspell's Temporal workflow should already do something similar, but the
explicit `has_changes_since` cheapness gate (avoids waking a sprite for idle users) is a nice
optimization worth copying. (`backend/services/curation_service.py`)

**Fire-and-forget async embedding with `embed_stale` reconciler**
On event push, Stash schedules an async embed task and sets `embed_stale = TRUE` on failure.
A Beat-scheduled reconciler picks up stale rows every 60s. This is resilient to API transients
without blocking the write path. (`backend/services/memory_service.py → _schedule_event_embed`,
`backend/tasks/embeddings.py`)

---

## 7. What to Ignore

**The search architecture (FTS-only agent tool)**
The agent's `search` tool calls `source_service.search_all` which fans out to FTS only — no vector,
no reranking, no fusion. Copying this would be a step backward from Hyperspell's hybrid retrieval.
It's a deliberate simplicity choice on their part (CLAUDE.md: "code simplicity is our most
important concern") but wrong for a retrieval-quality-first product.

**Local `all-MiniLM-L6-v2` as default embedding**
384-dimensional, ~2019-era model. Good for free-tier self-hosting where API cost matters.
Hyperspell should continue using higher-quality embedding models; the "works with no keys"
story isn't relevant for the Hyperspell use case.

**The Sprites cloud VM approach**
Running a persistent Linux VM per user (sprites.dev) so the curator agent has a real shell is an
interesting architecture for a product that wants agents to run arbitrary code. But it adds
infrastructure complexity (VM provisioning, sleep/wake latency, SSH WebSocket relay) that
Hyperspell's durable Temporal workflow doesn't need. Hyperspell's synthesis runs as structured
workflow steps, not as a free-roaming shell agent.

**No backwards compat / no fallbacks (CLAUDE.md rule)**
Stash's CLAUDE.md explicitly bans all fallback code paths: "no fallbacks of ANY kind, ANYWHERE,
under ANY circumstance." This is a velocity-optimizing dev culture rule for a startup. At
Hyperspell's data sensitivity level, some defensive fallbacks (e.g. partial redaction on synthesis
error) are worth the complexity.

**Whole-document embedding (6k char cap)**
The `MAX_TEXT_CHARS = 6_000` clip embeds only the first ~900 words of a long document. Don't copy
this approach for synthesis quality. Hyperspell should use per-sentence or windowed chunking for
long-form content.

**Auth0 for self-hosting**
Optional for self-hosted deploys but required for the managed product. Adds complexity and
vendor lock-in. Hyperspell's auth approach is independent.

---

*All claims grounded in: `backend/services/memory_service.py`, `backend/services/source_service.py`,
`backend/services/files_tree_service.py`, `backend/services/tool_loop.py`,
`backend/services/prompts.py`, `backend/services/embeddings/`, `backend/tasks/embeddings.py`,
`backend/tasks/extraction.py`, `backend/services/curation_service.py`,
`backend/services/sprite_service.py`, `stashvfs/`, `backend/routers/vfs.py`,
`backend/celery_app.py`, `docker-compose.yml`, `.env.example`, `ARCHITECTURE.md`, `README.md`,
`CLAUDE.md`, `plugins/openclaw-plugin/`, `backend/migrations/versions/0001_initial_schema.py`.*
