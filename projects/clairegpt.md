# CODEC  ClaireGPT: Logistics Knowledge Retrieval System

**Role:** Lead AI Engineer  
**Duration:** ~3 years (3 generations)  
**Domain:** Logistics  Process Knowledge  Contract Clauses  

---

## Summary

| Dimension | Detail |
|-----------|--------|
| Generations | CODEC (FYP) → CODEC 2.0 (Linux/GPU) → ClaireGPT (RAG agent) |
| Chunking strategies | 5 evaluated; Agentic chosen |
| Agent tools | 3: abbreviation · knowledge search · contract clauses |
| Embedding | SFT-Mistral (selected via RAGAS) vs SentenceBERT |
| ClaireGPT delivery | <1 month, GitHub Copilot AI-assisted coding |
| Deployment | Nginx · OpenShift · React.js frontend |

---

## Phase 1  CODEC: Student FYP Takeover (Hybrid Semantic Search)

**Stack:** Django  SentenceBERT  TF-IDF · python-pptx  python-docx  OpenShift

Took over a student final year project implementing a hybrid semantic search engine for logistics knowledge documents. The system combined SentenceBERT semantic embeddings with TF-IDF keyword scoring:

```
combined_score = k_coeff  sbert_score + (1 - k_coeff)  tfidf_score
```

Documents were extracted from PowerPoint (pptx) and Word (docx) using python-pptx / python-docx. Computer vision built a parallel image corpus  slide diagrams were separately embedded and searchable alongside text.

A **supervised click and rating boost** applied score adjustments based on past user interactions, providing lightweight personalisation without model retraining.

**Limitation:** All tensors loaded in-memory at startup (pickled embedder, corpus_embeddings, tfidf_matrix). Did not scale past ~1,000 documents. No LLM synthesis  returned raw page snippets.

---

## Phase 2  CODEC 2.0: Linux Refactor + GPU Acceleration

**Trigger:** OpenShift has no Windows container support  
**Change:** Linux refactor + CUDA inference

OpenShift does not support Windows containers. The original Django app used Windows-specific paths and dependencies. Refactored the entire codebase to be Linux-compatible: path handling, file encoding, model loading, and service startup updated for container-native environments.

Added GPU acceleration for SentenceBERT inference:

```python
device = torch.device("cuda:0" if torch.cuda.is_available() else "cpu")
```

**Outcome:** Production-deployed on OpenShift. Same search quality with stable Linux container and GPU-accelerated inference.

---

## Phase 3 — ClaireGPT Use Case: RAG Chatbot Evolution

**Pivot:** From search snippets to conversational knowledge agent  
**Delivery:** <1 month · GitHub Copilot AI-assisted coding

CODEC returned page snippets — users still had to read and synthesise across multiple results manually. The evolution: a full RAG chatbot that retrieves relevant knowledge and synthesises cited answers in natural language.

**Dify** was evaluated as a low-code RAG platform to accelerate delivery, but was not available in the on-prem environment. A custom build was chosen instead, developed with **GitHub Copilot AI-assisted coding** — shipped in under 1 month.

Three knowledge domains required:
- Process documentation (SOPs, policies, guidelines)
- Domain abbreviation definitions (logistics-specific terms)
- Contract clause data (from the BART contract extraction pipeline)

---

## Phase 4  Document Ingestion: Docling + EasyOCR + Multi-Source

**Sources:** Confluence  SharePoint  network folder  
**Fallback:** PDF + EasyOCR for older formats

Built a cron-based ingestion pipeline pulling from three sources, each with a dedicated processor. Docling converts each document to structured markdown, preserving headings, tables, and layout for cleaner downstream chunking.

For older or unsupported file formats, a fallback pipeline converts to PDF first, then runs Docling with EasyOCR (`force_full_page_ocr=True`) to handle scanned or image-heavy pages.

**Why Docling:** Preserves document structure as markdown  gives the chunker and embedder richer context than plain text dumps from raw PDF extraction.

---

## Phase 5  Chunking and Embedding Experimentation

