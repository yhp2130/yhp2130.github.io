# DIVA — Distributor Invoice Audit

← [Back to Portfolio](../README.md)

**Timeline:** ~1 year  
**Role:** Lead AI Engineer — LLM extraction pipeline, rule engine design, ABBYY integration  
**Repos:** `diva` / `invoicevouching`

---

## Summary

| Metric | Value |
|--------|-------|
| Extraction approaches evaluated | 2 (LLM, ABBYY) |
| Matching flags per invoice | 6 (SMD, currency, qty, price, SAP#, date) |
| Invoices processed per month | 100+ |

---

## Chronological Phases

### Phase 1 — Use Case: Finance Controller Distributor Reconciliation

Distributors submit monthly invoices for products they have sold to end customers.
The Finance Controller needed to reconcile these invoices against internal
**Sales & Marketing Data (SMD)** from the ERP — verifying that quantities, prices, currency,
and customer identity reported by the distributor matched the company's records.

The process was fully manual: controllers downloaded invoice PDFs, cross-referenced each line
item against ERP exports, and flagged discrepancies by hand. With 100+ invoices per month
from distributors across multiple countries and languages, this was a significant recurring burden.

---

### Phase 2 — Two-Track Exploration: LLM &amp; ABBYY in Parallel

The project split into two independent extraction tracks running in parallel:

| Track | Approach | Owner |
|-------|----------|-------|
| LLM extraction | pymupdf + pytesseract OCR → on-prem LLM (3-call pipeline: table CSV, header JSON, JSON correction) | AI Engineer (this project) |
| ABBYY extraction | ABBYY Vantage cloud API — trained document skill for structured field extraction | Finance developer |

The LLM pipeline handled multilingual invoices (Chinese, German, English) via pytesseract
(`chi_tra+chi_sim+deu+eng`) and pymupdf4llm for layout-aware text. Three LLM calls per invoice:
line-item table (semicolon-delimited CSV), header fields (JSON), JSON correction pass.

---

### Phase 3 — LLM vs ABBYY: Extraction Quality Comparison

The on-prem LLM produced **inconsistent extraction results** — field hallucination, variable JSON
structure, and poor handling of scanned invoices with low OCR confidence. Multi-page invoices and
non-standard layouts degraded consistency further.

ABBYY Vantage — using a trained document skill (`combined_plank3`) — produced structured, consistent
field extraction across invoice formats without prompt engineering or retry logic.

**Decision:** ABBYY chosen as the production extraction engine. LLM approach not deployed.

---

### Phase 4 — Rule Engine: Matching Logic &amp; ABBYY Integration

Designed and implemented `apply_rules()` — comparing ABBYY-extracted invoice data against ERP sales data.
Each invoice produces a structured flag set:

| Flag | Logic |
|------|-------|
| `smd_flag` | Invoice line item found / not found in ERP sales data |
| `currency_flag` | Invoice currency matches ERP reference currency |
| `quantity_flag` | Per-line quantity deviation check (invoice vs ERP) |
| `price_flag` | Per-line price comparison — mean / max / min |
| `sap_number_flag` | 4-state: -1 not found · 0 no match · 1 exact · 2 found in list |
| `date_flag` | Invoice date matches ERP expected date |

**Company name matching** — two-strategy approach:
- **ASCII names:** `thefuzz.partial_ratio` — ≥80 match · 30–79 partial (human review) · <30 mismatch
- **CJK names:** LLM semantic comparison — fuzzy matching fails on Traditional/Simplified/mixed scripts

**Design choice:** Rule-based over ML — finance audit requires explainable decisions ("field X mismatched by Y%"),
not model confidence scores. Rules are also configurable without retraining.

---

### Phase 5 — Production Deployment

The production pipeline (`main_abbyy.py`) runs on schedule: scans the invoice drop folder, calls the
ABBYY Vantage API for each PDF, passes extracted data through `apply_rules()`, and posts structured
match results to **Elasticsearch** via Logstash for the Finance Controller's review dashboard.

A **Vue.js frontend** surfaces exceptions (invoices with flag mismatches) for human review. Fully
matched invoices are auto-approved and routed for payment processing. The system is containerised
with **Docker** and served via FastAPI.

**Outcome:** Finance Controller reconciliation automated for standard invoices. Exception reports
replace manual cross-referencing. Full audit trail stored in Elasticsearch for compliance.

---

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Extraction (production) | ABBYY Vantage (cloud API, trained skill) |
| Extraction (experimental) | On-prem LLM — pymupdf, pymupdf4llm, pytesseract |
| Rule engine | Python (`diva_rules.apply_rules`) |
| Company matching | thefuzz.partial_ratio · LLM fallback (CJK) |
| Reference data | ERP REST API (SMD) |
| Backend | FastAPI |
| Frontend | Vue.js |
| Output | Elasticsearch · Logstash · CSV |
| Containerisation | Docker |
