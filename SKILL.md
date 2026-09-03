---
name: docs-to-mcp-kb
description: Build MCP-accessible retrieval knowledge bases from open-source project documentation (Sphinx rst/md and similar): cloud Embedding + BM25F dual-channel fusion, two-tier parent/child chunks, per-doc-type profiles, identifier tokenization, Reranker precision pass, markdown materialization, golden regression. Use this skill for ANY part of building a documentation knowledge base / RAG retrieval system — Sphinx doc extraction, Embedding/BM25/Reranker selection and tuning, chunking strategy, MCP tool wrapping, retrieval quality regression, even a single stage of it. Even when the user never says "knowledge base", if the intent is "make an agent able to retrieve a project's documentation", this skill applies.
---

# Documentation → MCP Retrieval Knowledge Base (v2 practitioner guide)

> This skill distills a complete real refactor (several large open-source
> documentation corpora: Sphinx docs → IR → BM25F/dense dual channel → fused
> ranking → five MCP tools). Every pitfall here was actually hit; every decision
> is backed by experiment data. Deep reading: `references/pitfalls.md`
> (42 pitfalls), `references/decisions.md` (12 ADRs), `references/templates.md`
> (14 method templates), `references/mcp_design.md` (MCP tool-layer spec).
> No documentation corpus or reference implementation ships with this skill —
> build your own following the workflow.

## Golden Rule: ask before you build

Confirm four things with the user before touching code:
1. **Doc source** — Sphinx? which version? path? any generation steps (codegen)?
2. **Embedding** — cloud API (vendor/dimensions/budget) or local model?
3. **Keyword engine** — Whoosh (pure Python, fully customizable) or Tantivy (fast, restricted tokenizer)?
4. **Delivery form** — which host consumes the MCP tool set? which tools are needed?

**Adapt, don't follow blindly**: this skill is an experience map, not a blueprint.
When a rule conflicts with your corpus, trust measured data and tell the user.

---

## Overall architecture

```
docs (rst/md)
  │ [build time] subagent deep-reads representative docs → per-doc-type profiles (human-frozen)
  ▼
extraction extension (Sphinx doctree-resolved AST) → structured block-tree JSON
  ▼
build_ir: per-profile parent selection → child splitting → five fields (title/path/body/code/symbols)
  ├─→ MD materialization (deterministic, stable line numbers) → kb_read / kb_grep / kb_read_path
  ├─→ Whoosh BM25F per-type indexes (independent params per type)
  └─→ cloud Embedding (text_type=document) → ChromaDB
  ▼
retrieval core: child-level fusion (normalized weighted) → norm_sum parent aggregation → optional Reranker
  ▼
MCP Server (FastMCP): search / kb_read / kb_read_path / kb_grep / kb_structure
```

Multi-KB: a `config.KBS` registry routes every path/collection/profiles/examples
by kb name; one codebase instantiates multiple MCP server entries via `KB_NAME`.

**Why two tiers**: child chunks give retrieval precision (BM25F length
normalization friendly); parents give full context. Key v2 evolution: hits return
**the parent section's full text + child outline** (zero second hop), not v1's
silent aggregation.

---

## Phase workflow

### Phase 0 — decision verification (half day)

- Write a `probe_embedding.py` probe for candidate models: connectivity, whether
  `text_type`/`instruct` actually take effect (cosine delta with/without
  instruction + token-count change), dimension cap, batch cap, latency.
  **Don't trust docs — run it.**
- Engine spike: scale real corpus to target size (100K blocks), measure index
  build time and query p95. Example criteria: build <60min and p95<500ms →
  Whoosh; else Tantivy.
- Verify the API key delivery chain (interactive shell ≠ non-interactive shell;
  MCP subprocess inherits from the HOST process).

### Phase 1 — subagent deep-read produces profiles

- One subagent per doc type deep-reads representative samples and produces a
  profile JSON: `{doc_type, source_hints{include/exclude_globs}, chunk_rules{
  parent boundary/min-max/code split}, field_weights, index_params{k1,
  per_field_B}, retrieval_hints}`.
