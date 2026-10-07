# Backend Architecture for Insurance AI Assistant

## Overview

This document describes the backend architecture for the enterprise insurance AI assistant. The backend must support:

- secure API interfaces
- retrieval and orchestration logic
- customer and policy context access
- vector-search integration
- document ingestion and metadata workflows
- auditability and telemetry
- operational resilience and scalability

The preferred implementation stack is Python with FastAPI, Pydantic, and AsyncIO, with enterprise-grade deployment on Azure and Kubernetes.

---

## 1. Backend Objectives

The backend should:

- expose a secure REST API for customers, advisors, and internal operations
- orchestrate retrieval, customer data gathering, and model reasoning
- enforce role-based and policy-based access checks before returning answers
- support both synchronous and asynchronous workflows
- keep the AI logic modular and testable
- log all decisions, retrievals, safety events, and answer traces

---

## 2. Recommended Backend Stack

### Core technologies

- Python
- FastAPI
- Pydantic
- AsyncIO
- Uvicorn / ASGI runtime
- Structured logging
- OpenTelemetry
- Redis for caches and short-term sessions
- MongoDB for metadata and audit information
- Qdrant for vector retrieval

### Why this stack is appropriate

- Python is a standard for AI, ML, and enterprise backend engineering
- FastAPI provides high performance and strong type validation
- Pydantic guarantees validated request and response schemas
- AsyncIO helps when calling retrieval systems, database services, and LLM wrappers concurrently
- The backend remains easy to test, secure, and operationalize in production

---

## 3. Backend Service Breakdown

```mermaid
flowchart TD
    U[Client Apps / Portals / Advisors] --> G[API Gateway]
    G --> API[FastAPI App]
    API --> Auth[Auth & Access Control]
    API --> Session[Session Service]
    API --> Query[Query Router]
    Query --> Orchestrator[Agent Orchestrator]

    Orchestrator --> Context[Customer Context Service]
    Context --> CRM[CRM / Policy Systems / Claims Systems]

    Orchestrator --> Retrieval[Retrieval Service]
    Retrieval --> Qdrant[(Qdrant)]
    Retrieval --> Redis[(Redis)]

    Orchestrator --> Rules[Policy Rule Service]
    Rules --> PolicyDB[(Policy Metadata / Rule Store)]

    Orchestrator --> LLM[LLM Gateway / Model Service]
    LLM --> Guard[Guardrails]
    Guard --> Valid[Response Validator]
    Valid --> Audit[Audit & Trace Store]
    Valid --> Reply[Final Response]

    Admin[Admin UI / Operations] --> Ingest[Document Ingestion Service]
    Ingest --> Blob[(Blob Storage)]
    Ingest --> DocIntel[Document Intelligence]
    Ingest --> Mongo[(MongoDB)]
    Ingest --> Qdrant
```

---

## 4. Service Responsibilities

### 4.1 API gateway and entrypoint
Responsibilities:

- provide HTTP API for all user requests
- route by workflow type
- enforce rate limits and request validation
- inject tenant / role context
- expose only the required interfaces externally

### 4.2 Authentication and authorization service
Responsibilities:

- verify identity via OAuth2/OIDC or enterprise IAM
- determine user role and access rights
- verify policy or claim access restrictions
- prevent cross-user data leakage

### 4.3 Session and conversation service
Responsibilities:

- store bounded conversation state and links to source turns/evidence
- optionally persist durable conversation history in an approved store under retention and deletion policies
- track user interaction history
- maintain short-lived memory for multi-turn conversations

Redis is an optional ephemeral session/cache layer with TTL and scoped keys, not the authoritative customer store or sole durable conversation record. MongoDB may hold conversation metadata/history only in separately governed collections if privacy, retention, deletion, and access requirements are approved. Customer profile, policy, claim, and eligibility facts must be fetched from enterprise systems of record on each authorized workflow; never derive them from LLM memory or old assistant messages.

### 4.4 Query router
Responsibilities:

- classify user intent
- distinguish FAQ, policy lookup, claim status, coverage inquiry, recommendation, or escalation
- choose whether to use retrieval, rules, or a human workflow
- route deterministic/simple requests without unnecessary LLM calls; invoke conversation-aware rewriting for ambiguous follow-ups and capped multi-query only for complex decomposable questions

### 4.5 Agent orchestrator
Responsibilities:

