# Method Template Index (production-proven skeletons)

> Each template: responsibility, key design decisions, core code skeleton.
> They are skeletons to re-implement in your own pipeline — the design decisions
> and invariants are the load-bearing part; this file explains WHY each piece
> is written this way.

---

## 1. Model probe script (probe_embedding.py)

**Responsibility**: before adopting any embedding model, verify its behavioral
contract with a standard probe set.

Probes:
1. A/A′ noise baseline: same request twice, cosine must =1.0 (proves determinism)
2. B: `text_type=query + instruct` → vector must differ from A significantly (instruct effective)
3. C/D: no text_type / instruct on the document side → must equal A (inert per docs)
4. E negative probe: invalid instruct type → 400 naming the field (schema proof)
5. F: batch N + non-default dimension
6. **Token-count comparison**: B's total_tokens ≈ A + instruct tokens — hard
   evidence the server prepends the instruction

Verdict template: `A vs B < 0.99 AND token increase → EFFECTIVE; otherwise ignored`.

## 2. Per-type profiles & the freeze script (freeze_profiles.py)

Profile JSON schema (shared by all types):

```json
{
  "doc_type": "api_reference",
  "source_hints": {"include_globs": [], "exclude_globs": []},
  "chunk_rules": {
    "parent_boundary": "<descriptive text>",
    "min_parent_chars": 300, "min_child_chars": 80,
    "max_child_chars": 2400, "code_split_max_chars": 6000,
    "split_code_by_comments": true, "keep_tables_whole": true,
    "merge_short_into_prev": true,
    "signature_blocks": {"signature_language": "C"}
  },
  "field_weights": {"title": 2.5, "path": 0.5, "body": 1.0,
                     "code": 0.9, "symbols": 3.0},
  "index_params": {"k1": 1.4,
                    "per_field_B": {"title": 0.2, "body": 0.7,
                                     "code": 0.6, "symbols": 0.25}},
  "retrieval_hints": {"downweight_trigger": {...}},
  "decisions_applied": ["D4","D9"],
  "open_questions_deferred": [...]
}
```

Freeze-script three principles: read the draft, bake human decisions in as
`D-number` entries, keep the draft for audit. **Empirical weight starting
points** (measured on a C-API-heavy corpus): api-type symbols dominant (3.0), title B low (0.2);
guide-type body raised (1.2); changelog uniformly low weight + downweight.

## 3. Extraction extension skeleton (kb_extract_ext.py)

Mount point and main loop:

```python
def setup(app):
    app.connect('doctree-resolved', on_doctree_resolved)
    return {'version': '2.0', 'parallel_read_safe': True,
            'parallel_write_safe': True}
```

Core recursion `_build_section(section, ..., level, root_title)`:
- title via `title_node.astext()` (section.names is lowercased)
- level==0 is H1: return a container; blocks go to preamble
- recurse subsections; `sphinx desc` (autodoc entries) go through
  `_build_desc` as standalone chunks marked `kind='desc'`
- content nodes convert to typed block dicts via `_block_of()`; unknown nodes
  fall back to text rendering

Noise toolkit: `_SKIP_TYPES` wholesale skip, container unwrapping,
SVG/zero-width regex cleaning, `:ref:` role → display text +
`.. _anchor:` line deletion (common residue in generated docs).

Output contract: one JSON per doc `{docname, file_path, title, top_titles[],
preamble[], chunks[]}`; chunk carries `kind`(section/desc)/`level`/
`heading_chain`/`canonical_path`/`root_title`/`blocks[]`/`children[]`/
`full_markdown`.

## 4. IR builder (build_ir.py)

Three key function patterns:

```python
def select_parents(doc, dtype):        # tree → flat parent list
    stack = []
    for u in pre_order(doc['chunks']):
        if _is_parent(u, dtype):
            rec = {..., '_ancestor': None}
            parents.append(rec)
            while stack and stack[-1]['level'] >= u['level']:
                stack.pop()
            rec['_ancestor'] = stack[-1] if stack else None   # ← bind AFTER pop!
            stack.append(rec)
        else:
            if stack: stack[-1]['absorbed'].extend(u.get('blocks', []))
```