- **Classification table is its own module** (classify.py): rules first,
  ambiguity to LLM.
- **Human review & freeze**: draft → user confirms decisions one by one
  (numbered D1..Dn) → freeze script bakes decisions into final JSON, keeps
  drafts for audit. Freeze script must be idempotent.
- Common three-type split: api_reference / guide / changelog_misc. Borderline
  short pages lean into guide.

### Phase 2 — extraction extension & build (user runs the build)

- Build-time AST capture: custom extension on `doctree-resolved` (AST complete,
  before HTML). Node-handling rules: references/pitfalls.md group 2.
- conf.py patch script: one-time backup (.bak) + register extension; idempotent,
  revertible (--restore).
- ⚠️ **Sphinx incremental build trap**: after editing the extension,
  `sphinx-build` may re-read only 2 "changed" files — new logic never applies
  to cached docs. Must `rm -rf _build` for a full rebuild. Discriminator:
  `[new config] N added` vs `N changed` in the log.
- Hand the build script to the user (minutes); on failure check the four
  expected warning classes first (autodoc import failures etc.).

### Phase 3 — IR build (densest pitfall zone)

IR = one JSON per doc: parents[] (with children[]). Key points:

- **Parent selection per type** (from profile): api=entries as parents
  (level≤3), guide=h2/h3, changelog=version sections (H2).
- **Tiny-entry folding**: entries <300 chars fold into their ancestor group —
  ⚠️ classic incident: aggregation bound the ancestor reference BEFORE the stack
  pop, content landed in dead dicts, conservation ratio fell to 0.63. Fix:
  **pop first, bind after**, and always run a "leaf-blocks character
  conservation" check after any change (extractor leaf text vs IR body+code,
  ratio ≈1.0).
- **preamble must be integrated**: document['preamble'] holds H1 lead blocks —
  missing it loses whole paragraphs.
- Child splitting pipeline: code blocks own a child (over-limit split at //
  comment anchors — threshold in CHARACTERS not lines!); tables never split;
  definition_list per item; prose buffer flushed at max_child_chars; short
  tails merge into the previous block.
- Five fields: body and code separated (prerequisite for BM25F per-field
  weights); symbols from one identifier tokenizer (exact + separator groups +
  subwords + lowercase variants, single-letter noise filtered).
- **Deterministic serialization**: same input must produce byte-identical
  output (the foundation of MD line-number stability). Lock with snapshot tests.

### Phase 4 — MD materialization + line mapping

- One .md per source doc; meta.json records parent_id ↔ file:start:end.
- Line stability from determinism: after rebuild, sha256 must match per file
  (part of acceptance).
- Acceptance spot check: every parent's start line lands on its heading line.

### Phase 5 — BM25F index layer (Whoosh)

- One index directory per doc type, each consuming its profile's weights/B.
- All five fields TEXT(analyzer=IdentifierAnalyzer); the ID field rejects an
  `indexed` kwarg (TypeError).
- ⚠️ **Custom Analyzer must live in an importable package module**: whoosh
  pickles it into the index schema; defined in `__main__` it fails to
  unpickle on open_dir.
- ⚠️ Analyzer must be **layered**: ordinary English words pass through
  untouched (full text needs them); only identifiers (with `_`/`.` or caps)
  get subword expansion. Misusing a "symbol-only filter" wipes all prose terms
  (the "soft contact" zero-hit incident).
- Index control file is `_MAIN_1.toc`, not `MAIN`.
- BM25F per-field usage: `BM25F(B=default, K1=k, weights={field:w},
  fieldname_B=x)`. **Always give symbols a low B (e.g. 0.25)** — enum definition
  pages are naturally long; the default 0.75 crushes them.

### Phase 6 — cloud vectorization

- Ingest side `text_type=document` without instruct; query side
  `text_type=query` + instruct. Correct asymmetric usage (verified by probe:
  instruct is captured by token counting).