- coordinate multi-step reasoning flows
- call retrieval, customer context, policy rules, and LLM services
- maintain deterministic workflow state for auditability

### 4.6 Customer context service
Responsibilities:

- fetch policy details, customer profile, claims state, coverage information
- combine business data with retrieved document evidence
- ensure any customer facts are permission-checked before use

### 4.7 Retrieval service
Responsibilities:

- build search queries based on user intent and customer context
- apply metadata filters (product, document type, region, policy version)
- consult Qdrant for dense vector retrieval and, where deployed, a synchronized lexical/BM25 index
- fuse and deduplicate candidates under identical access/version filters
- optionally apply conservative MMR for duplicate-heavy candidate sets
- call rerankers and then assemble evidence for the model
- retain source IDs/offsets if optional extractive context compression is applied

### 4.8 Policy rule service
Responsibilities:

- evaluate claim eligibility and business rules
- apply policy logic for coverage, exclusions, waiting periods, and claim procedure checks
- be strict about not inferring unsupported coverage scenarios

### 4.9 LLM gateway
Responsibilities:

- manage model API calls
- handle prompt templates and structured output generation
- track model version and cost usage
- enforce guardrails and fallback logic
- construct separate typed prompt blocks for system/security instructions, user request, customer facts, rule results, policy evidence, conversation context, and output contract
- use a bounded token budget; history is continuity context only and cannot override authoritative sources

### 4.10 Guardrail and validation service
Responsibilities:

- detect prompt injection and unsafe input
- ensure retrieved evidence is used properly
- validate answer grounding and citation completeness
- detect poor or unsupported output and route to review if needed

### 4.11 Audit and trace service
Responsibilities:

- record user request details
- record evidence IDs, source versions, rule versions, and decision provenance; retain raw prompt/evidence text only where an approved audit requirement explicitly requires it
- save model outputs and citations
- persist approval or escalation decisions

Ordinary telemetry uses opaque references and redaction. Access to any separately retained content-bearing audit record must be least-privilege, purpose-limited, and governed by retention/deletion policy.

---

## 5. Backend Request Flow

```mermaid
sequenceDiagram
    participant User
    participant API as FastAPI API
    participant Auth as Auth Service
    participant Router as Query Router
    participant Orchestrator as Agent Orchestrator
    participant Context as Customer Context Service
    participant Retrieval as Retrieval Service
    participant Rules as Policy Rule Service
    participant LLM as LLM Gateway
    participant Guard as Guardrail Layer

    User->>API: Send question
    API->>Auth: Validate identity & permission
    Auth-->>API: Authorized context
    API->>Router: Route request
    Router->>Orchestrator: Build workflow
    alt Self-contained policy FAQ
        Orchestrator->>Retrieval: Search authorized current policy docs
        Retrieval-->>Orchestrator: Relevant chunks / no-evidence status
    else Customer coverage or eligibility request
        par Authorized structured customer/policy lookup
            Orchestrator->>Context: Fetch required system-of-record facts
            Context-->>Orchestrator: Facts with source and freshness
        and Policy evidence retrieval
            Orchestrator->>Retrieval: Search using validated scope and filters
            Retrieval-->>Orchestrator: Versioned evidence / no-evidence status
        end
        opt Deterministic rule evaluation is required
            Orchestrator->>Rules: Evaluate with validated inputs
            Rules-->>Orchestrator: Versioned outcome and reason codes
        end
    else Claim status request
        Orchestrator->>Context: Read authorized claims system
        Context-->>Orchestrator: Claim facts with source and freshness
    end
    Orchestrator->>Guard: Validate required evidence, provenance and response contract
    Guard-->>Orchestrator: Pass, clarify, abstain or escalate
    alt Safe to generate
        Orchestrator->>LLM: Send combined context
        LLM-->>Guard: Draft answer
        Guard->>Guard: Safety + schema + citation validation
        Guard-->>API: Validated answer with citations
    else Missing evidence or failed validation
        Guard-->>API: Clarification, abstention or human handoff
    end
    API-->>User: Response
```

---

## 6. Data Contracts and Domain Models

### 6.1 Request models
Use Pydantic schemas for:

- conversation request
- policy query request
- claims lookup request
- recommendation request with explicit subject authorization, purpose/consent and idempotency header
- document upload request
- admin workflow action request

