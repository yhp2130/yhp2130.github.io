# Pro-Toolbox: Virtual Process Consultant

**Role:** Lead AI Engineer  
**Duration:** <2 weeks to production  
**Domain:** Organisational Process Knowledge  Process Improvement  

---

## Summary

| Dimension | Detail |
|-----------|--------|
| Delivery | <2 weeks concept to production (GitHub Copilot SpecKit) |
| Knowledge sources | Confluence (process docs)  ARIS (BPMN process metadata) |
| Execution paths | Direct  Fast path  ReAct deep thinking |
| Phase sub-agents | 5: Initiation/Planning, Execution/Stabilization, Project Management, Change/Communication, Monitoring/Control |
| Frontend | React.js (TypeScript)  SSE streaming  per-message feedback loop |
| CI/CD | GitLab CI/CD  ArgoCD  Helm  OpenShift  Artifactory |

---

## Overview

A virtual process consultant that helps process owners and users understand organisational processes and receive actionable improvement guidance. Given a natural language query, the system retrieves relevant knowledge from Confluence and ARIS (the organisation's BPMN process modelling software), then synthesises a cited consultant-quality answer with recommendations and rationale.

---

## Phase 1  Design: Data Sources and Knowledge Architecture

**Sources:** Confluence (process documentation)  ARIS (BPMN process metadata)

Two complementary knowledge sources were identified:

| Source | Content | Access |
|--------|---------|--------|
| **ARIS** | Process names, IP codes, ownership, scope, mission, hierarchy, goals | REST API  Excel workbook  CSV; auto-fetched on startup, refreshed hourly |
| **Confluence** | Process documentation, SOPs, improvement guidelines, intranet articles | Internal semantic search platform |

ARIS metadata is auto-provisioned at startup: the system downloads the Excel workbook from the ARIS REST API, extracts the relevant sheet with openpyxl, and writes an atomic CSV. A background task refreshes this every hour. On failure, the existing CSV is preserved  the system continues with stale data rather than going down.

**Design decision:** ARIS metadata kept as local CSV rather than live API calls  decouples the query path from ARIS REST availability and enables fuzzy + exact scoring without network latency per query.

---

## Phase 2  Internal Platform Integration: Semantic Search as Reusable Component

**Principle:** Reuse over reinvent  constraints: rate limits, source update frequency, on-prem

Rather than building a custom embedding and vector store pipeline, the system integrates with an **internal semantic search platform** already maintained by another team. This platform handles document indexing, embedding, and retrieval across Confluence and intranet sources, with built-in rate limiting, source update frequency management, and access control.

A thin `search_engine` adapter calls the platform's REST API (`POST /search`) with per-agent project scoping and mode configuration (`llm` / `vector` / `context`). Each agent can override the search mode and target project IDs independently via its YAML config file.

**Why reuse:** The platform already manages update frequency, rate limiting, and TLS cert handling on-prem  building equivalent infrastructure would have doubled scope with no user-visible benefit.

---

## Phase 3  Agent Architecture: Planner  Phase Agents  Checker

**Library:** OpenAI native SDK (AsyncOpenAI)  on-prem endpoint  per-agent YAML config

Built with OpenAI's native Python SDK (`openai.AsyncOpenAI`) pointed at an on-premise OpenAI-compatible endpoint  no additional orchestration layer. Each agent is independently configured via a YAML file (model, temperature, max_tokens, tool permissions, search engine overrides).

```
User Query
    
    
Planner Agent LLM routing ARIS Agent         ─ Checker Agent 
                              IFX Process Mgmt    Checker Agent 
                                                                       
                             phase routing                    Planner consolidate
                                                                       
                                      Final Cited Response
                         Initiation/Planning 
                         Execution/Stabilize 
                         Project Management  
                         Change/Communicate  
                         Monitoring/Control  
                        
```

- **Planner Agent:** LLM routing  classifies query as `aris`, `ifx_process_mgmt`, or `direct`; expands process abbreviations; reconstructs vague queries into precise search phrases
- **ARIS Agent:** fuzzy + exact search on the local CSV  phrase matching, acronym detection, token coverage scoring, hierarchy-depth weighting; top-10 results
- **IFX Process Mgmt Agent:** second-level routing into 5 process phase sub-agents or a general search path
- **Checker Agent:** mandatory gate after every sub-agent run  cross-references results, filters citations to only those used in the final answer; blocks responses with no citations

**Citation policy:** Every response must surface at least one traceable citation. Responses without citations are never shown to the user.

---

## Phase 4  Execution Paths: Fast Path and ReAct Deep Thinking

**Paths:** direct  fast (sequential)  ReAct loop (iterative, user-triggered)

| Path | When | Mechanism |
|------|------|-----------|
| **Direct** | Conversational / utility task (greeting, translate, summarise) | Planner answers directly; no sub-agents; no citation required |
| **Fast path** | Standard knowledge queries | Sequential: ARIS Agent  IFX Agent  Checker  Planner consolidation |
| **ReAct loop** | User explicitly requests deep thinking | Iterative: plan ExecutionPlan (SubGoals with dependency graph)  execute tools in parallel (asyncio)  observe  sufficiency check  re-plan if needed |

The **ReAct loop** uses an `ExecutionPlan`  a structured decomposition into `SubGoal` objects, each with a tool, parameters, and `depends_on` dependency list. Parallel sub-goals are executed concurrently with asyncio. A `SufficiencyEvaluator` decides whether to finalise or re-plan after each iteration, up to a configurable `REACT_MAX_ITERATIONS` limit.

**Fast path default:** The ReAct loop adds latency and is only activated when the user explicitly enables deep thinking mode. Standard queries always take the fast path to stay under the 25-second agent timeout.

---

## Phase 5  Frontend: React.js with Feedback Loop

**Framework:** React.js (TypeScript)  SSE streaming  per-message rating + comment

Built a React.js + TypeScript frontend with an SSE streaming hook (`useChat`) for real-time response rendering as the agent pipeline produces output. Components: `ChatWindow`, `MessageBubble`, `CitationPanel`, `ThinkingPanel` (shows live agent steps).

**Feedback loop:** Users submit a rating (thumbs up / thumbs down) and optional comment per assistant response via `PUT /feedback/{message_id}`. Feedback is stored inline in the conversation's SQLite record  only the conversation owner can update their feedback.

**Feedback use:** QnA pairs with ratings are stored persistently for future evaluation, fine-tuning, and identifying which query types the system handles poorly.

---

## Phase 6  CI/CD: Build  Dev  Staging  Prod

**Stack:** GitLab CI/CD  ArgoCD  Helm  OpenShift  Artifactory

Branch-to-environment mapping:

| Branch / Tag | Environment | Trigger |
|-------------|-------------|---------|
| `feature/*` MR |  | Test + lint only |
| `develop` | Dev | Test + lint + build + deploy (ArgoCD auto-sync) |
| `main` | Staging | Test + lint + build + deploy (ArgoCD auto-sync) |
| `v*` tag | Prod | Test + lint + build + deploy (manual gate) |

Pipeline stages:
- **build-ci**  pre-built CI image with Python deps; only rebuilds on `requirements.txt` change
- **test**  pytest (backend) + frontend tests
- **lint**  ruff + ESLint
- **build**  OpenShift BuildConfig  Artifactory Docker registry
- **update-manifests**  commit new image tags to Helm values with `[skip ci]`
- **rotate-token**  monthly scheduled service account token renewal

All Python and npm packages are proxied through Artifactory (direct internet access to PyPI / npmjs.org blocked on-prem).

**Outcome:** Full CI/CD pipeline shipped within the <2-week delivery window alongside the application itself, using GitHub Copilot SpecKit spec-driven development.

---

## Retrospective Learning

- Per-agent YAML config files (model, temperature, tool permissions, search overrides) proved essential for tuning agent behaviour without code changes  this pattern scales well as more agents are added
- The sufficiency evaluator in the ReAct loop avoids wasted iterations but requires careful threshold tuning: too aggressive and it exits early on sparse results; too lenient and latency grows
- Routing the ARIS Agent separately from the semantic search agent keeps ARIS metadata fresh without coupling to the search platform's update cycle

---

## Stack

| Category | Tools |
|----------|-------|
| LLM SDK | OpenAI (AsyncOpenAI, on-prem endpoint) |
| API | FastAPI |
| Frontend | React.js (TypeScript)  Vite |
| Knowledge retrieval | Internal Semantic Search API  ARIS REST API |
| Data | SQLite (conversations + feedback) |
| Infrastructure | Docker  OpenShift |
| CI/CD | GitLab CI/CD  ArgoCD  Helm  Artifactory |
| Dev tooling | GitHub Copilot SpecKit |