- Batching + concurrency + exponential backoff on 429/5xx; resume by child_id
  idempotency; collection metadata records model/dimensions and auto-rebuilds
  on mismatch.
- usage field names differ per model (input_tokens vs prompt_tokens) — read
  total_tokens.
- ChromaDB default l2 is **squared euclidean**: unit-vector cosine =
  `1 − dist/2` (not 1−dist).
- When the per-text token cap dwarfs actual sizes, **drop the char truncation**
  (prove with the token-count script first).
- Full vectorization is API-bound: write the script, **hand it to the user**.

### Phase 7 — fusion retrieval core

- Child-level fusion: `child_fused = w·norm_dense + (1−w)·norm_bm25`, norm is
  per-channel min-max. Single `weight` knob exposed upward (0=keyword, 1=semantic).
- Parent aggregation (chosen by experiment): `parent_score = Σ matched_child_fused
  / ln(1+matched_n)`. Denominator uses MATCHED count — single strong hits in
  small sections aren't punished; many-match giants are damped.
  Basis: 6-scheme comparison experiment (sum/mean/max/norm_sum/top3/wmax).
  ⚠️ Fix the measurement basis first (leaf-blocks vs full_markdown which double
  counts subtrees).
- Reranker: hard limits are service-specific — probe yours (reference service:
  per-doc ≤4000 tokens (silent truncation), ≤500 docs
  and `query×N+Σdocs ≤120K` tokens per request). Oversized parents degrade to
  **per-child scoring with max** — no new granularity. Batching uses the local
  tokenizer for exact budgets.
- **Reranker defaults OFF**: golden regression showed net negative on
  symbol/version queries (Hit@1 12→9, the English-QA default instruction
  favors prose answer pages and under-scores C definition pages). Keep as an
  opt-in switch. **Every default comes from ablation data, not intuition.**
- Degradation: any embedding/reranker failure → remaining channel answers with
  a `[degraded]` note; never fail wholesale. Transient SSL errors retry once
  before degrading.

### Phase 8 — MCP Server (FastMCP)

Five-tool responsibility chain: search (main, hits carry full text) →
kb_read_path (canonical-path section read) → kb_grep (cross-file regex, last
resort) → kb_read (arbitrary range continuation) → kb_structure (orientation).

Description-prompt writing rules (see references/mcp_design.md):
1. **Usage only, no internals** (no fusion formulas/model names/normalization)
2. Parameter teaching with concrete examples; presets for weight/instruct
3. Tools cross-reference into a closed workflow
4. **Nested-subagent guard sentence**: "If YOU are that delegated agent, do not
   spawn further agents -- run the searches yourself and return the summary."
5. State limits and truncation behavior explicitly (footer continuation etc.)

### Phase 9 — golden regression & ablation (P7 methodology)

- Golden set covers: exact symbols / enum-constant ownership (search constant
  name, expect the enum page) / partial identifiers / giant structs / English
  concepts / Chinese cross-lingual / version intent / multi-target ambiguity.
  13 cases to start, expandable.
- Ablation configs: pure bm25 / pure dense / fusion without precision pass /
  full pipeline — Hit@1/Hit@3/MRR each.
- **Before concluding on anomalies, check three things**: ① golden substring
  invalidated by folding/refactor; ② measurement basis double counting;
  ③ transient network error swallowed into degraded (this round's three 999s
  were all these — none were retrieval defects).
- Latency benchmarks per stage: dense (API RTT ~300ms), whoosh (~36ms),
  rerank (~360ms), end-to-end p50.
- **Real-usage test (the last gate)**: dispatch two long-lived subagents as
  real users of each corpus — round 1 business questions only (zero tool
  teaching), round 2 advanced follow-ups steered into the same session by
  message, then an interview
  (tool counts/scores/worst moment/degraded count). Verify every defect claim
  against files before acting (this round: 3 claims → 1 confirmed, 2 false);
  land exposed UX issues as "fix + description sync" pairs.

