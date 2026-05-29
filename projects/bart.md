# ICC  BART  Contract Clause Extraction

 [Back to Portfolio](../README.md)  [See also: ClaireGPT](clairegpt.md)

**Timeline:** ~3 years (ICC  BART  BART Pipeline  ClaireGPT integration)  
**Role:** Lead AI Engineer  model design, training, production pipeline, integration

---

## Problem

Logistics contracts are reviewed per cycle  each containing dozens of clauses covering
liability caps, penalty terms, SLA obligations, and payment conditions.
**Manual review took days per cycle** and required specialist knowledge.

---

## Metrics

| Metric | Value |
|--------|-------|
| Clause classifier weighted F1 | 0.8 |
| Contracts manually labelled | 200 (BSA + CAA) |
| Contracts processed per cycle | 100+ |
| MLOps | GitOps  zero manual deployments |

---

## Chronological Phases

### Phase 1  ICC Takeover: Regex + LDA Predecessor

Took over from a colleague. The existing system used **regex pattern matching** combined
with **Latent Dirichlet Allocation (LDA)** for topic modelling on OCR-converted contract text.

Results were poor due to:
- OCR noise from scanned copies (spotted pages, skew, low contrast)
- Legal clause complexity  LDA topic coherence broke down on long, nested, cross-referencing clauses

**Decision:** Abandon LDA. Move to supervised transformer-based classification.

---

### Phase 2  Model Experimentation: BERT, RoBERTa  CUAD

| Model | Training | Observation |
|-------|---------|-------------|
| BERT base | Fine-tuned on labelled contracts | Baseline  reasonable but lower F1 |
| RoBERTa large | Fine-tuned on labelled contracts | Better generalisation across clause styles |
| RoBERTa + CUAD | Domain pre-trained on CUAD | Marginal improvement  logistics contracts differ from CUAD's US legal style |
| RoBERTa-large-MNLI | Fine-tuned (NLI head) | Best overall  **selected for production** |

**Insight:** MNLI pre-training gave RoBERTa strong entailment reasoning for matching clause text to topic labels.

---

### Phase 3 — Data Labelling: 200 Contracts (BSA + CAA)

Manually labelled 200 contracts using Label Studio. Two contract types:

| Type | Description | Challenge |
|------|-------------|-----------|
| BSA (Basic Supply Agreement) | Full standalone logistics contract | Long documents, all 8 topics present |
| CAA (Contract Amendment Agreement) | Overwrites specific clauses in a prior BSA | Sparse labels  only amended clauses appear |

8 topics extracted:

| Topic | Clause Scope |
|-------|-------------|
| qty_tolerance | Acceptable quantity variance before rejection |
| delivery_term | Incoterms and delivery responsibility |
| incoming_inspection | Acceptance testing obligations and timelines |
| liability_cap | Maximum financial exposure per party |
| order_response | Acknowledgment and lead time obligations |
| payment_term | Payment due dates and late payment conditions |
| product_change_notification | Notice period for part/process changes |
| warranty_period | Defect coverage duration and remediation terms |

---

### Phase 4  Transcription: Nougat over OCR

Contract PDFs are structurally similar to academic papers  dense paragraphs, numbered clauses, embedded tables.
Standard OCR (EasyOCR, pytesseract) was sensitive to scanning noise.

Switched to **Nougat** (Meta) which treats pages as images and generates clean text without a separate OCR step.
Post-processors: LaTeX removal, repeated-line deduplication, header/footer removal, line-length filtering.

Pre-processors: Chinese character removal (bilingual contracts), DETR table extraction (tables overpainted white before Nougat).

---

### Phase 5  SetFit vs Supervised Fine-tuning

Tested **SetFit** (ll-mpnet-base-v2 + few-shot contrastive fine-tuning) against standard
supervised fine-tuning of RoBERTa for multilabel multiclass classification.

Both approaches produced comparable F1.

**Decision:** Chose standard supervised fine-tuning — simpler pipeline, no contrastive pair generation,
easier to maintain. SetFit's advantage is mainly when labelled data is scarce; 200 labelled contracts was sufficient.

---

### Phase 6  Pipeline Design: Parallel Transcribe + Classify + Extract

| Stage | Component | Role |
|-------|-----------|------|
| 1. Table detection | DETR (detr-doc-table-detection) | Detect & extract tables; overpaint white so Nougat reads text only |
| 2. Transcription | Nougat | Convert page images  clean paragraph text |
| 3. Classification | RoBERTa-large-MNLI (fine-tuned) | Multilabel multiclass: assign sentences to 8 clause topics |
| 4. Extraction | BART (seq2seq) | Summarise classified clause text into structured output per topic |
| 5. Storage | Oracle DB | Persist extracted clauses with contract ID, topic, sentence, confidence score |

CAA handling: amendment contracts overwrite prior BSA clauses  missing topics fall back to BSA values from the previous cycle.

---

### Phase 7  Production Deployment: Human-in-the-Loop

Deployed with a human-in-the-loop admin interface (Vue.js + Django REST): admins review, amend, and approve AI-extracted clause data before downstream use.

Full GitOps CI/CD: every merge to main triggers tests  linting  Docker build  push to Artifactory. ArgoCD syncs Helm chart through dev  staging  prod on OpenShift. Manual approval gate at staging  prod.

**Outcome:** F1 = 0.8 across 8 topics. Project renamed ICC  BART.

---

### Phase 8  Tool Conversion: ClaireGPT Integration

BART wrapped as a **LangChain tool** within the ClaireGPT agent. Users can ask natural-language questions about contract clauses; the agent invokes the BART tool to retrieve and summarise relevant clause content.

---

### Phase 9  GPT-4.1 Retrospective

Post-deployment experiment using GPT-4.1 for clause extraction on the same contract set.
Results were comparable to the fine-tuned pipeline.

**Learning:** For low-volume structured extraction from varied document formats, multimodal LLMs are the pragmatic first choice. The fine-tuned pipeline remains valuable for high-volume, on-prem, privacy-sensitive deployments.

---

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Annotation | Label Studio |
| Transcription | Nougat |
| Table detection | DETR (detr-doc-table-detection) |
| Classification | RoBERTa-large-MNLI (fine-tuned) |
| Extraction | HuggingFace BART (seq2seq) |
| Experiments | BERT  RoBERTa  CUAD  SetFit |
| Backend | FastAPI  Django REST |
| Frontend | Vue.js |
| Agent integration | LangChain |
| Database | Oracle DB |
| CI/CD | GitLab CI  Artifactory  ArgoCD  Helm  OpenShift |
| Training | On-prem GPU · PyTorch |
