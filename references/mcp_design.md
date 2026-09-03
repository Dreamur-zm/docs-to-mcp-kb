# MCP Tool-Layer Design Spec

> The five-tool responsibility chain, description-prompt writing rules, and limits & truncation management.
> Implementation pattern: FastMCP + stdio; re-implement it in your own pipeline package.

---

## 1. Tool responsibility chain (mutual references form a closed loop)

```
search_knowledge_base        Main entry: hits carry parent-chunk full text + child-chunk directory
  ├─ kb_read_path(path)      Precise read of one section by canonical path (50KB budget)
  ├─ kb_grep                 Cross-file regex for exact literals (last line of defense)
  ├─ kb_read                 Continued reading of any line range in any file
  └─ kb_structure            Targeted: document inventory / single-document outline
```

The description must state the priority semantics clearly: search comes first;
grep is the exact-literal supplement after search; read is the continuation means
for content beyond the hit section's range.

## 2. Description-prompt writing rules (aligned with mainstream MCP-server description style)

**Write usage only, never the underlying implementation.** The caller does not need
to know the fusion formula, the normalization scheme, or model names — that is "how
it is implemented"; it needs to know "when to use it, how to fill the parameters,
what is returned, and what happens on failure".

Required blocks:
1. **One-sentence positioning** (what it returns)
2. **Parameters block**: per-parameter explanation + concrete example values
3. **Preset table** (e.g. the verbatim text of the four English instruct presets)
4. **Usage notes**: composition of hit content, degraded behavior, context-budget
   advice, the nested-subagent guard sentence, workflow references
5. **Limits statement**: truncation thresholds, caps, the footer continuation mechanism

### Nested-subagent guard sentence (copy verbatim)

> If YOU are that delegated agent, do not spawn further agents -- run the
> searches yourself and return the summary.

Place it at the end of the "context budget" advice. Reason: a long-lived subagent
reading the "delegate to a subagent" advice does not know that it is itself a
subagent and will nest without end.

### Bad example → good example

Bad: "Fuse BM25F with weighted-normalized dense vectors, then rerank via <model name>"
Good: "weight: 0..1 balance between keyword ranking (0) and semantic ranking (1)"

## 3. Parameter design conventions

- **Queries split three ways**: keywords (keywords → BM25F) / query (natural
  language → dense) / instruct (English task instruction → the dense-side API
  parameter). Leaving any of the three empty switches off the corresponding channel.
- **A single weight knob** `weight ∈ [0,1]`: 0 = pure keyword, 1 = pure semantic,
  linear interpolation. Better than two weights: intuitively semantic, with clear
  degradation at the extreme values.
- **Boolean capability switches**: defaults are decided by ablation experiments
  (the data behind rerank defaulting to off: symbol/version-type queries Hit@1 12→9).
- Caps are clamped inside the tool (top_k≤30), and stated explicitly in the description.

## 4. Return format conventions

- Plain-text content; structured information uses stable prefix lines
  (`path=`, `child blocks(N):`, `Section text:`) so callers can extract it with regex.
- Every hit carries its own full text (capped at 6000 characters + a continuation
  hint) plus per-channel sub-scores and the fused score — this explainability lets
  the upper-level model judge by itself whether to re-query with different weights.
- The status line is always the first line: `weight=... channels={...} reranked=...`;
  degraded information `[degraded] ...` immediately follows.

## 5. Limits & truncation inventory (tool layer)

| Tool | Limit | Value | How surfaced |
|---|---|---|---|
| search | top_k ≤30; rerank candidates 20 | clamped | noted in description |
| kb_read | default window 2000 lines; 50KB byte budget; line length truncated at 2000 | footer three states: paging / capped / EOF + next offset | |
| kb_read_path | 50KB budget per section | `(End of section)` / truncation notice | |
| kb_grep | ≤100 hits; whole lines returned (truncated at >2000); stops scanning at the cap | `(Results truncated...)` + narrowing suggestion | |
| Global | MD-store path escape rejected | error message | |

Design principle: **truncation must remain continuable** (provide the exact next
offset / an alternative tool); never drop content silently.

## 6. Degradation matrix

| Failure | Behavior | Marker |
|---|---|---|
| embedding API failure (after 1 retry) | fallback to BM25F-only | `[degraded] dense unavailable: ...` |
| rerank API failure | keep the fused ranking | `[degraded] rerank unavailable: ...` |
| transient SSL jitter | retry once before entering the degraded path | — |
| no API key (deployment environment) | same failure path | usable at startup (keyword retrieval does not depend on a key) |

## 7. Test pattern

stdio end-to-end acceptance script: spawn the server as a subprocess → initialize →
list_tools asserts the five tools → real calls per tool assert the output shape
(structure markers / footers / hit counts). The API key is passed to the subprocess
through env. One run ≈10s; after any change to the MCP layer it must be all green.

## 8. Host integration

Standard mcpServers configuration (works in any MCP host: Claude Desktop, Claude Code, IDE integrations, ...):

```json
{
  "mcpServers": {
    "my-docs-kb": {
      "command": "python3",
      "args": ["-m", "mypkg.mcp_server"],
      "env": {"PYTHONPATH": "<abs>/your-pipeline/src"}
    }
  }
}
```

Note: the server uses absolute paths internally (the data directory is derived from
the code location), no cwd dependency; the first call incurs ~1-2s lazy loading
(chroma+whoosh), which is normal.


## 9. Multi-KB: description localization & registry injection

The same mcp_server.py instantiates servers for multiple corpora via the `KB_NAME`
environment variable. Corpus-specific content in the description prompt (corpus
name, example symbols, example paths, symbol-type instruct preset) is always
written as placeholders (`<CORPUS>`/`<EX_KW>`/`<EX_PATH>`/`<EX_SYM_GREP>`/`<EX_INSTRUCT_SYM>`...);
real values are injected from config.KBS[kb]["examples"] at registration time.

- Examples must be concrete (strong steering for the LLM); concrete values follow
  the corpus — guaranteed by the registry
- Acceptance = bidirectional leak scan: each instance's forbidden-word set = the
  other corpus's symbol set; passing requires a zero count
- Onboarding a new corpus only needs one registry entry (path+collection+examples);
  zero description changes

Hard-won lesson: a half-implemented placeholder mechanism (only the first line
replaced, body examples hardcoded) is stealthier than none at all — the corpus-B
instance's description once taught the model to grep a nonexistent enum. The
bidirectional scan must be run before going live.
