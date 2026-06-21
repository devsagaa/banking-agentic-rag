# Banking Agentic RAG — Learning Roadmap

**Mentor mode:** Senior Staff AI Engineer. Concept → architecture → tradeoffs → minimal code → exercise, for every stage. No stage gets implemented end-to-end in one shot — we go in order, and each stage is independently testable before the next begins.

**Scoping note on this document:** You asked for "individual prompts for each task." Writing out every micro-task prompt for all 15 stages right now would mean generating most of the project before you've touched code — which is exactly what you said you don't want. So this doc gives you one **kickoff prompt per stage** — paste it in when you're ready to start that stage, and we'll break it into per-task prompts live, the way the rest of the project has gone.

---

## How this connects to ComplianceRAG

Stages 1, 2, 3, 5, 6, and 7 overlap heavily with what's already built in ComplianceRAG — PyMuPDF ingestion, sentence-transformers embeddings, hybrid dense + BM25 retrieval, raw Bedrock generation, and an evals layer. I don't have full visibility into exactly how far each piece got, so I've marked my assumption per stage below as **[build]**, **[audit]**, or **[extend]**. Correct me wherever I've guessed wrong — it changes whether we're writing new code or stress-testing what exists.

This roadmap also deliberately reintroduces things you've already touched (OCR, table extraction, query rewriting, reranking) because the original build was about *getting a working RAG pipeline fast without LangChain abstractions*. This one is about *hardening every layer to a production bar and wrapping it in agentic orchestration* — different goal, so some "already done" pieces will get rebuilt to a stricter standard.

---

## Project Structure

```
banking-agentic-rag/
├── data/
│   ├── raw_pdfs/            # RBI circulars, Basel III text, internal policy docs
│   ├── processed/           # extracted text, tables, chunks (per stage 1-2 output)
│   └── eval_sets/           # benchmark Q&A pairs, adversarial test cases
├── src/
│   ├── ingest/               # Stage 1 — PDF/OCR/table extraction
│   ├── chunking/              # Stage 2
│   ├── embeddings/             # Stage 3
│   ├── vectorstore/            # Stage 4 — pgvector layer
│   ├── retrieval/              # Stages 5, 7, 8, 9 — basic, hybrid, rewrite, rerank
│   ├── generation/              # Stage 6 — prompting, citations
│   ├── security/                # Stage 10 — input validation, PII, authz
│   ├── graph/                    # Stages 11, 12 — LangGraph nodes/edges, agentic loop
│   ├── evaluation/                # Stages 13, 14 — grounding checks, eval harness
│   └── api/                       # Stage 15 — FastAPI, tracing, deployment
├── tests/                          # mirrors src/, one test module per stage
├── configs/                         # model names, thresholds, retry limits
├── docker/
└── README.md
```

Each `src/` subpackage should run and have its own tests *before* the next stage's package imports it. That's what "independently testable" means in practice — `pytest tests/ingest/` should pass with zero dependency on `src/graph/` even existing yet.

---

## Master Roadmap

| # | Stage | One-line goal | Status |
|---|-------|----------------|--------|
| 1 | PDF Processing | Clean text + tables + OCR from real banking PDFs | [audit/extend] |
| 2 | Chunking | Split docs without breaking clauses from their conditions | [audit] |
| 3 | Embeddings | Vectorize chunks, understand what the vectors encode | [audit] |
| 4 | Vector Database | Persist + query embeddings at scale with metadata filters | [build] |
| 5 | Basic Retrieval | Dense top-k retrieval, measured against a real eval set | [audit] |
| 6 | Basic RAG | Grounded, cited answers — refuses when context is insufficient | [audit/extend] |
| 7 | Hybrid Search | BM25 + dense, fused, tested on exact-term banking queries | [audit/extend] |
| 8 | Query Rewriting | Expansion, clarification, decomposition before retrieval | [build] |
| 9 | Reranking | Cross-encoder rerank for top-k precision | [build] |
| 10 | Security | Prompt injection, PII, authorization-aware retrieval | [build] |
| 11 | LangGraph | Re-platform the pipeline as a state machine | [build] |
| 12 | Agentic RAG | Retrieval grading + self-correction retry loop | [build] |
| 13 | Hallucination Detection | Sentence-level groundedness checking | [build] |
| 14 | Evaluation | Versioned retrieval + generation metric scorecard | [extend] |
| 15 | Production Readiness | FastAPI + tracing + failure handling + cost tracking | [build] |