### 6.2 Response models
Use structured response models for:

- final answer payload
- citations
- guardrail status
- retrieval metadata
- workflow trace summary
- recommendation response with outcome, source/rule/catalog versions, timestamps, disclosures, citations and stable error envelope
- distinct policy facts, customer facts with source/freshness, deterministic business-rule results with reason codes, and model-generated explanation/recommendation

The API must not flatten these categories into an unattributed answer string. Policy facts cite approved document evidence; customer facts identify the authorized system of record and verification time; eligibility outcomes identify the governed rule/version. Recommendations/explanations are explicitly non-binding unless a separately approved workflow defines otherwise. Server-side validation resolves citation IDs and rejects unsupported material claims.

The recommendation endpoint is a bounded synchronous operation in the target contract. Use stable outcomes for completed/no candidates/insufficient data/review required and a common error envelope (`code`, `message`, `trace_id`, `retryable`). Map authentication/authorization, invalid input, idempotency conflict, rate limit, dependency unavailable and deadline expiry distinctly; do not return partial candidates if a required source or rule service failed. Keep a server-side release control disabled by default until its authoritative integrations and API contract are implemented, tested and approved. While disabled, authenticate the caller and verify endpoint-level permission without looking up the customer, then return a generic `503 recommendation_unavailable` with a trace ID. When enabled, verify subject authorization and consent/purpose before reading customer data. Do not reveal customer existence or fall back to conversational generation.

### Example response skeleton

```python
from pydantic import BaseModel
from typing import List, Optional

class Citation(BaseModel):
    doc_id: str
    chunk_id: str
    section_title: Optional[str]
    page_number: Optional[int]
    version: str

class AnswerResponse(BaseModel):
    answer: str
    citations: List[Citation]
    confidence: float
    guardrail_status: str
    escalation_required: bool = False
```

---

## 7. Asynchronous Processing Patterns

Some backend workflows are not user-interactive and should run asynchronously.

### Example workloads

- document ingestion
- embedding generation
- chunk indexing
- background audits
- batch data refreshes
- reindexing after policy updates

### Pattern recommendation

- Use async job queues or event-driven processing for these tasks
- Keep synchronous user-facing workflows short and responsive
- Do not block the user API while large ingestion or indexing tasks execute
- The selected Kafka ingestion broker and worker execution are treated as at-least-once; do not claim exactly-once side effects

### Idempotency and replay boundaries

- Derive a stable ingestion job key from tenant/source document ID/source checksum/version and pipeline generation; persist stage status before acknowledging completion.
- Make extraction artifacts, chunk IDs, embedding writes, and metadata transitions idempotent for that key. Use staging generations and only publish after cross-store count/ID/version reconciliation.
- On duplicate broker delivery, acknowledge a completed stage as a no-op; route conflicting checksums or non-retryable poison events to a dead-letter queue for review.
- LangGraph node replay is not equivalent to idempotent LLM output. Keep model calls bounded by node/request deadlines and retry only transient provider errors. For side-effecting tools, pass an idempotency key and reconcile operation status before retry; never assume a model call has exactly-once semantics.
- Keep retries capped with jitter, per-dependency timeouts, an overall request deadline, and circuit breakers for sustained failures. Cancel sibling parallel work when its result can no longer affect a response.
- Set an API-level deadline and pass remaining time to every graph node/dependency; nested retries must not exceed it. Use async I/O for network-bound dependencies, separate bounded pools/semaphores per dependency, and fail-fast overload responses when admission limits are reached.
- Retry only transient/idempotent operations within the remaining deadline; do not retry validation, authorization, stale-version, or deterministic rule failures. Honor provider retry-after guidance for throttling and use a shared per-request retry budget.

The graph state should be a minimal, typed, request-scoped object with authorized scope, correlation/idempotency key, route, validated evidence references, rule result, deadline, and terminal outcome. Persistent checkpoint storage and retention are not specified; disable durable checkpoints until their security, privacy, and recovery requirements are approved.

Interactive route deadlines are end-to-end budgets, not independent timeouts that can accumulate. Allocate a budget to authorization, optional history/customer reads, retrieval/reranking, generation, guardrails/citation validation, and response serialization for each route. Product owners must set the numeric objectives; the backend enforces remaining-time propagation and cancels work on deadline. Ingestion, offline Ragas, trace export and nonessential summary refresh remain asynchronous and outside the interactive request budget.