```python
def fold_tiny_api_parents(parents):     # small entries fold into their ancestor
    for p in parents:
        if p['level'] >= 2 and p['_ancestor'] and _total_chars(p) < 300:
            p['_ancestor']['absorbed'] += p['blocks'] + p['absorbed']
```

Child-splitting state machine: `buf` prose buffer (flush at ≥max_child) /
code stands alone as a child (over-limit split at comment anchors) / tables
stand alone, never split / deflist items enter buf one by one / short tails
merge back into the previous prose child.

Acceptance trio (run on every rebuild):
- Conservation: ΣIR(body+code) / Σextractor leaf-blocks ≈ 1.0 (±2%)
- Structure: orphans=0, seq contiguous, paths unique, parent_id=md5(path)[:16]
- MD line-map spot check: parent start lines land on heading lines

## 5. MD materialization + line mapping (materialize_md)

Deterministic serialization: fixed first line `# {title}`, parents joined in
order with blank lines between. meta.json records
`{docname: {file, title, parents:[{parent_id,path,start,end}]}}`.
Acceptance: start line minus `#` equals the heading exactly; ranges never
overlap; a rerun produces sha256-identical files.

## 6. IdentifierAnalyzer layering pattern

```python
class IdentifierAnalyzer(Analyzer):        # must live in an importable module!
    def __call__(self, value, **kw):
        for tok in RegexTokenizer(_IDENT_RE)(value, **kw):
            yield Token(text=tok.text)               # ordinary words kept unconditionally
            if '_' in t or '.' in t or not tok.text.islower():
                for t in tokenize_text_symbols(tok.text):   # expand identifiers only
                    if t != tok.text: yield Token(text=t)
```

`_IDENT_RE = r"[A-Za-z_][A-Za-z0-9_.]*|\d+(?:\.\d+)+"` (version numbers become
whole tokens). Query routing, three branches: identifiers → symbols
core-must-match (Term 3.0 / Prefix 2.0 / title 4.0 AND subword bonus 0.6);
trailing `*` → pure-Prefix wide search; everything else →
MultifieldParser(OrGroup), version numbers as whole-token Prefix.

## 7. BM25 backend interface (dual implementations reserved)

```python
class BM25Backend(Protocol):
    def build(self, children, doc_type): ...
    def search(self, doc_type, query, limit) -> list[dict]: ...
```
Whoosh implementation + (future) Tantivy share the signature; switching swaps
the class only. Per-type = one index directory per doc type + its own
make_bm25f(profile params).

## 8. Embed/Rerank client patterns (embed_client.py / rerank_client.py)

Shared skeleton:
- requests against the native endpoint (one less SDK dependency)
- exponential-backoff retries (429/5xx only; 4xx raises)
- usage stats read total_tokens
- **local tokenizer for exact token counts** (required by rerank budgets)

Embed-specific: batch 20 with concurrency, checkpoint/resume (child_id
idempotency), collection metadata model/dimensions mismatch auto-rebuild.
Rerank-specific: budget batching `qt*N+Σdocs≤115K`, char-truncation fallback
for over-limit docs, multi-batch (orig_index, score) merge.

## 9. Fusion retrieval core (retrieve.py)

```python
# child-level fusion
rec['subs'][channel] = round(w_ch * norm_value, 4)
# parent aggregation norm_sum
fused_child = sum(rec['subs'].values())
g['matched'].append(fused_child)
score = sum(g['matched']) / math.log(1 + len(g['matched']))
# Rerank pass (optional): oversized parents split into children, take max
```

Return shape: results[{parent_id,path,title,score,dense,bm25f,rerank_score,
matched_children,n_children,children[outline],body(cap 6000 + hint)}] +
degraded/weight/channels/reranked.