---

## Stage 1 — PDF Processing

**Goal:** Convert raw banking PDFs (RBI circulars, Basel III text, internal credit/treasury policies) into clean structured text and tables, with OCR fallback for scanned pages.

**Learning objectives**
- Text-layer vs scanned PDFs, and how to detect which you're dealing with
- OCR fundamentals (layout detection, Tesseract vs managed services)
- Why table extraction is a separate problem from text extraction
- Preserving document structure (headers, sections) for downstream chunking

**Tasks**
1. Build a classifier: does this PDF have a text layer, or does it need OCR?
2. Extract raw text + page metadata with PyMuPDF.
3. Add OCR fallback for scanned pages (pytesseract, or Textract if you want managed).
4. Extract tables separately — structured rows/columns, not flattened text.
5. Build a document inventory report: page count, OCR ratio, table count per doc.

**Milestone:** Given 5 real banking PDFs (mixed clean/scanned), produce structured JSON per doc with preserved headers, extracted tables, and a coverage report.

**Common mistakes**
- Flattening tables into prose text — corrupts numeric data once embedded
- Ignoring headers/footers — "Confidential — Internal Use Only" pollutes every chunk
- Mishandling multi-column layouts — text gets interleaved out of order

**Interview questions**
- How would you handle a 200-page regulatory PDF mixing scanned and digital pages?
- Why is table extraction harder than text extraction, and how do you preserve table semantics for retrieval?
- Textract vs open-source OCR — what's the tradeoff?

**Stretch goal:** Extract embedded charts/diagrams as images and caption them with a vision model (multimodal RAG).

**Libraries:** PyMuPDF, pytesseract, pdfplumber/camelot, AWS Textract (optional)

**Kickoff prompt:** *"Let's start Stage 1: PDF Processing. I have [N] sample banking documents: [list]. Audit what I already built in PyMuPDF for ComplianceRAG against this stage's bar — OCR fallback, table extraction, structure preservation — then walk me through closing the gaps. Concept first, then a skeleton I implement myself."*

---

## Stage 2 — Chunking

**Goal:** Split documents into retrieval-sized units that don't separate a rule from its exceptions or conditions.

**Learning objectives**
- Fixed-size vs semantic vs structural chunking
- Chunk size/overlap tradeoffs, and why naive token chunking breaks regulatory text
- Metadata enrichment per chunk (doc_id, section_path, page, doc_type, effective_date)

**Tasks**
1. Implement a fixed-size token chunking baseline.
2. Implement structure-aware chunking using Stage 1's preserved headers/clauses.
3. Add overlap and measure how much boundary information it actually saves.
4. Attach rich metadata to every chunk.
5. Manually rate coherence on a sample of chunks.

**Milestone:** For 10 sample clauses, show that no chunk separates a rule from its exception/condition.

**Common mistakes**
- Chunking before cleaning — table fragments land mid-sentence
- Overlap that duplicates entire clauses instead of preserving boundary context
- Losing the page/section reference needed later for citations

**Interview questions**
- How do you choose chunk size for regulatory/legal text vs conversational data?
- What's the cost of overlap at scale?
- How would you evaluate whether a chunking strategy is actually good?

**Stretch goal:** Semantic chunking — use embedding similarity between adjacent sentences to find natural breakpoints instead of fixed boundaries.

**Libraries:** tokenizer of choice (tiktoken/HF), custom structural splitter

**Kickoff prompt:** *"Stage 2: Chunking. Here's how ComplianceRAG currently chunks documents: [describe]. Is this structure-aware or fixed-size? Help me evaluate it against clause-preservation, then improve it."*

---

## Stage 3 — Embeddings

**Goal:** Convert chunks into vectors and understand what they actually encode.

