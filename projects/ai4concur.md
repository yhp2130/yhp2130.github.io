# AI4Concur — Finance Receipt Intelligence

← [Back to Portfolio](../README.md)

**Role:** AI Engineer — model design, OCR pipeline, Concur API integration  
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

Scoped the POC to **Singapore travel receipts** (taxi, Grab, meals) to bound the labeling effort.
Singapore receipts share common format patterns, making them a tractable starting point before
extending to hotel folios.

---

### Phase 2 — Label Studio: Document Classification

Used **Label Studio** to label receipt images into **7 document classes** — separating scanned
physical receipts from digital e-receipts and handling multi-receipt scans:

| Class | Meaning |
|-------|---------|
| `digital_need` | Digital e-receipt with required fields visible |
| `digital_no_need` | Digital doc, no expense-relevant fields |
| `receipt_single` | Single scanned physical receipt |
| `receipt_2` | 2 receipts on one scanned page |
| `receipt_3` | 3 receipts on one scanned page |
| `receipt_multiple` | 4+ receipts on one page |
| `vaccination` | Vaccination cert — out-of-scope, exclude |

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

> Verified from codebase: `squeezenet1_0_Adam_Aug1` (classifier) + `resnet50_070822` (Faster RCNN)

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

Evaluated three document understanding models:

| Model | Approach | Outcome |
|-------|---------|---------|
| LayoutLM | Layout-aware transformer for key-value extraction | Struggled with wide multi-column hotel formats |
| Donut | End-to-end document understanding (no OCR) | Training converged but generalisation poor |
| Nougat | Document parsing model | Did not converge on training data |

**Final approach:** DETR (`detr-doc-table-detection`) for table region detection + SetFit for
few-shot line item classification.

**Root cause of 70% ceiling:** Hotel invoice formats vary significantly across hotel chains —
column widths, item naming conventions, multi-currency rows. Per-vendor templates would not scale.

---

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Annotation | Label Studio |
| Image classification | PyTorch SqueezeNet (CNN) |
| Receipt segmentation | PyTorch Faster RCNN (ResNet50) |
| OCR | EasyOCR (custom fine-tuned weights) |
| Field extraction | Regex NER (date + amount) |
| Hotel table detection | DETR (`detr-doc-table-detection`) |
| Hotel classification | SetFit (few-shot) |
| Evaluated (hotel) | LayoutLM · Donut · Nougat |
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