---

## 8. Storage and Integration Layer

### Persistent stores

- MongoDB for metadata and audit records
- Qdrant for retrieval vectors
- Redis for sessions and caches
- Blob storage for raw source documents and extracted outputs

### Integration patterns

- use repository/service abstractions for each storage system
- isolate AI service calls behind ports/adapters
- treat external systems as dependencies to help with testing and resilience

---

## 9. Authentication and Authorization Model

### Identity model

- enterprise SSO / OAuth2 / OIDC
- service-to-service authentication via managed identity or JWTs

### Authorization model

- RBAC for roles such as adviser, customer, admin, operations
- ABAC for customer-specific contextual access
- document access filtered at retrieval time

### Principle

Never assume a user can access all policy documents just because they are logged in.

---

## 10. Security in the Backend

### Required protections

- input validation at API boundaries
- sanitization for user-controlled content
- prompt-injection defense in orchestration logic
- masking of PII before sending data to LLMs when necessary
- secure storage and rotation of secrets
- least-privilege service permissions
- strict logging without leaking sensitive data

### High-risk backend flows

- customer-specific policy retrieval
- claims eligibility decisions
- denials or exceptions handling
- internal document ingestion and metadata updates

These should usually have additional validation and audit checks.

### PII controls: Presidio and NeMo Guardrails

These tools address different risks and are not substitutes:

| Control | Primary role | Does not replace |
|---|---|---|
| Microsoft Presidio (or an approved equivalent DLP/PII service) | Detect and optionally anonymize identifiable text before an external model/trace boundary | Identity/authorization, business-purpose minimization, prompt-injection defense, policy grounding, or citation validation |
| NeMo Guardrails | Dialogue/input/output policy checks and configured safety/behavior rails | Reliable PII discovery, access control, source-of-truth verification, deterministic rules, or schema/citation checks |

For insurance data sent to an external model or trace service, first minimize fields and avoid sending data not needed for the task. When classified PII may cross that boundary, use Microsoft Presidio as the reference detection/masking implementation or an approved enterprise DLP equivalent that passes the same tests. Calibrate recognizers for policy identifiers and regional formats; measure false negatives/positives and verify masked prompts remain useful. Do not restore masked identifiers inside model context. If a response requires an identifier, render it only through an authorized application path after response validation. Redact traces independently; do not log raw text as a workaround.

NeMo can complement that PII layer for prompt injection and dialogue policy. Neither tool is a security boundary by itself: enforce authZ before retrieval and validate structured output, provenance, citations, and customer data in deterministic application code.

```mermaid
flowchart LR
    USER[Untrusted request] --> SCHEMA[API schema, size and rate validation]
    SCHEMA --> AUTH[Authentication and authorization]
    AUTH --> MIN[Minimize authorized context]
    MIN --> PII[PII detect / mask when policy requires<br/>Presidio or approved DLP]
    PII --> INJECT[Prompt-injection / policy screening<br/>NeMo and application controls]
    INJECT --> RETRIEVE[Filtered retrieval and system-of-record calls]
    RETRIEVE --> DOCSAFE[Label retrieved content as untrusted data<br/>screen embedded instructions]
    DOCSAFE --> PROMPT[Typed, provenance-labelled prompt]
    PROMPT --> LLM[LLM]
    LLM --> NEMO[Output policy checks]
    NEMO --> VALIDATE[Schema, evidence, citation and PII validation]
    VALIDATE -->|Pass| RESP[Return validated response]
    VALIDATE -->|Fail| SAFE[Reject, abstain or human review]
```

Before release, document approved PII classes, model/trace destinations and terms, masking behavior, audit/trace retention and deletion, and detector outage handling. Verify the selected Presidio/DLP configuration with regional insurance identifiers, false-negative testing, output checks and trace-redaction tests. If masking is required by policy and the control is unavailable, do not send the affected data externally; fail closed or route to an approved non-external workflow. NeMo configuration does not satisfy this PII gate.

---

## 11. Observability and Telemetry

The backend must create rich operational telemetry.

### Required telemetry

