# docs-to-mcp-kb

A skill + reference implementation for turning open-source project
documentation (Sphinx rst/md) into **MCP-accessible retrieval knowledge
bases**: cloud Embedding + BM25F dual-channel fusion over two-tier
parent/child chunks, wrapped as five MCP tools with markdown direct-read
fallbacks.

> Distilled from a production refactor that built three live knowledge bases —
> **MuJoCo** (936 parents / 2,808 children), **mjlab** (430 / 890) and
> **RSL-RL** (106 / 139) — every pitfall in this repository was actually hit,
> and every default is backed by ablation data.

## How it works

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

## The five MCP tools

| Tool | Role |
|---|---|
| `search_knowledge_base` | Main entry. Child-level fusion (keywords→BM25F, query→dense) aggregated to parent sections; hits carry full section text + child outline; optional qwen3-rerank pass |
| `kb_read_path` | Read a section directly by the canonical path from a search hit (50KB budget, `max_lines` for giant sections) |
| `kb_read` | Line-range reads over the materialized markdown (opencode-style `N: content`, 2000-char line cap, 50KB budget, continuation footers) |
| `kb_grep` | Python-regex content search across the MD store (grouped output, 100-match cap, zero-hit self-diagnosis) |
| `kb_structure` | Corpus map / per-document section outline |

## Repository layout

```
docs-to-mcp-kb/
├── SKILL.md                  # the skill: 9-phase workflow + pitfall quick tables
├── references/
│   ├── pitfalls.md           # 42 pitfalls (symptom → root cause → fix → prevention)
│   ├── decisions.md          # 12 ADRs (engine/embedding/fusion choices with data)
│   ├── templates.md          # 14 method templates (probe/freeze/build_ir/analyzer/...)
│   └── mcp_design.md         # MCP tool-layer spec (description rules, limits, degradation)
├── README.md / README_zh.md
└── kbcore/                   # reference implementation (see source repo kb_v2/src/kbcore/)
    ├── identifier_tokenizer.py   # case-preserving identifier + version-number tokenizer
    ├── build_ir.py               # profile-driven parent selection & child splitting
    ├── bm25_backend_whoosh.py    # per-type BM25F indexes
    ├── embed_client.py           # qwen3.7 embedding client (batch/retry/resume)
    ├── rerank_client.py          # qwen3-rerank client (token-budget batching)
    ├── retrieve.py               # fusion core (norm_sum aggregation, degradation)
    └── mcp_server.py             # FastMCP five-tool server (KB_NAME multi-KB)
```

## Quick start

1. **Adopt the skill**: copy `docs-to-mcp-kb/` into your skill directory and
   follow `SKILL.md`'s 9-phase workflow for your documentation corpus.
2. **Reference implementation**: point `PYTHONPATH` at `kbcore/` (or vendor it),
   then drive the pipeline:
   - `build_ir.py --kb <name>` after a Sphinx build emits extractor JSON
   - `bm25_backend_whoosh.py build --kb <name>`
   - `vectorize.py --kb <name>` (cloud embedding, checkpoint/resume)
3. **Wire your host**: add an `mcpServers` entry per corpus (`KB_NAME` selects
   paths/collection/profiles/examples).

Requirements: Python ≥3.10, `sphinx` (extraction), `whoosh-reloaded`,
`chromadb`, `mcp>=1.2,<2`, `requests`; a DashScope-style embedding endpoint
(`EMBEDDING_API_KEY`, model `qwen3.7-text-embedding`) and `qwen3-rerank` for
the optional precision pass.

## Why not plain RAG?

Because documentation corpora are hostile to naive RAG: API pages, concept
guides and changelogs need different chunking; symbols need subword matching
(`mjtGeom` must match from `mjTEXROLE_USER`); giant structs defeat fixed-size
splitters; and pure semantic search misses exact identifiers. This repo's
answer: per-doc-type profiles, BM25F with field weights and identifier
tokenization, parent-section returns for context, and a 42-item pitfall list
so you don't re-learn them the hard way. Start at [SKILL.md](SKILL.md).

## License note

The pipeline code and skill text are released under the repository license.
**Third-party documentation corpora (MuJoCo, mjlab, RSL-RL docs) are NOT
included** — build your own from the upstream sources with the provided
extraction pipeline.