Degradation matrix: dense failure → bm25-only + degraded; rerank failure →
fused order + degraded; dense retries once (sleep 1s) before giving up.

## 10. MCP Server pattern (mcp_server.py)

FastMCP five tools + description-writing rules in references/mcp_design.md.
Code points:
- Lazy singleton: the global var name ≠ the accessor function name (anti-shadowing)
- Every tool wraps try/except returning error text (MCP never raises to callers)
- kb_read's `_collect(p_abs, start, limit_lines, max_bytes)` generic line
  collector is reused by kb_read_path (limit 10**9 = byte-budget-only)

## 11. Golden regression script (p7_regression.py)

- GOLDEN: `(label, search_kwargs, [acceptable path substrings])` — multi-target
  cases list several substrings; substrings must follow the architecture
  (folding changes paths to their group)
- CONFIGS: bm25/dense/fuse/full four-way ablation
- Output: Hit@1/Hit@3/MRR table + per-stage latency (dense/bm25/rerank/e2e) +
  JSON persisted
- Triage mantra: abnormal rank → check ① golden substring invalidated
  ② basis double counting ③ transient network swallowed into degraded


## 12. Multi-KB registry (config.KBS)

```python
KBS = {
  "corpus_a": {"display": "Corpus A",
               "sphinx_kb": REPO/"sphinx-src/corpus_a/doc/_build/kb",
               "data": DATA_DIR/"corpus_a", "profiles": PROFILES/"corpus_a",
               "collection": "corpus_a_kb",
               "examples": {"<EX_KW>": "geom_kind", ...}},
  "corpus_b": {...same shape...},
}
def get_kb(name): ...   # validates + injects name
```

Points:
- TWO path bases: data/profiles are relative to the pipeline root; sphinx_kb is
  relative to the repo root (the pipeline root's parent). Mixing them bit us twice.
- classify registry mirrors the shape: `_CLS = {"corpus_a": fn, "corpus_b": fn}`
  with one entry point `classify(kb, docname)`.
- Full-pipeline threading: build_ir/bm25/vectorize CLIs take `--kb`;
  Retriever(kb=...); MCP uses the `KB_NAME` env var. **Every replacement must
  assert + run a distribution sanity check** (the all-guide misclassification
  incident).

## 13. Tool-description localization (per-KB examples registry)

Description bodies carry placeholders (first line `<CORPUS>`, examples
`<EX_KW>/<EX_PATH>/<EX_SYM_GREP>/<EX_INSTRUCT_SYM>`...), injected uniformly at
registration:

```python
def _register(fn):
    doc = fn.__doc__ or ""
    doc = doc.replace("<CORPUS>", DISPLAY)
    for token, value in _KB.get("examples", {}).items():
        doc = doc.replace(token, value)
    fn.__doc__ = doc
    return mcp.tool()(fn)          # decorators removed; explicit registration at the end
```

Acceptance = **two-way leak scan**: the corpus-A instance's forbidden set =
corpus-B symbols (and vice versa); zero leaks to pass. Examples must stay concrete
(LLM steering power), but concrete values follow the deployed corpus.

## 14. Two-round real-usage test (the user's-eye acceptance gate)

```
subagent A (corpus 1) ← round-1 prompt: business questions only, ZERO tool teaching
subagent B (corpus 2) ← same (dispatched in parallel)
   ↓ steering message into the same session
round 2 ← advanced questions (natural extension of round-1 conclusions)
   ↓ steering message
interview ← tool counts / three-axis scores / worst moment / degraded count / highlight+improvement
```

Design red lines:
- Round-1 prompts must **never name a tool or a call order** — spoilers make
  the description's real steering power unmeasurable
- Round 2 must extend round 1's conclusions naturally (a physics attribute →
  the state-flags enum; first training → custom obs/rewards)
- The interview states "no new research needed" so it doesn't run more searches
- Every "defect claim" in feedback gets reproduced against files first
  (measured: of 3 claims only 1 was real)