- request latency by endpoint
- error rate and exception traces
- retrieval latency and status
- model/provider latency, per-attempt token usage, retry count, and cost by route and stage
- business workflow state transitions
- safety and guardrail triggers
- audit events for policy or claims answers
- in-flight requests, dependency pool wait, event-loop lag, admission rejects and P50/P95/P99 by route
- queue age/consumer lag/DLQ, Qdrant/BGE saturation, cache bypass/invalidation, citation rejects and no-hit/abstention/escalation outcomes

### Implementation

- OpenTelemetry instrumentation across services
- structured logs with correlation IDs
- trace IDs linking user request → orchestration → retrieval → answer
- centralized dashboards and alerting in Azure Monitor / Application Insights
- stage-level metrics for query rewrite, routing, dense/lexical search, fusion, optional MMR/compression, BGE, generation, guardrails, and citation validation
- record every model attempt in the request call ledger with model/config version, stage, token count, retry reason, latency and outcome; attribute embedding/OCR and background evaluation spend separately from interactive generation
- use low-cardinality metric labels (route, model, stage, outcome); never label metrics with customer IDs or raw content; use opaque identifiers in access-controlled traces
- configure alert thresholds from approved SLOs and measured baselines, with named owners/runbooks; monitor calls/route and token/cost budget exhaustion without logging prompt contents
- Ragas runs offline or asynchronously on privacy-reviewed samples; it never blocks a synchronous user response
- LangSmith is optional for redacted LLM experiments/traces and does not replace OTel/Azure production telemetry
- avoid raw conversation, PII, or full retrieved passages in ordinary logs; use opaque references, redaction, and governed trace access

---

## 12. Failure Handling and Resilience

### Backend failure scenarios

- model API unavailable
- retrieval database failure
- customer policy API timeout
- Redis cache outage
- document ingestion pipeline delay
- unexpected data schema mismatch

### Recommended strategies

- per-dependency deadlines and an overall request deadline; bounded retries with exponential backoff and jitter only for transient/idempotent operations
- degraded mode for non-critical workflows
- fallback safe responses when retrieval quality is poor
- circuit breakers for dependencies
- idempotent processing for ingestion, metadata, and side-effecting tools; at-least-once queue redelivery is expected
- cancel parallel work when the deadline expires; do not retry authorization denials, malformed requests, or deterministic validation failures
- permit an approved alternate LLM only after separate quality, privacy, and operational qualification; otherwise abstain or hand off

### Important principle

Failure should degrade safely, not silently provide incorrect customer-specific claims.

---

## 13. Scalability Model

### Horizontal scaling strategies

- replicate API pods behind the gateway
- independently scale retrieval and orchestration layers
- use queue-based ingestion processing
- cache public/role-scoped FAQ evidence only when measured; personalized facts and eligibility are not cache authorities

### Adaptive scaling triggers

- number of concurrent requests
- LLM latency and queue backlog
- retrieval latency and Qdrant load
- storage workload for ingestion
- worker queue wait, dependency connection-pool wait, event-loop lag, and provider throttling/saturation

Use bounded async connection pools for CRM/policy/claims, MongoDB, Redis, Qdrant, embedding and LLM clients. Keep per-dependency concurrency limits and request deadlines so a slowdown cannot exhaust all API workers. Apply bulkheads between interactive workflows and background ingestion/evaluation. Autoscaling adjusts capacity but does not replace overload control: use gateway admission/rate limits and bounded queues, and return a retryable overload response when safe capacity is exhausted.

### Cache safety

Redis is optional acceleration, not a source of truth. Prefer immutable, approved public/role-scoped retrieval evidence. Partition entries by tenant and authorization scope and include every result-affecting dimension (jurisdiction, effective date, document/index generation, retrieval configuration and prompt/schema version as applicable). Reauthorize before use and invalidate on source, ACL, policy-version or configuration changes. Never cache eligibility/rule decisions or authoritative customer facts. Personalized response caching is disabled by default; enable only with a documented benefit, scoped key design, short approved TTL, invalidation tests and a verified cache-bypass path. If safe scope or freshness cannot be checked, bypass the cache.

---

## 14. Testing Strategy for Backend

The backend should be covered at multiple levels.

### Unit tests

- auth logic
- validation and request parsing
- orchestration path selection
- metadata filtering logic
- guardrail checks

### Integration tests

- retrieval integration with Qdrant
- metadata lookup integration with MongoDB
- customer context API calls
- policy rule engine behavior

### End-to-end tests

