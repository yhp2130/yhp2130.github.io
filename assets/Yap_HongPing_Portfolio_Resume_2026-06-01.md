# Yap Hong Ping

Singapore PR  |  +65-97773450  |  [yhp2130@gmail.com](mailto:yhp2130@gmail.com)  |  [linkedin.com/in/yaphongping](https://linkedin.com/in/yaphongping)  |  [yhp2130.github.io](https://yhp2130.github.io)

Agentic AI  |  RAG  |  NLP  |  Computer Vision  |  MLOps

## Applied AI Engineer  |  Data Scientist  |  8+ Years Production AI

Full-stack AI Engineer specializing in production-grade NLP, RAG, and agentic AI systems. Built and deployed LangChain ReAct agents serving 100+ queries/day across 1,000+ enterprise documents. Deep experience in chunking strategy optimization, SFT-Mistral embedder fine-tuning, and end-to-end ML lifecycle management using Docker, Kubernetes, and GitOps.

## Key Strengths

- **Data Science:** Frames ambiguous problems into measurable AI objectives; iterates with stakeholders from POC to production
- **AI Application:** Delivers ML, Deep Learning (CV and NLP), and Generative AI (LLM/RAG) solutions across domains
- **Full-Stack Deployment:** Integrates models into web applications (React.js, Vue.js, Streamlit) and APIs (FastAPI)
- **Stakeholder Engagement:** Agile delivery with regular demos, retrospectives, and cross-functional collaboration

## Core Skills  |  Python
- **ML / Deep Learning:** PyTorch, TensorFlow, Scikit-learn, HuggingFace (BART, RoBERTa-MNLI)
- **Agentic AI and RAG:** LangChain (ReAct, tool-calling), OpenAI AsyncOpenAI, Elasticsearch kNN/HNSW, RAGAS evaluation
- **Backend and Full-Stack:** FastAPI, FastMCP, React.js (TypeScript/SSE), Vue.js, Streamlit
- **MLOps:** GitLab CI/CD, ArgoCD, Helm, Artifactory, OpenShift (dev/staging/prod)
- **Data:** RDBMS (Oracle SQL, MySQL), Vector DB (Elasticsearch kNN/HNSW), Cloudera Hadoop
- **AI Platforms:** Datarobot, Dataiku, Dify, SAP Joule, Copilot Studio, GitHub Copilot

## PROFESSIONAL EXPERIENCE

### Infineon Technologies Asia Pacific - Staff Analyst, Data Scientist  (Nov 2021-Present)

**Agentic Process Knowledge RAG**
- Delivered in under 2 weeks using GitHub Copilot SpecKit; 2 knowledge sources (Confluence via internal semantic search, ARIS via REST API to local CSV with hourly refresh)
- Architected production multi-agent system with **3 execution modes:** Direct path (utility tasks, no citations), Fast path (sequential: ARIS to IFX to Checker), ReAct path (ExecutionPlan/SubGoal dependency graph, asyncio parallelism, SufficiencyEvaluator re-planning)
- Designed Planner to 5 Phase Sub-Agents to Checker architecture; citation-gated checker logic; query reconstruction for vague inputs; per-agent YAML config (model, temperature, tool permissions)
- React.js + TypeScript frontend with SSE streaming; per-message feedback (thumbs up/down + comment) stored in SQLite
- **CI/CD:** GitLab CI/CD to ArgoCD to Helm to OpenShift (dev/staging/prod)
- **Stack:** Python, OpenAI AsyncOpenAI, FastAPI, Pydantic, React.js, SQLite

**Document Retrieval and Agentic RAG Chatbot**
- 3 iterative enhancements over 3 years: CODEC (TF-IDF+SBERT hybrid, snippets) to CODEC 2.0 (Linux/GPU for OpenShift) to ClaireGPT (single-agent RAG with tool-calling)
- **96% query time reduction** (2hr to 5min); 100+ queries/day in production; 1,000+ enterprise documents
- Benchmarked 5 chunking strategies (fixed, sentence, semantic, recursive, agentic); selected agentic chunking + SFT-Mistral embedder via RAGAS evaluation
- 3 role-gated LangChain tools: `fetch_abbrev_explanation` (abbreviation API), `fetch_clm_document_chunks` (Elasticsearch kNN), `fetch_customer_clauses` (LLM to SQL to Oracle - BART tables)
- Document ingestion: Docling to structured markdown; EasyOCR fallback; cron-based from Confluence, SharePoint, network folder
- Chat history: client-side per-user cache (multi-turn) + server-side QnA collection (feedback for fine-tuning)
- **Stack:** Python, LangChain, Elasticsearch kNN (HNSW dense_vector), SFT-Mistral, Docling, FastAPI, React.js, Nginx, OpenShift

**NLP Contract Clause Extraction**
- 3 years (takeover to BART pipeline to ClaireGPT tool integration); 200 labelled contracts (BSA + CAA); weighted F1=0.8; 100+ contracts/cycle
- Evaluated BERT, RoBERTa, CUAD, SetFit; selected RoBERTa-large-MNLI (MNLI pre-training gave strong entailment reasoning)
- Pipeline: DETR table detection (overpaint white) to Nougat transcription to RoBERTa classifier (8 topics) to BART seq2seq extraction to Oracle DB
- 8 clause topics: qty_tolerance, delivery_term, incoming_inspection, liability_cap, order_response, payment_term, product_change_notification, warranty_period
- Human-in-the-loop: Vue.js + Django REST admin for review/approve; wrapped as LangChain tool in ClaireGPT
- **Stack:** Python, HuggingFace (RoBERTa-MNLI, BART), PyTorch, Nougat, DETR, FastAPI, Vue.js, Django, Oracle DB

**Travel Expense Receipt Intelligence**
- 6 months POC to Build-vs-Buy evaluation; 70% overall POC accuracy
- SqueezeNet CNN (7 doc classes) + Faster RCNN (ResNet50) + EasyOCR (custom fine-tuned) + regex NER (50+ date formats) + 7-flag validator
- 7 validation flags: receipt missing, amount over SGD30, amount mismatch, date mismatch, receipt expired (over 90 days), non-working day, policy timing (OT window)
- SAP integration: event-driven webhook on expense submission; OAuth2; SAP Concur v4 REST API; SAP BTP deployment
- Evaluated LayoutLM, Donut for hotel folios - neither generalised; GPT-4.1 multimodal postmortem confirmed LLM-first for varied layouts
- **Stack:** Python, PyTorch (SqueezeNet, Faster RCNN), EasyOCR, Label Studio, FastAPI, SAP Concur API, SAP BTP

**Distributor Invoice Audit Automation**
- 1 year; rule-based invoice vouching for AP reconciliation (100+ invoices/month)
- Two-track exploration: LLM extraction vs ABBYY Vantage; chose ABBYY for consistent structured output
- 6 matching flags: smd_flag, currency_flag, quantity_flag, price_flag (mean/max/min), sap_number_flag (4-state), date_flag
- Company matching: ASCII via thefuzz.partial_ratio; CJK via LLM semantic comparison (fuzzy fails on Traditional/Simplified/mixed)
- Design choice: rule-based over ML - finance audit requires explainable decisions, not confidence scores
- **Stack:** Python, ABBYY Vantage, FastAPI, Vue.js, Elasticsearch, Logstash, Docker

### PSA Corporation - Principal Engineer, Smart Systems Solutions  (Jul 2018-Nov 2021)

- **Predictive Maintenance (Maritime Equipment):** Semi-supervised pipeline (anomaly detection to clustering to classification) on time-series sensor data; reduced unplanned downtime
- **CV Equipment Monitoring (POC):** Object detection for structural crack and roller alignment inspection; deployed on edge devices
- **Cloudera Hadoop data pipelines** for large-scale industrial sensor ingestion and feature engineering

### PSA Corporation - Senior Mechanical Engineer  (Jul 2013-Jul 2018)

- Automated component life tracking and failure pattern analysis; translated mechanical domain knowledge into ML feature engineering

## EDUCATION

- **Master of Technology (Knowledge Engineering)**, National University of Singapore - ISS (2018-2020)
- **Bachelor of Engineering (Mechatronic Engineering)**, Nanyang Technological University (2009-2013)