---

## Pitfall quick table (details in references/pitfalls.md)

| # | Pitfall | One-line fix |
|---|---|---|
| 1 | Sphinx incremental build swallows new extraction | rm -rf _build; check `N added` |
| 2 | Fold aggregation loses content (dead dicts) | pop before binding ancestor; conservation ≈1.0 |
| 3 | preamble lost wholesale | H1 lead blocks become a doc-level parent |
| 4 | Line count used as char threshold | unify all size units to characters |
| 5 | Analyzer drops ordinary English words | layering: words kept, identifiers expanded |
| 6 | Custom Analyzer pickle failure | move into an importable package module |
| 7 | Prefix subwords too broad | core-must-match + title anchoring |
| 8 | chroma l2 = squared euclidean | cos = 1 − dist/2 |
| 9 | rerank-on default is net negative | defaults from ablation data |
| 10 | golden substrings invalidated by folding | goldens follow the architecture |
| 11 | audit basis double counts | leaf-blocks basis |
| 12 | subagent reports "missing" by bare title | verify the data source before acting |
| 13 | FastMCP name shadowing | lazy singleton var ≠ accessor fn name |
| 14 | whoosh ID(indexed=) doesn't exist | ID(stored=True, unique=True) |
| 15 | index control file name wrong | match `_MAIN_*.toc` |
| 16 | Path.with_suffix().as_path() missing | as_posix() |
| 17 | docname with subdir forgotten mkdir | parent.mkdir(parents=True) |
| 18 | usage field names differ per model | read total_tokens |
| 19 | transient SSL blips | retry once, then degrade |
| 20 | pinned old wheels that won't install | curated list + system-site-packages |

**Multi-KB & real-usage supplement (numbers map to pitfalls.md #31–#42)**

| # | Pitfall | One-line fix |
|---|---|---|
| 31 | inline script edits truncate source | tmp+rename atomic write; assert replaces |
| 32 | build params hardcoded in parallel | profile is the single source; build_ir reads _params_for |
| 33 | migration semantic drift voids vectors | behavior lock: byte-identical products to accept |
| 34 | golden substrings assumed across corpora | grep the IR to confirm substrings exist |
| 35 | PyPI same-name traps | check the official dependency list, not the grabbed name |
| 36 | two-round real-usage test spoiled | round 1 zero tool teaching; round 2 same-session message |
| 37 | subagent hallucinated defects | reproduce every claim against files |
| 38 | kb_read_path floods giant sections | max_lines param + exact continuation footer |
| 39 | silent zero-hit grep copy | context-aware suggestion chain, noted in description |
| 40 | cross-corpus description leaks | per-KB examples registry; two-way leak scan to zero |
| 41 | autodoc heavy-dependency decisions | diff against the mock list; watch PyPI same-name traps |
| 42 | set -u / silent no-op replaces | ${VAR:-} + assert + distribution sanity check |

---

## Development invariants (order = dependency)

```
chunking rule change → rebuild IR+MD → rebuild Whoosh → (embed text changed?) re-embed
extractor change     → rm -rf _build full Sphinx → rebuild IR → … whole chain
any refactor         → conservation check + line-map spot check + golden regression
```

Task split: API-bound (vectorization/Sphinx builds) go to the user;
local second-scale tasks (IR/indexes/regression) run here.

---

## New-corpus onboarding checklist

1. Source tree recon (scale/structure/toctree reachability/generation steps)
2. Subagent deep-read → profiles human-frozen
3. classify.py gains that corpus's decision table
4. In order: Sphinx build (user) → build_ir → BM25F index → vectorization (user)
5. Golden set expansion (incl. adversarial: big-parent weak match vs small-parent strong match)
6. Real-usage test: two business rounds + interview (see Phase 9); verify defect claims against files
7. MCP config gains one server entry (KB_NAME distinct) or reuses the existing one