- FAQ response flow
- claims-status flow
- claim-eligibility flow
- product recommendation flow
- escalated human-review flow

---

## 15. Backend Architectural Decision Summary

The recommended backend architecture is:

- FastAPI-based REST API with validated schemas
- modular service decomposition for retrieval, policy logic, agent orchestration, and validation
- strong identity and access enforcement
- asynchronous processing for ingestion and other heavy jobs
- Azure-native storage and telemetry architecture
- structured auditability for regulated workflows

This architecture is suitable for a production insurance AI assistant because it supports both user-facing intelligence and operational reliability while keeping the system secure, maintainable, and auditable.

---

## 16. Full Backend Architecture Diagram

```mermaid
flowchart TD
    Client[Customer / Advisor / Admin] --> Gateway[API Gateway / WAF]
    Gateway --> API[FastAPI App]
    API --> Auth[Auth and Authorization]
    API --> Session[Session Service]
    API --> Router[Query Router]
    Router --> Orchestrator[Agent Orchestrator]

    Orchestrator --> Ctx[Customer Context Service]
    Ctx --> CRM[CRM / Claims / Policy Systems]

    Orchestrator --> Retrieval[Retrieval Service]
    Retrieval --> Qdrant[(Qdrant)]
    Retrieval --> Redis[(Redis)]

    Orchestrator --> Rules[Policy Rule Engine]
    Rules --> PolicyStore[(Rule / Metadata Store)]

    Orchestrator --> LLM[LLM Gateway]
    LLM --> Guard[Guardrails]
    Guard --> Valid[Validation Layer]
    Valid --> Audit[(Audit / Trace Store)]
    Valid --> Resp[Final Response]

    Admin --> Ingest[Document Ingestion Service]
    Ingest --> Blob[(Azure Blob Storage)]
    Ingest --> DocIntel[Azure AI Document Intelligence]
    Ingest --> Mongo[(MongoDB)]
    Ingest --> Qdrant

    Observability[Azure Monitor / App Insights / OTel] --> API
    Observability --> Orchestrator
    Observability --> Retrieval
    Observability --> Ingest
    KeyVault[Azure Key Vault] --> API
    KeyVault --> Orchestrator
    KeyVault --> Ingest
```

This diagram reflects the full backend engine behind the insurance AI assistant: secure user entry, policy-aware retrieval, orchestration, validation, and operational monitoring.

## Backend Dependency Diagram

```mermaid
flowchart TD
    Client[Web / Mobile / Advisor App] --> Gateway[API Gateway]
    Gateway --> API[FastAPI App]
    API --> Auth[Auth & Access Control]
    API --> Session[Session Service]
    API --> Query[Query Router]

    Query --> Orchestrator[Agent Orchestrator]
    Orchestrator --> Context[Customer Context Service]
    Context --> CRM[CRM / Policy / Claims Systems]

    Orchestrator --> Retrieval[Retrieval Service]
    Retrieval --> Qdrant[(Qdrant)]
    Retrieval --> Redis[(Redis)]

    Orchestrator --> Rule[Policy Rule Engine]
    Rule --> PolicyStore[(Policy Rules / Metadata)]

    Orchestrator --> Model[LLM Gateway]
    Model --> Guard[Guardrails]
    Guard --> Validate[Validation Layer]
    Validate --> Audit[(Audit / Trace Store)]
    Validate --> Response[Final Response]

    Admin[Operations Admin] --> Ingest[Ingestion Service]
    Ingest --> Blob[(Blob Storage)]
    Ingest --> Intel[Document Intelligence]
    Intel --> Mongo[(MongoDB Metadata)]
    Ingest --> Qdrant

    Telemetry[OpenTelemetry / Azure Monitor] --> API
    Telemetry --> Orchestrator
    Telemetry --> Retrieval
    Telemetry --> Ingest
```

This diagram highlights the backend service interactions, dependency chain, and the data flow that supports secure, policy-aware responses and asynchronous ingestion work.

---

## 17. Edge Case & Failure Handling

The backend must treat authorization, customer context, policy evidence, and response validation as correctness gates. Timeouts and dependency errors must be visible and must not be converted into successful-looking but incomplete answers.

### Scenario: Invalid, expired, or incomplete identity/authorization context