**Learning objectives**
- How embedding models are trained (contrastive learning, intuition level)
- Similarity metrics — cosine vs dot product vs L2 — and when the choice matters
- Domain shift: general embeddings vs financial/regulatory text
- Batch embedding for cost and throughput

**Tasks**
1. Baseline with sentence-transformers.
2. Compare against a Bedrock embedding model (Titan or Cohere) on retrieval quality.
3. Nearest-neighbor sanity check — pick a known regulatory term, verify retrieved chunks make sense.
4. Benchmark embedding latency/cost at corpus scale.

**Milestone:** An embedding pipeline with a swappable model interface (sentence-transformers ↔ Bedrock with no downstream code change) and a written sanity-check report.

**Common mistakes**
- Embedding chunks with boilerplate/headers still attached
- Forgetting to normalize vectors when using cosine similarity
- Assuming higher dimensionality always means better retrieval

**Interview questions**
- Walk me through what an embedding vector actually represents.
- Why might a general-purpose embedding model underperform on banking/legal text?
- When does cosine vs dot product similarity actually change your results?

**Stretch goal:** Evaluate a domain-adapted embedding model using contrastive pairs built from an RBI glossary.

**Libraries:** sentence-transformers, boto3 (Bedrock Titan/Cohere)

**Kickoff prompt:** *"Stage 3: Embeddings. I'm using sentence-transformers in ComplianceRAG today. Let's benchmark it against a Bedrock embedding model on my eval queries and decide if it's worth swapping."*

---

## Stage 4 — Vector Database

**Goal:** Persist and query embeddings efficiently with metadata filtering, at a scale that holds up.

**Learning objectives**
- ANN indexing (HNSW, IVF) at a conceptual level
- Index build time vs query time tradeoffs
- Metadata filtering alongside vector search
- Why pgvector is a natural fit if you're already comfortable in SQL

**Tasks**
1. Stand up pgvector; design schema (chunk_id, doc_id, embedding, metadata columns).
2. Implement insert + similarity query.
3. Add metadata filtering (e.g. restrict to `doc_type = 'credit_risk_guideline'`).
4. Benchmark query latency as corpus grows: 100 → 10k → 100k chunks.
5. Compare against one alternative (Chroma or Qdrant) for tradeoffs, even if you don't switch.

**Milestone:** Filtered similarity query returns ranked results in under 200ms at your test corpus size.

**Common mistakes**
- Not indexing the embedding column — full table scan
- Storing metadata as unstructured JSON when it should be filterable columns
- Forgetting ANN search is approximate — recall is never 100%

**Interview questions**
- Explain HNSW indexing at a high level.
- When would you choose pgvector over a dedicated vector DB?
- How do you do metadata filtering at scale without killing recall?

**Stretch goal:** Index re-build/versioning strategy for when source regulations get amended.

**Libraries:** pgvector, SQLAlchemy/psycopg2

**Kickoff prompt:** *"Stage 4: Vector Database. I want to stand up pgvector from scratch — walk me through schema design first, before any code."*

---

## Stage 5 — Basic Retrieval

**Goal:** Implement dense semantic retrieval end-to-end, and define "good retrieval" with numbers before adding complexity.

**Learning objectives**
- Top-k mechanics and similarity score interpretation
- The precision/recall tension at different k values

**Tasks**
1. Query → embed → search → top-k chunks.
2. Build a manual eval set: 10–15 banking questions with known correct source chunks.
3. Measure recall@k for k = 3, 5, 10.
4. Inspect failure cases — what kinds of queries retrieve badly?

**Milestone:** A baseline recall@5 number on your manual eval set, plus a written list of failure patterns.

**Common mistakes**
- Evaluating only on easy questions
- Not separating retrieval failures from generation failures
- Trusting similarity scores as calibrated confidence

**Interview questions**
- How do you debug a RAG system when answers are wrong — retrieval or generation?
- What does a similarity score of 0.82 actually tell you?

**Stretch goal:** Visualize the embedding space (UMAP/t-SNE) for eval queries vs corpus to see where retrieval clusters fail.

**Libraries:** numpy, Stage 3/4 pipeline

**Kickoff prompt:** *"Stage 5: I want to build a real eval set of 15 banking questions and get a recall@k baseline before touching anything else."*

