# Architecture Decision Records (ADR digest)

> Each entry: Background → Alternatives → Decision → Revisit when.
> All decisions are experiment- or measurement-backed; when a revisit condition
> fires, re-evaluate instead of holding the line.

## ADR-1 Embedding: cloud qwen3.7-text-embedding replaces local Ollama
- **Background**: local 0.6b held ~1G VRAM and needed residency management; user
  mandated cloud migration (API/SDK).
- **Alternatives**: local Ollama / DashScope v4 / qwen3.7-text-embedding.
- **Decision**: qwen3.7-text-embedding, 1024 dims, native HTTP protocol
  (requests, no SDK). Rationale: same semantic family (Qwen3), Code retrieval
  +20%, batch 20, 128K-token per-text, loose RPM, ¥0.5/M (full re-embed ¥0.18).
- Key measurements: instruct/text_type contract verified effective; vectors are
  unit-norm (chroma l2 equivalent to cosine).
- Revisit when: corpus grows to the 10M-token scale, or private deployment is required.

## ADR-2 BM25F engine: Whoosh first, Tantivy fallback
- Alternatives: Tantivy (fast; python bindings lack custom tokenizers) /
  Whoosh (slow, fully customizable) / SQLite FTS5 (zero deps but pseudo-BM25F).
- **Decision**: Whoosh continuation. Deciding factors: native per-field B and
  weights + fully customizable Analyzer + pure pip. Performance concerns are
  offset by per-type sharded indexes (one query hits only the 2–4 relevant
  small indexes).
- Revisit when: corpus >500K blocks and p95 >500ms → switch to Tantivy
  (pre-tokenization plan ready: Python-side split, space-joined at write).
  Both backends share one interface class.

## ADR-3 Per-type profiles replace a single rule set
- Background: v1's one-size-fits-all chunking made API entries and narrative
  pages compromise each other.
- **Decision**: build-time subagent deep-reads representative samples produce
  per-type profiles (parent boundaries/thresholds/field weights/index params);
  one shared builder dispatches per type.
- Proven value: api-type symbols weight 3.0 vs guide-type body dominance —
  inexpressible under a single rule set.

## ADR-4 Child-direct return + explicit deep read, replacing silent parent aggregation
- v1: hit child → aggregate → auto-return the parent's full text (context
  complete but size uncontrolled, ranking diluted).
- v2 first cut: return child body — user feedback demanded section-level context.
- **Final**: **hits return the parent section's full text + child outline**
  (6000-char cap + kb_read_path continuation).
- Lesson: requirements oscillate; the "return granularity" must be a parameter,
  not hard-coded.

## ADR-5 Aggregation formula norm_sum (six-scheme experiment)
- `parent_score = Σ matched_child_fused / ln(1 + matched_n)`
- Experiment: 13 goldens × 6 schemes (sum/mean/max/normsum/topk3/wmax), all
  medRank=1; norm_sum best balances Hit@1 vs big-parent inflation.
- Denominator basis: **matched count** (not total children) — a single strong
  hit in a small section isn't punished; many-match giants are damped.
- Revisit when: corpus expansion adds adversarial goldens (big-parent weak
  match vs small-parent strong match); if schemes diverge, consider a second
  parent-level Whoosh index (full_markdown as documents).

## ADR-6 Reranker defaults OFF
- Experiment: 13-golden ablation, fuse Hit@1 12/13 vs full(+rerank) 9/13. The
  default English-QA instruction biases the reranker toward prose answer pages
  and under-scores C definition pages.
- **Decision**: MCP parameter exposed (on for concept-heavy ambiguous queries);
  description notes the ~+0.4s latency cost.
- Revisit when: a symbol-tailored rerank instruct is evaluated.

## ADR-7 MD materialization + line mapping (the Read/Grep direct layer)
- Motivation: the last resort when semantic/keyword retrieval fails; agents can
  grep-locate then read precisely.
- Key guarantee: **deterministic serialization** (rebuild sha256-identical) →
  line numbers are long-lived references.
- Design: one md per source doc; meta.json stores parent line ranges;
  kb_read_path maps a canonical path to the range directly (50KB budget).

## ADR-8 SPLADE removal
- Rationale: maintenance cost (GPU-resident service/standalone process/auto
  spawn) vs marginal benefit (limited against BM25F+dense); explicit user
  decision.
- Legacy: the degraded semantics (degraded flag) migrated from SPLADE to the
  dense/rerank channels.

## ADR-9 MCP tool surface = five-tool responsibility chain
- search (main) / kb_read_path (direct read) / kb_read (range continuation) /
  kb_grep (last resort) / kb_structure (orientation).
- Description-writing rules in references/mcp_design.md.
- Evolution record: RRF score → similarity score; dual weights → single weight
  knob; child body → parent full text + child outline. Each evolution synced
  descriptions and the cross-reference chain.

## ADR-10 Golden regression front-loading
- Every ranking-related change (aggregation formula/weights/rerank switch) runs
  the 13-golden × 4-config ablation.
- Triage mantra: abnormal rank → check ① golden substring invalidated by
  folding/refactor ② measurement basis double counting ③ transient network
  degradation — this round's three "999/miss" were all these, none algorithmic.

## ADR-11 Profile v2: executable DSL, the single build-time source of truth
- **Background**: v2's first cut ran build_ir on a hardcoded PARAMS dict (tuned
  for mujoco); the profile's human-reviewed numbers only took effect on the
  query side — exposed when the second corpus (mjlab) landed: the deep-read
  decisions (guide granularity 400/120, api code weight 0.5, author stripping)
  were never consumed.
- **Decision**: chunk_rules upgraded to an executable DSL — `parent_mode
  (entries+sections/sections)`, `parent_max_level`, `fold_below_chars`,
  `promote_deep_over_chars`, `min/max_child_chars`, `code_split_max_chars`,
  `split_code_by_comments`, `entry_boundary_re`, `strip_author_tail` — build_ir
  dropped PARAMS and only reads `_params_for(kb, dtype)`.
- Migration discipline: MuJoCo's three profiles migrated "numbers as-is";
  rebuild products must match the golden-locked version **byte-identically**
  (936/2808, changelog 133) to pass; any semantic drift during migration
  (133→145→97) rolls back.
- Revisit when: a new corpus needs NEW execution semantics (FAQ pairs, giant
  tables) — extend the DSL keys first, then write the profile. Never run
  "descriptive text + hardcoded implementation" dual tracks.

## ADR-12 Multi-KB registry + the real-usage test gate
- **Decision**:
  1. config.KBS registry unifies paths/collection/profiles/examples; every
     pipeline module takes `--kb`; MCP instantiates via the `KB_NAME` env var
     (one codebase, multiple deployment entries).
  2. Tool-description examples are **localized per corpus** (per-KB examples
     registry + injection at registration); acceptance = two-way leak scan to
     zero.
  3. A new corpus's final acceptance = **two-round real-usage test**: round 1
     business questions only (zero tool teaching), round 2 advanced follow-ups
     (send_message, same session), closing interview (tool counts/scores/pain
     points/degraded count); every defect claim verified against files.
- Evidence: mjlab first round answered 4+4 questions at high quality (scores
  8.5–9.5/10); the interview exposed 3 improvements, all landed
  (kb_read_path max_lines / grep zero-hit self-diagnosis / structure outline
  description emphasis).