- **Why it can happen:** Expired or malformed tokens, identity-provider outage, missing role/tenant claims, or stale policy/customer relationship data.
- **How the architecture detects it:** Authentication middleware validates issuer, signature, audience, and expiry; authorization checks required claims and customer scope before accessing data.
- **How the system handles/recover from it:** Reject invalid credentials; refresh only through the approved identity flow; retry identity-provider errors only when transient and within request deadlines. Do not retry a policy denial as a service error.
- **Fallback behavior:** Return a generic unauthorized/forbidden response without revealing whether the requested policy or claim exists.
- **Impact on the user/system:** Request cannot proceed until credentials or access are corrected; protected data remains unavailable.
- **Monitoring/alerting required:** Track authentication failures separately from access denials; alert on identity-provider latency/outage and anomalous tenant/role access patterns.

### Scenario: Customer context or policy/claims system times out or returns stale data

- **Why it can happen:** Upstream service degradation, network timeout, rate limiting, stale replica, or incompatible response schema.
- **How the architecture detects it:** Per-dependency deadlines, response schema and freshness checks, circuit-breaker state, and source timestamps.
- **How the system handles/recover from it:** Use bounded retries with backoff for transient failures; validate data contracts; open the circuit during sustained failure and emit a correlated trace. Do not use stale customer data unless explicitly permitted by the business freshness policy.
- **Fallback behavior:** Continue only with a public/general policy answer when retrieval authorization and response wording make that safe; otherwise request retry or route to an advisor. Never present a general answer as customer-specific.
- **Impact on the user/system:** Personalized eligibility or claims flows may be unavailable; non-customer-specific FAQs may continue.
- **Monitoring/alerting required:** Alert on upstream error rate, timeout rate, freshness lag, circuit-breaker opens, and schema-validation failures.

### Scenario: Qdrant or retrieval dependency is unavailable

- **Why it can happen:** Vector service outage, network partition, index overload, deployment issue, or shard/replica failure.
- **How the architecture detects it:** Connection/query timeouts, readiness checks, retrieval latency and error metrics, and missing-index/version validation.
- **How the system handles/recover from it:** Apply bounded retry and circuit-breaker behavior; restore connectivity or fail over to a validated replica/index when configured; preserve correlation IDs and retrieval diagnostics.
- **Fallback behavior:** Do not generate policy claims without evidence. Return an explicit temporary-unavailable/no-evidence response or initiate human review.
- **Impact on the user/system:** RAG questions fail safely; unrelated session and administrative features may remain available.
- **Monitoring/alerting required:** Alert on Qdrant health, query errors/latency, replica status, index freshness, and no-evidence rate changes.

### Scenario: LLM provider throttles, times out, or returns malformed output

- **Why it can happen:** Provider outage, rate/token limits, transient network errors, model change, or output exceeding the expected schema.
- **How the architecture detects it:** Request deadlines, provider status/error codes, token-budget checks, structured-output parsing, and schema validation.
- **How the system handles/recover from it:** Retry only transient errors with bounded exponential backoff and jitter; honor rate-limit retry guidance; use an approved lower-cost/backup model only if it has passed governance and quality evaluations; capture model/configuration identifiers.
- **Fallback behavior:** Return a safe retry message or advisor handoff; never return partial or malformed generated text as a completed answer.
- **Impact on the user/system:** Increased latency, temporary loss of generative responses, or escalation; API remains responsive within its deadline budget.
- **Monitoring/alerting required:** Alert on provider error and throttle rates, latency, token-budget exhaustion, invalid structured outputs, fallback model usage, and cost anomalies.

### Scenario: Redis session/cache outage or cache contains stale data

- **Why it can happen:** Redis failover, network interruption, memory pressure, eviction, TTL/configuration error, or cache key omits authorization/version scope.
- **How the architecture detects it:** Cache connection errors, hit/miss and eviction metrics, TTL checks, and validation of cached tenant/authorization scope and document/index-version metadata.
- **How the system handles/recover from it:** Treat cache as non-authoritative; bypass it and read from the source of truth when safe. Invalidate entries after ACL, document/version, index-generation or configuration changes; use scoped keys and approved TTLs. Do not cache eligibility or customer facts as authority.
- **Fallback behavior:** Reconstruct ephemeral session context or require the user to restate lost context; if authorization scope or evidence freshness cannot be re-established, bypass the cache and do not serve customer-specific data.
- **Impact on the user/system:** Higher latency or loss of conversational continuity; no correctness or access decision relies solely on cache.
- **Monitoring/alerting required:** Monitor Redis availability, memory/evictions, cache hit rate, invalidation lag, and detected scope/version mismatches.