**Strategies evaluated:** Fixed-size  Sentence  Semantic  Recursive  **Agentic (chosen)**  
**Embedding comparison:** SFT-Mistral vs SentenceBERT, evaluated via RAGAS

Documents vary widely  structured SOPs, slides, scanned PDFs, and unstructured text  so no single fixed strategy fits all:

| Strategy | Behaviour | Trade-off |
|----------|-----------|-----------|
| Fixed-size | Splits by token/character count | Fast but cuts mid-sentence |
| Sentence | Splits on sentence boundaries | Better coherence, ignores topic shifts |
| Semantic | Splits on embedding similarity drops | Topic-aware, computationally expensive |
| Recursive | Hierarchical split with fallback delimiters | Reasonable default, misses semantic shifts |
| **Agentic** | LLM identifies topic boundaries, assigns chunk title + summary | Best retrieval quality; slower; some boundary information loss |

**Agentic chunking chosen** (AgenticChunker)  the LLM generates a chunk title and summary per chunk, improving both retrieval precision and answer synthesis.

**RAGAS evaluation:** SFT-Mistral (fine-tuned Mistral embedder) outperformed SentenceBERT on the domain corpus and was selected for production.

Each chunk is pushed to Elasticsearch via Logstash with fields: `vector` (dense embedding), `summary`, `title`.

**Key finding:** Agentic chunking improved retrieval relevance most significantly. LLM-generated chunk titles give the retriever richer signal than raw chunk text.

---

## Phase 6  Agentic RAG: Single Agent with 3 Tools

**Framework:** LangChain tool-calling agent  
**Tools:** 3 specialised tools connecting to different backends

| Tool | Function | Backend |
|------|----------|---------|
| `fetch_abbrev_explanation` | Abbreviation finder | Domain abbreviation API (NTLM auth) |
| `fetch_clm_document_chunks` | Knowledge search | Elasticsearch kNN on vector field |
| `fetch_customer_clauses` | Contract clause search | LLM  SQL  Oracle DB (BART tables) |

The agent receives the user query and conversation history, selects and calls tools (single or multi-turn), then synthesises a cited response.

**Role-based access control:** `TOOL_PERMISSIONS` dict gates which tools each user role may invoke.

**BART integration:** `fetch_customer_clauses` uses LLM-to-SQL generation  the agent passes the natural language query to an LLM which outputs an Oracle SQL statement, executed against the BART-extracted clause database, connecting two independent systems through the agent layer.

---

## Phase 7  Production: Nginx  React.js  Chat History

**Frontend:** React.js  **Reverse proxy:** Nginx  **Platform:** OpenShift

**Two usage patterns:**

- **Client loading (per-user chat history cache):** each user's conversation history is cached client-side and sent with each request  giving the agent full multi-turn context without server-side session management
- **Server loading (QnA collection):** every queryresponse pair with user feedback (via `FeedbackRequest`) is stored server-side in `chat_history/` for future fine-tuning and evaluation

**Outcome:** Production-deployed knowledge chatbot with multi-turn dialogue, role-gated tool access, and structured QnA data collection for continuous improvement.

---

## Retrospective Learning

- Agentic chunking is powerful but slower  for large-scale ingestion, a hybrid approach (semantic split first, then agentic refine on key sections) would balance quality and throughput
- SFT-Mistral embedder is significantly larger than SentenceBERT  deployment cost vs quality trade-off should be re-evaluated as smaller domain-fine-tuned models improve
- LLM-to-SQL generation works well for structured clause lookups but requires prompt engineering to constrain to safe, read-only queries
- Per-user client-side chat history is simple to implement but grows unbounded  a session window or server-side summary would be more robust at scale

---

## Stack

| Category | Tools |
|----------|-------|
| Agentic framework | LangChain |
| Vector store | Elasticsearch  Logstash |
| Embedding | SFT-Mistral  SentenceBERT |
| Chunking | AgenticChunker |
| Document parsing | Docling  EasyOCR  python-pptx  python-docx |
| Database | Oracle DB |
| API | FastAPI |
| Frontend | React.js |
| Infrastructure | Docker  Nginx  OpenShift |
| LLM (on-prem) | On-prem LLM endpoint |