---

## Stage 6 — Basic RAG

**Goal:** Close the loop — retrieved chunks → LLM → grounded, cited answer.

**Learning objectives**
- Prompt construction for context-grounded answering
- "Answer from context" vs "answer from parametric knowledge" — and why the distinction is dangerous to blur in banking
- Basic citation mechanics: mapping claims back to source chunks

**Tasks**
1. Build a prompt template that forces answering only from provided context.
2. Add an explicit "insufficient context" refusal path.
3. Add citation tags linking claims to chunk_id/page.
4. Run against your Stage 5 eval set — measure false "I don't know"s and false confident answers.

**Milestone:** A real banking question (e.g. minimum capital adequacy ratio under Basel III) answered end-to-end with verifiable citations.

**Common mistakes**
- Prompts that let the model fall back on parametric knowledge — looks right, may be outdated or wrong
- No refusal mechanism for insufficient context
- Citations that are fabricated rather than verified against retrieved chunks

**Interview questions**
- How do you prevent an LLM from answering from training data instead of retrieved context?
- How would you design citation *verification*, not just citation generation?

**Stretch goal:** Self-consistency check — generate the answer twice with shuffled retrieval order, compare.

**Libraries:** raw Bedrock API

**Kickoff prompt:** *"Stage 6: audit ComplianceRAG's generation prompt against context-only grounding and a hard refusal path, then let's tighten it."*

---

## Stage 7 — Hybrid Search

**Goal:** Combine dense and sparse (BM25) retrieval — banking queries need exact matches on circular numbers and defined terms as much as conceptual matches.

**Learning objectives**
- Why pure dense retrieval fails on exact terms ("Section 35A", "RBI/2023-24/15")
- BM25 mechanics
- Score fusion: Reciprocal Rank Fusion (RRF) vs weighted sum

**Tasks**
1. Implement BM25 over your chunk corpus.
2. Implement RRF to combine BM25 + dense rankings.
3. Build a small eval set split into "exact term" vs "conceptual" queries.
4. Compare recall@5 dense-only vs hybrid, per query type.

**Milestone:** Hybrid beats dense-only specifically on the exact-term subset, with no regression on conceptual queries.

**Common mistakes**
- Naively weighting dense/sparse scores without normalizing ranges first
- Testing only on query types where hybrid was never going to matter
- BM25 tokenization that mangles regulatory codes/numbers

**Interview questions**
- Why add BM25 to a system that already has dense embeddings?
- Explain RRF and why it's preferred over naive score averaging.
- Give a query type where pure dense retrieval is guaranteed to fail.

**Stretch goal:** A query classifier that routes between dense-only, sparse-only, and hybrid based on query type.

**Libraries:** rank_bm25 or custom BM25

**Kickoff prompt:** *"Stage 7: I already have hybrid dense+BM25 in ComplianceRAG. Let's build the exact-term-vs-conceptual eval split and pressure-test it properly."*

---

## Stage 8 — Query Rewriting

**Goal:** Transform raw queries into better retrieval queries before they hit search.

**Learning objectives**
- Query expansion vs rewriting vs decomposition
- HyDE (Hypothetical Document Embeddings)
- When rewriting adds latency/cost for no retrieval gain

**Tasks**
1. Simple expansion — acronym handling ("CAR" → "Capital Adequacy Ratio").
2. LLM-based rewriting using conversation history for ambiguous queries.
3. Query decomposition for multi-part questions.
4. A/B test rewritten vs raw queries on your eval set.

**Milestone:** Measurable recall improvement on multi-part/ambiguous questions, with a logged before/after comparison.

**Common mistakes**
- Rewriting that adds cost without measurable retrieval gain
- Over-expanding until the query drifts from user intent
- Not handling follow-up questions ("what about for NBFCs?") using conversation context

**Interview questions**
- When does query rewriting hurt more than it helps?
- Explain HyDE and its tradeoffs.
- How do you handle follow-up questions in a RAG system?

**Stretch goal:** Implement HyDE — embed a hypothetical generated answer instead of the raw query.

**Libraries:** raw Bedrock API