### Scenario: Request overload, timeout cascade, or duplicate async job

- **Why it can happen:** Traffic spike, retry storm, slow dependencies, unbounded request concurrency, at-least-once queue delivery, or worker restart after partial completion.
- **How the architecture detects it:** Gateway rate limits, request queue depth, latency/error SLOs, dependency saturation, worker heartbeat, duplicate idempotency key, and queue age.
- **How the system handles/recover from it:** Apply admission control, per-dependency deadlines, bounded concurrency and backpressure; use idempotency keys for jobs and writes; retry transient failures with capped backoff; route poison messages to a dead-letter queue.
- **Fallback behavior:** Return a clear throttled/retry-later response for interactive traffic; preserve document jobs durably for replay instead of holding requests open.
- **Impact on the user/system:** Some requests are delayed or rejected under overload, while bounded work prevents cascading failure and duplicate indexing.
- **Monitoring/alerting required:** Alert on saturation, queue depth/age, timeout cascades, retries, throttling, duplicate jobs, dead-letter volume, and worker restarts.

### Scenario: Query rewriting or multi-query planning changes scope or exhausts the request budget

- **Why it can happen:** A follow-up has multiple plausible antecedents, rewrite/model timeout, query expansion produces too many searches, or concurrent subqueries overload retrieval services.
- **How the architecture detects it:** Validate rewrite confidence and scope against the original query; enforce subquery, token, deadline, and concurrency budgets; monitor per-stage latency and retrieval-call count.
- **How the system handles/recover from it:** Use direct retrieval for self-contained questions; for uncertain rewrite ask clarification; cap complex multi-query fan-out and cancel unfinished work when the request deadline expires.
- **Fallback behavior:** Use the unchanged original query only when it is self-contained; otherwise return clarification/unavailable rather than guessing or widening access.
- **Impact on the user/system:** A clarification or slightly slower complex request; prevents wrong-domain searches and runaway fan-out.
- **Monitoring/alerting required:** Track rewrite failures/confidence, intent-preservation review, subqueries per request, cancellation, fan-out cost, and p95 latency by route.

### Scenario: Offline evaluation or optional trace export fails

- **Why it can happen:** Ragas/evaluator dependency failure, LangSmith export outage, quota exhaustion, or privacy redaction failure.
- **How the architecture detects it:** Evaluation job status, export errors, DLP/redaction findings, and trace-access audit events.
- **How the system handles/recover from it:** Retry or pause background evaluation/export; quarantine unsafe traces; preserve local versioned evaluation artifacts and Azure operational traces.
- **Fallback behavior:** Continue serving through deterministic validation and OTel/Azure monitoring; evaluator and trace-platform availability never gates user requests.
- **Impact on the user/system:** Delayed experiment insights, with no user-path outage.
- **Monitoring/alerting required:** Alert on evaluator backlog/failure, export and redaction errors, unexpected trace content, and unapproved access.

### Backend dependency recovery flow

```mermaid
flowchart TD
    Req[Validated request] --> Call[Call dependency with deadline]
    Call --> Result{Dependency result}
    Result -- Success --> Validate[Validate freshness, schema and authorization scope]
    Result -- Transient error --> Retry{Retry budget remains?}
    Retry -- Yes --> Backoff[Backoff with jitter]
    Backoff --> Call
    Retry -- No --> Circuit[Open circuit / mark dependency unhealthy]
    Result -- Denied or invalid --> Reject[Fail closed and audit]
    Validate --> Good{Valid result?}
    Good -- Yes --> Continue[Continue workflow]
    Good -- No --> Reject
    Circuit --> Fallback{Safe fallback available?}
    Fallback -- Yes --> Limited[Return limited answer or use approved alternative]
    Fallback -- No --> Escalate[Return unavailable or route to human review]
    Limited --> Audit[(Correlated audit and telemetry)]
    Escalate --> Audit
    Reject --> Audit
    Continue --> Audit
```

This flow bounds retries by a deadline and prevents invalid or unauthorized dependency results from entering the model context. A fallback is used only when it is explicitly safe; otherwise, the request fails visibly and is audited.
