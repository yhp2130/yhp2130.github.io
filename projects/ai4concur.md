# AI4Concur — Finance Receipt Intelligence

← [Back to Portfolio](../README.md)

**Role:** AI Engineer / Data Scientist — model design, OCR pipeline, Concur API integration  
**Timeline:** ~6 months  
**Type:** POC → Build-vs-Buy evaluation

---

## Problem

Finance staff processed hundreds of expense receipts monthly for SAP Concur.
Each receipt required manual reading, category assignment, and field entry (date, amount,
merchant, tax, currency). High volume, repetitive, error-prone.

**Goal:** Automate receipt → SAP Concur data entry end-to-end via event-driven pipeline.

---

## Outcomes

| Metric | Value |
|--------|-------|
| Overall POC accuracy | **70%** on test samples |
| Document classes labelled | 7 (Label Studio) |
| Validation flags | 7 rule-based checks |
| Hotel extraction models evaluated | LayoutLM, Donut, Nougat |

---

## Project Phases (Chronological)

### Phase 1 — Ideation & Scope

Scoped the POC to **Singapore travel claims** (taxi, Grab) and **hotel folios** — explicitly
excluding meeting and entertainment receipts. Singapore travel receipts share common format patterns,
making them a tractable starting point before extending to hotel folios.

---

### Phase 2 — Label Studio: Document Classification

Used **Label Studio** to label receipt images into **7 document classes** — separating scanned
physical receipts from digital e-receipts and handling multi-receipt scans:

| Class | Meaning |
|-------|---------|
| `digital_need` | Digital page that **contains** required expense information |
| `digital_no_need` | Digital page that **does not contain** required information |
| `receipt_single` | Scanned copy with a single receipt |
| `receipt_2` | Scanned copy with 1 row, 2 receipts side by side |
| `receipt_3` | Scanned copy with 1 row, 3 receipts side by side |
| `receipt_multiple` | Scanned copy with multiple rows of multiple receipts |
| `vaccination` | Vaccination certificate — out-of-scope, exclude from pipeline |

---

### Phase 3 — Label Studio: Bounding Box Annotation

For multi-receipt scans, used Label Studio's **bounding box drawing tool** to annotate individual
receipt regions. These annotations trained the Faster RCNN model to segment multiple receipts from
a single scan before OCR.

---

### Phase 4 — Model Training

Two models trained in parallel:

**Document Classifier:**
- Started with DenseNet; final model: **SqueezeNet** (lighter, faster, sufficient for 7-class task)
- Input: 224×3px, normalized with ImageNet stats, Adam optimizer
- **Image augmentation:** rotation (±15°), brightness/contrast variation, random crop — simulates mobile photo conditions

**Receipt Detector:**
- **Faster RCNN with ResNet50 backbone** — detects individual receipt bounding boxes in multi-receipt scans

`squeezenet1_0_Adam_Aug1` (classifier) + `resnet50_070822` (Faster RCNN)

---

### Phase 5 — Pipeline Design

Full processing pipeline:

```
Image
  → CNN Classifier (digital vs scanned, single vs multi)
  → [scanned multi] Faster RCNN segmentation → crop individual receipts
  → Rotation correction (EasyOCR confidence-based: 0°/90°/-90°/180°)
  → EasyOCR (custom fine-tuned weights)
  → Regex NER: date extraction (50+ format permutations) + amount extraction
  → 7-flag validator
  → SAP Concur API write OR manual review queue
```

**7 Validation flags** (from codebase):
- Receipt missing
- Amount > SGD30 threshold
- Amount mismatch with claim
- Date mismatch
- Receipt expired (>90 days)
- Non-working day
- Policy timing (OT window check: after 21:00 or before 06:30)

---

### Phase 6 — SAP Concur API: Event-Driven Webhook

Redesigned from batch CSV processing to **event-driven**:
- SAP Concur fires a webhook on expense report submission
- **FastAPI endpoint** (`/event_trigger`) receives the event
- Fetches receipt images via **SAP Concur v4 REST API** (OAuth2 refresh token)
- Triggers pipeline immediately — eliminates batch delay, enables real-time validation feedback

---

### Phase 7 — Hotel Invoice Extension

Hotel folios are structurally different — multi-row itemized charges, table layouts, multi-page.
Repeated the Label Studio annotation process for table region bounding boxes.

Ran a series of experiments on hotel invoice extraction:

| Model | Approach | Outcome |
|-------|---------|---------|
| LayoutLM | Layout-aware transformer for key-value extraction | Did not meet expected performance — hotel formats too wide and varied |
| Donut | End-to-end document understanding (no OCR stage) | Training converged but generalisation poor — format variability too high |

Neither model achieved the extraction quality needed. **Project concluded at this stage** — POC overall accuracy settled at **70%**.

---

### Retrospective Learning

After the project concluded, a similar hotel invoice extraction experiment was conducted using **GPT-4.1 with multimodal (vision) input**. Performance was markedly better — the model handled varied hotel formats without per-vendor fine-tuning.

Key takeaway: **multimodal LLMs are the pragmatic first choice for structured document extraction where layout varies widely**, and specialist fine-tuned models are better reserved for high-volume, narrow-format tasks with sufficient labelled data.

---

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Annotation | Label Studio |
| Image classification | PyTorch SqueezeNet (CNN) |
| Receipt segmentation | PyTorch Faster RCNN (ResNet50) |
| OCR | EasyOCR (custom fine-tuned weights) |
| Field extraction | Regex NER (date + amount) |
| Evaluated (hotel) | LayoutLM · Donut |
| Webhook/API | FastAPI · SAP Concur v4 REST API (OAuth2) |
| Deployment | SAP BTP |
| Training | On-prem GPU · Image augmentation |

---

## Build-vs-Buy Post-Mortem

Finance chose a commercial SaaS for receipt processing. POC value:
- Defined accuracy benchmarks used to **evaluate SaaS vendors**
- Vendor selection informed by field extraction tests
- Technical work shaped the buy decision even without becoming production system

**Key learning:** Build-vs-buy analysis should be included at project start — total cost of
ownership, maintenance burden, and available vendor solutions.