**Kickoff prompt:** *"Stage 8: Query Rewriting. Let's start with acronym/term expansion for banking-specific shorthand, then add LLM-based rewriting for ambiguous follow-ups."*

---

## Stage 9 — Reranking

**Goal:** Improve top-k precision with a cross-encoder reranker after initial retrieval.

**Learning objectives**
- Bi-encoder vs cross-encoder tradeoffs (speed vs accuracy)
- Retrieve-broad-then-rerank-narrow as a pattern
- Retrieval grading as the conceptual precursor to LangGraph's grading node (Stage 12)

**Tasks**
1. Retrieve top-20 with hybrid search.
2. Rerank with a cross-encoder down to top-5.
3. Measure precision@5 before/after on your eval set.
4. Benchmark the added latency cost.

**Milestone:** Demonstrable precision improvement at top-5, with a noted latency number that's still acceptable for production.

**Common mistakes**
- Reranking too few candidates — no room to actually improve
- Trusting an off-the-shelf reranker blindly on a domain it wasn't tuned for
- Ignoring latency cost when planning for production

**Interview questions**
- Why not just use a cross-encoder for retrieval directly, skipping the two-stage approach?
- What's the latency/accuracy tradeoff of reranking 20 vs 100 candidates?

**Stretch goal:** A custom reranking signal combining recency (regulation effective date) with relevance — newer regulations should outrank superseded ones.

**Libraries:** sentence-transformers CrossEncoder, or Cohere Rerank

**Kickoff prompt:** *"Stage 9: Reranking. Walk me through cross-encoder reranking conceptually, then let's add it on top of my hybrid retrieval and measure precision@5."*

---

## Stage 10 — Security

**Goal:** Harden the pipeline against malicious input, data leakage, and unauthorized access before agentic complexity makes debugging harder.

**Learning objectives**
- Direct vs indirect prompt injection — and why RAG is especially exposed to indirect injection (malicious instructions hiding inside retrieved documents)
- PII detection: regex/NER vs LLM-based
- Authorization-aware retrieval — filtering at the vector-search layer, not post-hoc on generated text
- Output filtering as defense-in-depth

