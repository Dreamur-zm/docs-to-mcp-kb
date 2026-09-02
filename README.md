# docs-to-mcp-kb

A skill + reference material for turning open-source project documentation
(Sphinx rst/md and similar) into **MCP-accessible retrieval knowledge bases**:
cloud Embedding + BM25F dual-channel fusion over two-tier parent/child chunks,
wrapped as five MCP tools with markdown direct-read fallbacks.

> Distilled from a real production refactor that shipped live knowledge bases
> for several large open-source projects. Every pitfall in this repository was
> actually hit, and every default is backed by ablation data.

## What's in this folder

```
docs-to-mcp-kb/
├── SKILL.md                  # the skill: 9-phase workflow + pitfall quick tables
├── references/
│   ├── pitfalls.md           # 42 pitfalls (symptom → root cause → fix → prevention)
│   ├── decisions.md          # 12 ADRs (engine/embedding/fusion choices, with data)
│   ├── templates.md          # 14 method templates (probe/freeze/build_ir/analyzer/...)
│   └── mcp_design.md         # MCP tool-layer spec (description rules, limits, degradation)
├── README.md                 # this file (English)
└── README_zh.md              # Chinese version
```

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
| `search_knowledge_base` | Main entry. Child-level fusion (keywords→BM25F, query→dense) aggregated to parent sections; hits carry full section text + child outline; optional rerank precision pass |
| `kb_read_path` | Read a section directly by the canonical path from a search hit (50KB budget, `max_lines` for giant sections) |
| `kb_read` | Line-range reads over the materialized markdown (line-numbered `N: content`, 2000-char line cap, 50KB budget, continuation footers) |
| `kb_grep` | Python-regex content search across the MD store (grouped output, 100-match cap, zero-hit self-diagnosis) |
| `kb_structure` | Corpus map / per-document section outline |

## Quick start

1. **Adopt the skill**: copy `docs-to-mcp-kb/` into your skill directory and
   follow `SKILL.md`'s 9-phase workflow for your documentation corpus.
2. **Lean on the references while you build**: `references/templates.md` for
   copy-paste method templates, `references/pitfalls.md` when something looks
   wrong, `references/decisions.md` when you must choose an engine, embedding
   or fusion scheme, `references/mcp_design.md` when wrapping the tool layer.
3. **Wrap as MCP**: expose the five-tool chain per `references/mcp_design.md`,
   then register one server entry per corpus in your MCP host's config (a
   `KB_NAME`-style env var keeps multi-corpus deployments clean).

Technology assumptions behind the defaults (validated — see
`references/decisions.md`): Python ≥3.10, `sphinx` (extraction),
`whoosh-reloaded` (BM25F), `chromadb` (dense store), `mcp>=1.2,<2` (server),
a cloud embedding endpoint (`EMBEDDING_API_KEY`) and an optional reranker.

## Why not plain RAG?

Because documentation corpora are hostile to naive RAG: API pages, concept
guides and changelogs need different chunking; symbols need subword matching
(`parseHTTPResponse` must be findable from "parse http response"); giant
structs defeat fixed-size splitters; and pure semantic search misses exact
identifiers. This skill's answer: per-doc-type profiles, BM25F with field
weights and identifier tokenization, parent-section returns for context, and
a 42-item pitfall list so you don't re-learn them the hard way. Start at
[SKILL.md](SKILL.md).

## License
This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