**Tasks**
1. Input validation layer — length limits, injection pattern detection.
2. PII detection on both query and generated output.
3. Authorization-aware retrieval — metadata-filtered search (a junior analyst can't retrieve board-only treasury docs).
4. Adversarial test set — inject fake instructions inside a planted "policy doc" and see if retrieval can hijack the model.
5. Output filter blocking responses with PII or unauthorized-content markers.

**Milestone:** A 10+ case adversarial test suite passes — every injection/leakage attempt is blocked or safely handled, with failures logged.

**Common mistakes**
- Filtering only the user's query while ignoring that retrieved documents can carry injected instructions
- Doing authorization as a filter on generated text instead of filtering at retrieval time
- Regex-only PII detection that misses context-dependent PII (e.g. bare account numbers)

**Interview questions**
- Direct vs indirect prompt injection — why is RAG especially vulnerable to the indirect kind?
- How would you design document-level access control for a RAG system serving multiple roles?
- Where in the pipeline should PII filtering happen, and why does placement matter?

**Stretch goal:** A red-team harness that auto-generates injection variants and runs as a regression suite on every pipeline change.

**Libraries:** Microsoft Presidio (PII), custom injection-pattern classifier, vector DB metadata filtering

**Kickoff prompt:** *"Stage 10: Security. Let's start with indirect prompt injection — I want to plant adversarial instructions inside a fake policy doc and see if my current pipeline is vulnerable, before we build any defenses."*

---

## Stage 11 — LangGraph

**Goal:** Learn LangGraph as an orchestration layer and re-platform the linear pipeline onto it.

**Learning objectives**
- Graph state design (TypedDict/Pydantic schema)
- Nodes as pure functions
- Conditional edges for routing
- Why a graph beats a chain of function calls once you need loops, retries, or branches

**Tasks**
1. Define your state schema (query, rewritten_query, retrieved_chunks, graded_chunks, answer, grounding_score, etc.).
2. Wrap each Stage 1–10 component as a node.
3. Wire linear edges first — replicate the existing pipeline exactly.
4. Add your first conditional edge (e.g. route to "insufficient context" if all retrieval scores are below threshold).
5. Visualize the graph.

**Milestone:** The full pipeline runs as a LangGraph graph with at least one conditional branch, producing output identical to the pre-LangGraph version (regression-tested).

**Common mistakes**
- Treating LangGraph as magic instead of recognizing it's a state machine you could hand-roll
- Overcomplicating state — storing everything instead of just what nodes actually need
- Not handling node failures — a thrown exception silently corrupting state downstream

**Interview questions**
- What problem does LangGraph solve that a simple function pipeline doesn't?
- Walk me through your state schema design — why this and not more/less?
- How do conditional edges differ from a plain if/else in a regular function?

**Stretch goal:** A human-in-the-loop interrupt — pause the graph for approval before answering high-risk query categories.

**Libraries:** langgraph

**Kickoff prompt:** *"Stage 11: LangGraph. Before any code, let's design the state schema together for my pipeline — what actually needs to live in state vs what can stay local to a node."*

---

## Stage 12 — Agentic RAG

**Goal:** Add self-correction — retrieval grading, query reformulation on failure, adaptive retrieval — using LangGraph's branching and looping.

**Learning objectives**
- Retrieval grading (LLM-as-judge scoring chunk relevance pre-generation)
- The Corrective RAG (CRAG) pattern
- The agentic loop: when to retry, when to give up, when to ask for clarification

**Tasks**
1. Build a "grade documents" node scoring each retrieved chunk relevant/irrelevant.
2. Add a conditional edge: too few relevant chunks → route back to query rewriting and retry, with a bounded retry count.
3. Add a fallback path (web search, or escalate to human) if internal retrieval repeatedly fails — relevant in banking, where some queries shouldn't be answered from internal docs alone.
4. Test the full loop on deliberately hard/ambiguous queries.

**Milestone:** A query that fails on first retrieval, gets reformulated by the graph, and succeeds on retry — with the full trace logged.

**Common mistakes**
- Unbounded retry loops — runaway cost and latency risk
- Grading with the same model/prompt style as generation — correlated blind spots
- Not logging the retry path, making failures undebuggable later

**Interview questions**
- Walk me through the Corrective RAG (CRAG) pattern.
- How do you prevent an agentic retry loop from running away in cost or latency?
- What's the failure mode when your grading node and generation node share the same blind spots?

**Stretch goal:** Adaptive k — let the graph decide how many chunks to retrieve based on query complexity instead of a fixed k.

**Libraries:** langgraph, Stage 9's reranker as part of the grading signal

**Kickoff prompt:** *"Stage 12: Agentic RAG. Let's design the retrieval-grading node first — what should 'relevant' mean for a banking policy chunk, concretely?"*

---

## Stage 13 — Hallucination Detection

**Goal:** Catch ungrounded claims before they reach the user — a wrong "fact" about a regulation has real consequences in banking.

**Learning objectives**
- Faithfulness/groundedness as distinct from correctness
- NLI-based and LLM-as-judge grounding checks
- Claim decomposition — checking each sentence against context separately

**Tasks**
1. Build a groundedness checker: for each sentence in the answer, verify it's supported by its cited chunk.
2. Add a confidence/grounding score to every response.
3. Define a threshold — low-grounding answers get flagged or blocked, not shown as-is.
4. Build an adversarial eval set of answers with deliberately injected unsupported claims; measure catch rate.

**Milestone:** Your checker catches a defined target (e.g. 90%+) of injected unsupported claims, with an acceptable false-positive rate on genuinely grounded answers.

**Common mistakes**
- Checking groundedness against the whole context instead of the specific cited chunk — too lenient
- Using the same model for generation and grounding-checking without controlling for shared bias
- Treating fluency/confidence as a proxy for groundedness

**Interview questions**
- What's the difference between faithfulness and factual correctness?
- How do you build a hallucination detector that doesn't just trust the model that generated the answer?
- How do you handle false-positive cost in banking — flagging a correct answer as ungrounded?

**Stretch goal:** Claim-level citation verification — each sentence traceable to a specific source span, not just "is the answer grounded overall."

**Libraries:** NLI cross-encoder, or LLM-as-judge via raw Bedrock

**Kickoff prompt:** *"Stage 13: let's build the adversarial eval set first — answers with deliberately injected unsupported claims — before we build the detector that has to catch them."*

---

## Stage 14 — Evaluation

**Goal:** A systematic, repeatable harness covering retrieval and generation quality — every pipeline change gets measured, not vibes-checked.

**Learning objectives**
- Retrieval metrics: recall@k, precision@k, MRR
- Generation metrics: faithfulness, answer relevance, context relevance
- Benchmark construction (synthetic + curated)
- LLM-as-judge design and its known biases (position bias, verbosity bias)

**Tasks**
1. Build a benchmark: 50+ banking Q&A pairs with gold-source chunks, spanning easy/conceptual/exact-term/multi-hop categories.
2. Implement retrieval metrics against the benchmark.
3. Implement generation metrics — your own LLM-as-judge harness (consistent with avoiding heavy abstractions), cross-checked against RAGAS.
4. Run the full suite end-to-end, produce a scorecard.
5. Re-run after Stages 7–13 to show measured, not claimed, improvement.

**Milestone:** A versioned scorecard you can re-run in under N minutes and diff against the previous run.

**Common mistakes**
- Evaluating only on easy/synthetic questions that don't reflect real usage
- Using the same LLM for judging as for generation, without sanity-checking the judge's own accuracy
- A single aggregate score instead of a breakdown by query category

**Interview questions**
- How do you validate that your LLM-as-judge is actually a good judge?
- Walk me through building a benchmark dataset for a domain with no existing labeled data.
- What's the difference between offline eval and production monitoring?

**Stretch goal:** A CI regression gate — the eval suite runs on every PR and blocks merges if a metric drops below threshold.

**Libraries:** RAGAS or DeepEval (cross-check), your own LLM-as-judge harness

**Kickoff prompt:** *"Stage 14: I want to extend my existing evals layer into a full benchmark — let's design the 50-question set first, with category labels, before touching metric code."*

---

## Stage 15 — Production Readiness

**Goal:** Take the system from "works in my notebook" to deployable and operable.

**Learning objectives**
- Observability for LLM pipelines — tracing every node's input/output, latency, cost
- Structured logging vs LLM-specific tracing
- Failure handling: timeouts, retries, circuit breakers, graceful degradation
- Per-query cost tracking

**Tasks**
1. Wrap the LangGraph pipeline in a FastAPI service with proper async handling.
2. Add Langfuse/OpenTelemetry tracing across every node.
3. Add structured logging with correlation IDs per request.
4. Add timeout + retry + circuit breaker around external calls (Bedrock, vector DB).
5. Add per-query cost tracking (tokens in/out × model pricing).
6. Containerize with Docker; write a deployment runbook.

**Milestone:** A locally-deployed FastAPI service handling a banking question end-to-end, full tracing visible in Langfuse, and graceful handling of a simulated Bedrock timeout.

**Common mistakes**
- No graceful degradation — one slow node times out the entire request with no fallback
- Logging without correlation IDs — untraceable across 8+ graph nodes
- Ignoring cost tracking until the AWS bill arrives

**Interview questions**
- How do you trace a single request through a multi-node agentic pipeline in production?
- What's your strategy when a downstream dependency (Bedrock, vector DB) is slow or down?
- How do you track and control LLM cost per request at scale?

**Stretch goal:** Canary/shadow deployment — route a percentage of traffic to a new pipeline version and compare eval scores before full rollout.

**Libraries:** FastAPI, Langfuse, OpenTelemetry, Docker

**Kickoff prompt:** *"Stage 15: I want to wrap the graph in FastAPI with tracing first, then layer in failure handling and cost tracking."*

---

## Using this doc

Pick a stage, paste its kickoff prompt (edited with your actual specifics) as your next message, and we go deep on that stage only — concept, architecture, tradeoffs, minimal code, exercises for you to implement, review. Don't jump ahead to Stage 11 because LangGraph is the exciting part; the security stage (10) is the one most likely to actually come up as a hard interview question, and it only makes sense once 1–9 are solid.
