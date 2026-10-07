# RAG Architecture for Insurance AI Assistant

## Overview

This document describes the Retrieval-Augmented Generation (RAG) architecture for the insurance AI assistant. The goal is to answer customer and advisor questions with high accuracy, enterprise-grade security, source-grounding, and policy-aware reasoning.

The RAG layer is the heart of the system because insurance knowledge is mostly in long, dense, policy-heavy documents such as:

- policy documents
- terms and conditions
- product brochures
- claims procedures
- FAQs
- regulatory guidance

The system must not rely on the LLM to "remember" policy rules. It must retrieve the relevant evidence first, then reason over it, and then answer with citations.

---

## Conversation History & Context Management

Multi-turn conversation history is a first-class input to query understanding, but it is not a substitute for authoritative policy documents or customer systems of record. Conversation memory helps resolve references and preserve continuity; every policy or coverage claim must still be verified against currently authorized, version-valid sources.

### Conversation state and storage

- Assign each conversation a stable opaque ID, tenant/owner scope, creation time, last activity, and retention/expiry policy.
- Store the canonical message history in the approved conversation store with encryption at rest and in transit, access checks on every read, and audit logging. MongoDB may hold durable conversation metadata/history where approved by the enterprise data-retention policy; Redis is only an optional short-lived cache, not the source of truth.
- Keep message roles, timestamps, turn IDs, and links to retrieved evidence and generated answers. Do not store credentials or unnecessary sensitive data in conversation memory.
- Apply deletion, retention, legal-hold, and subject-access requirements consistently to raw turns, summaries, embeddings, caches, and trace records.
- Scope history retrieval to the authenticated user, tenant, conversation ID, and permitted customer/policy context. Never retrieve across users or conversations merely because text is similar.

### Conversation-aware query rewriting

For a follow-up such as “What about the waiting period?”, the query understanding service should resolve the missing subject from the current conversation before document retrieval.

1. Keep the user’s latest message unchanged as the canonical input.
2. Load a compact session summary and retrieve a bounded set of relevant prior turns (see relevant-history retrieval below).
3. Resolve references such as “it”, “that plan”, or “the waiting period” using recent turns and explicit customer/product context.
4. Produce a standalone rewritten search query and structured carry-forward constraints (for example, product, coverage type, jurisdiction, policy year, and unresolved ambiguity).
5. Validate that the rewrite preserves the user’s intent and does not broaden customer, policy, or authorization scope. Keep the original query alongside the rewrite for tracing.
6. Run document retrieval using the rewritten query and current authorization/effective-date filters. Do not treat prior assistant statements as factual policy evidence.

For example, after discussing the Health Plus policy, “What about the waiting period?” can be rewritten as “What waiting period applies under the Health Plus policy?” while retaining the original question and the policy scope from verified context.

Rewriting is a query transformation, not answer generation. It should use a low-latency model or deterministic logic where suitable, return a typed result containing the original query, standalone query, resolved entities/constraints, source turn IDs, confidence, and clarification status, and have a bounded timeout. If references cannot be resolved confidently, ask a clarifying question rather than guessing.

### Relevant-history retrieval

- First include the most recent turns needed to preserve conversational continuity, subject to the token budget.
- For older turns, retrieve only semantically relevant messages or compact turn summaries scoped to the same authorized conversation and subject.
- Initial operating limit: include at most the latest four user/assistant turn pairs or the history token allocation (15% of the model context budget), whichever is reached first; retrieve no more than five older turns/summaries unless an evaluated workflow has a documented need.
- Prefer user statements and explicit confirmed details over previous assistant-generated prose. Carry forward user-provided preferences or facts only as conversational context; verify policy/customer facts against systems of record.
- Apply recency and relevance ranking, deduplicate overlapping turns, and exclude unrelated or superseded discussion.
- Do not allow retrieved history to override current permissions, current customer context, or current policy/document versions.
- If no relevant history is found, the latest question remains a standalone query; if a critical reference remains ambiguous, ask the user to clarify.

### Summarization and long-conversation management

- Maintain a compact rolling summary when a conversation exceeds the recent-turn budget. Summaries should capture topic, confirmed user-provided details, selected product/policy context, unresolved questions, and relevant turn IDs.
- Clearly label summary fields by provenance: user-provided, system-of-record verified, or assistant-generated. Treat summaries as navigation/context only, never as authoritative evidence.
- Refresh summaries from canonical turns, not from a previous summary alone; preserve links to source turns so a summary can be checked or corrected.
- Keep a sliding window of recent turns plus retrieved relevant older turns. Do not append the complete transcript to the model prompt.
- Start summary refresh when the recent-turn window would exceed its budget or before adding a new topic segment; enforce maximum summary tokens using the same model-specific tokenizer as prompt construction.
- On topic change, start a new topic segment or summary scope while retaining the conversation ID; on a new conversation, do not reuse history unless explicitly requested and authorized.
- If summary generation or storage fails, continue with the bounded recent-turn window when possible; otherwise ask the user to restate the required context.

### Context-window and token-budget policy

Use model-specific tokenization and configure budgets per model/version. A starting model-context budget (including reserved completion capacity) can allocate approximately:

| Context element | Starting budget | Policy |
|---|---:|---|
| System, safety, and output instructions | 10% | Fixed and never displaced by history |
| Latest user message and rewritten query | 10% | Always preserve the original message |
| Relevant recent turns and summary | 15% | Select only relevant, scoped context |
| Customer context and business rules | 15% | Minimize data; authoritative sources only |
| Retrieved document evidence | 40% | Rank and trim to the strongest citation-ready chunks |
| Reserved output and safety margin | 10% | Protect completion length and model/tokenizer variance |

These are initial planning values, not fixed limits. Enforce a hard input-token ceiling below the model context limit after reserving output tokens, and trim in this order: unrelated older history, redundant summary details, low-ranked evidence, then nonessential customer context. Never trim safety instructions, authorization constraints, the latest user message, or citation requirements. If the remaining evidence cannot fit without losing necessary qualifications, split the workflow, retrieve a narrower evidence set, or ask a clarifying question; do not silently truncate policy clauses.

### Privacy, correctness, and evaluation controls

- Encrypt stored conversation data and embeddings; apply least-privilege access, retention, deletion, and audit policies.
- Do not place sensitive raw conversation text in cache keys, metrics, or ordinary logs. Use opaque IDs and redacted/aggregated telemetry.
- Isolate history retrieval and caches by tenant, user, conversation, and relevant authorization version. Invalidate cached context when permissions or policy scope change.
- Defend against prompt injection contained in prior user or assistant messages. History is untrusted input and must not alter system/developer instructions or access policy.
- Measure rewrite intent preservation, reference-resolution accuracy, correct clarification rate, history-retrieval precision, token usage, latency, and downstream retrieval/citation quality. Test ambiguous follow-ups, topic switches, stale summaries, revoked access, and very long conversations.

### Conversation-aware query flow

```mermaid
flowchart TD
    U[Latest user message] --> AUTH[Authenticate and authorize conversation access]
    AUTH --> LOAD[Load bounded recent turns and summary]
    LOAD --> HIST[Retrieve relevant older turns<br/>same user, tenant and conversation]
    U --> REWRITE[Conversation-aware query rewriter]
    LOAD --> REWRITE
    HIST --> REWRITE
    REWRITE --> RESOLVE{Reference resolved confidently?}
    RESOLVE -- No --> CLARIFY[Ask a clarifying question]
    RESOLVE -- Yes --> VALIDATE[Validate intent and scope<br/>preserve original query]
    VALIDATE --> RETRIEVE[Search authorized current policy sources]
    RETRIEVE --> EVIDENCE[Ranked policy evidence]
    EVIDENCE --> ANSWER[Generate and validate grounded response]
    ANSWER --> STORE[Persist turn, evidence links and audit metadata]
    STORE --> SUMMARY{Summary refresh required?}
    SUMMARY -- Yes --> UPDATE[Refresh compact provenance-aware summary]
    SUMMARY -- No --> DONE[Conversation state ready for next turn]
    UPDATE --> DONE
    CLARIFY --> STORE
```

The history store and relevant-turn retrieval provide bounded conversational context; the rewriter converts a follow-up into a standalone search query while preserving scope and the original wording. The confidence gate prevents guessing when a reference is ambiguous. Policy retrieval remains a separate authoritative step, and evidence links are stored with the new turn so later history can distinguish verified sources from assistant text. Summarization is refreshed only as needed and remains traceable to canonical turns.

---

## 1. Objectives of the RAG Layer

The RAG architecture must:

- retrieve policy-relevant evidence with high precision
- filter results by product, policy version, region, coverage type, and user permissions
- improve retrieval quality with re-ranking
- ground answers in source documents and chunks
- support both FAQ and policy reasoning use cases
- resolve multi-turn follow-up questions using bounded, permission-scoped conversation history
- reduce hallucinations and unsupported conclusions
- support auditability and explainability for regulated workflows

---

## 2. Major RAG Workflows

The RAG lifecycle is split into three workflows so ingestion, query-time retrieval, and response generation can be scaled, secured, monitored, and recovered independently:

1. **Document ingestion pipeline** prepares approved source documents and publishes versioned chunks to the search index.
2. **Query retrieval pipeline** authenticates and scopes a request, finds relevant evidence, and returns a ranked evidence set.
3. **Final RAG response flow** generates a constrained answer, validates it, attaches citations, and escalates when confidence or risk requires human review.

---

## 3. RAG Design Principles

### 3.1 Retrieval-first design
The model should not answer from raw memory alone. It should retrieve evidence first and reason over it.

### 3.2 Policy-aware filtering
A customer’s answer depends on:

- policy number or policy ID
- product type
- coverage type
- policy effective date
- claim type
- region / jurisdiction
- document version / issue date
- user role and permission scope

### 3.3 Evidence-first answering
Every answer must map back to one or more source chunks. The final response should cite relevant policy references and claim procedures.

### 3.4 Precision over recall
For insurance content, an overly broad retrieval set causes poor answers. The system should prefer smaller, higher-quality evidence sets with good reranking.

---

## 4. Knowledge Sources

The RAG system draws on multiple sources:

- insurance policy documents
- product brochures
- claim manuals and claim guidelines
- terms and conditions
- FAQs and internal knowledge articles
- customer policy records and claims data
- regulatory and legal references where needed

These are not all treated equally. Some are internal and role-restricted. Others are public-facing or advisor-facing only.

---

## 5. Document Ingestion Pipeline

```mermaid
flowchart TD
    A[Admin uploads approved document] --> B[(Azure Blob Storage<br/>immutable source)]
    B --> C[Document-created event]
    C --> D[Event broker<br/>Event Hubs or Kafka]
    D --> E[Airflow ingestion workflow]
    E --> F[Azure AI Document Intelligence]
    F --> G[OCR, layout and table extraction]
    G --> H[Normalize text and preserve page/section provenance]
    H --> I{Extraction and metadata quality checks}
    I -- Pass --> J[Insurance-aware chunking]
    I -- Fail --> X[Quarantine and operations review]
    J --> K[Enrich chunks with version, product, jurisdiction and access metadata]
    K --> L[Generate embeddings]
    L --> M[(Qdrant<br/>versioned vector payloads)]
    K --> N[(MongoDB<br/>document, chunk and processing metadata)]
    M --> O[Index validation]
    N --> O
    O --> P[Mark document version searchable]
    E -. transient processing failure .-> RETRY{Retry budget remains?}
    RETRY -- Yes --> E
    RETRY -- No --> DLQ[Dead-letter queue / operations review]
```

### Pipeline explanation

The ingestion workflow turns an approved source file into searchable, traceable evidence. Blob Storage retains the immutable original; the event broker decouples uploads from processing bursts; and Airflow coordinates retryable, observable steps. Document Intelligence extracts OCR, layout, and table structure, while normalization and quality checks catch extraction problems before indexing. Insurance-aware chunking keeps clauses and tables interpretable, and enriched metadata supports policy, version, and access filters. Qdrant serves vector retrieval; MongoDB records document lineage, chunk metadata, and processing state. Index validation ensures a version is not exposed as searchable until its vectors and metadata are consistent. Failed or low-quality documents are quarantined or retried rather than silently published.

---

## 6. Chunking Strategy

Insurance documents are often long and clause-heavy. Therefore, simple chunking is not enough.

### Recommended chunking rules

- Prefer semantic chunking over arbitrary fixed-size splitting
- Preserve section boundaries where possible
- Keep table data and policy terms together when they belong to the same clause
- Avoid splitting across legal definitions or exclusions if it creates ambiguity
- Add metadata at chunk level

### Example chunk metadata

- document_id
- version_id
- product_type
- policy_year
- coverage_line
- jurisdiction
- section_title
- page_number
- source_url
- extraction_quality_score
- created_at

---

## 7. Hybrid Retrieval Strategy

Hybrid retrieval performs best for insurance documents.

### 7.1 Semantic search
Semantic retrieval uses embeddings to find conceptually similar text.

Use cases:

- vague questions such as “what does this policy cover after hospitalization?”
- cross-document paraphrases
- broad contextual intent matching

### 7.2 Lexical search
Lexical search helps with exact terms and policy wording.

Use cases:

- clause names
- exclusions such as “pre-existing condition”
- policy wording like “deductible” or “waiting period”

### 7.3 Metadata filtering
This is essential in enterprise insurance contexts.

Examples:

- policy type = health insurance
- region = India / United States / APAC
- effective_date between X and Y
- document version = latest approved version only
- customer segment = retail / SME / corporate

### 7.4 Why hybrid retrieval matters
Insurance questions often combine vague natural language with exact policy terminology. Semantic search alone is not enough; lexical + metadata filtering increases precision.

---

## 8. Re-ranking Layer

After retrieval, the system should rerank candidate chunks to keep only the strongest evidence.

### Recommended approach

- Use a cross-encoder model such as BGE cross-encoder or a comparable reranker
- Rank top-k chunks based on relevance to the query and customer context
- Use metadata weighting as part of the ranker decision

### Why reranking is necessary

Naive vector search often retrieves many nearby but irrelevant chunks. Re-ranking reduces noise and dramatically improves answer quality for legal and policy content.

### Query Retrieval Pipeline

```mermaid
flowchart TD
    U[Customer or advisor query] --> API[Authenticated assistant API]
    API --> ACL[Authorize user and resolve tenant / customer scope]
    ACL --> HISTORY[Load bounded recent turns and summary]
    ACL --> HISTRET[Retrieve relevant older turns<br/>same authorized conversation]
    U --> REWRITE[Conversation-aware query rewriting]
    HISTORY --> REWRITE
    HISTRET --> REWRITE
    REWRITE --> RESOLVE{Follow-up reference resolved?}
    RESOLVE -- No --> CLARIFY[Ask for clarification]
    RESOLVE -- Yes --> QN[Create standalone query<br/>retain original for audit]
    QN --> CTX[Customer Context Service]
    CTX --> SYS[Policy / claims systems of record]
    CTX --> FILTER[Build metadata and effective-date filters]
    ACL --> FILTER
    QN --> CACHE{Authorized retrieval cache hit?}
    CACHE -- Yes --> CACHED[Cached candidate chunks]
    CACHE -- No --> SEARCH[Run hybrid retrieval]
    FILTER --> SEARCH
    SEARCH --> HEALTH{Search services healthy?}
    HEALTH -- Yes --> DENSE[Dense semantic search]
    HEALTH -- Yes --> LEX[Lexical search for exact terms]
    HEALTH -- No --> RETRY{Retry budget remains?}
    RETRY -- Yes --> BACKOFF[Bounded backoff]
    BACKOFF --> SEARCH
    RETRY -- No --> UNAVAILABLE[Retrieval unavailable]
    UNAVAILABLE --> ESC[No-answer or human-review path]
    DENSE --> FUSE[Merge and deduplicate candidates]
    LEX --> FUSE
    FUSE --> RANK[Cross-encoder reranker]
    RANK --> GATE{Access, version and relevance checks}
    CACHED --> GATE
    GATE -- Pass --> EVID[Ranked evidence chunks]
    GATE -- Fail / low confidence --> ESC[No-answer or human-review path]
    EVID --> TRACE[Record retrieval trace and scores]
    ESC --> TRACE
    CLARIFY --> TRACE
    QDRANT[(Qdrant)] --> DENSE
    LEXIDX[(Lexical index, if deployed)] --> LEX
    REDIS[(Redis, optional cache)] --> CACHE
    TRACE --> NEXT[Pass evidence or escalation status to response flow]
```

The query pipeline authenticates the caller and resolves customer and tenant scope before loading any conversation history. A bounded recent-turn window, compact summary, and scoped relevant-history retrieval feed a conversation-aware rewriter that produces a standalone query; if the follow-up is still ambiguous, the system asks for clarification instead of guessing. The original wording is retained for audit. Customer context then produces metadata filters for product, jurisdiction, effective date, document version, and permissions. Dense search in Qdrant handles paraphrases, while an optional lexical index finds exact clause terms; candidate fusion and cross-encoder reranking improve precision. Access, version, and relevance gates prevent unauthorized or stale evidence from proceeding. Redis can reduce repeat-query latency only when the cache key includes authorization, conversation, and policy scope. Retrieval scores and source identifiers are traced for evaluation and audit; weak or disallowed evidence leads to an explicit no-answer or review status rather than forced generation.

---

## 9. Context Assembly

Once the relevant chunks are selected, the system assembles a grounded context for the model.

### Context may include

- the standalone rewritten query plus the original latest user message
- a compact conversation summary and only the relevant prior turns needed for continuity
- provenance labels for remembered details (user-provided, system-verified, or assistant-generated)
- top retrieved chunks
- top metadata-filtered chunks
- customer policy details
- claim-specific context
- answer-style rules and formatting instructions

Conversation history is context for resolving references and maintaining continuity, not evidence for policy claims. Do not pass the complete transcript. The prompt builder enforces the model-specific token budget defined in [Conversation History & Context Management](#conversation-history--context-management), preserving system/safety instructions, the current question, required policy qualifications, and citations before optional older history.

### Example of context payload

```json
{
  "original_query": "What about the waiting period?",
  "rewritten_query": "What waiting period applies under the Health Plus policy?",
  "conversation_context": {
    "summary": "User is asking about Health Plus inpatient coverage.",
    "relevant_turn_ids": ["turn-18", "turn-19"],
    "provenance": "user-selected product; policy terms must be verified from current source documents"
  },
  "customer_context": {
    "policy_id": "POL-1245",
    "product": "Health Plus",
    "policy_year": 2025,
    "coverage_type": "inpatient"
  },
  "resolved_constraints": {
    "jurisdiction": "authorized policy jurisdiction",
    "policy_version": "current version for effective date"
  },
  "retrieved_chunks": [
    {"doc_id": "policy-health-2025", "chunk_id": "c-341", "score": 0.94}
  ]
}
```

---

## 10. Grounded Answer Generation

The LLM should answer only from the retrieved context and policy rules. It should be explicitly instructed to:

- answer conservatively
- cite supporting evidence
- mention uncertainty where policy language is unclear
- avoid assuming customer coverage without evidence
- produce structured responses when needed

### Example answer behavior

- “According to Section 3.2 of the Health Plus policy, emergency inpatient treatment is covered when admitted within 24 hours of an accident.”
- “This answer is based on the latest policy version effective 1 Jan 2025.”

### Final RAG Response Flow

```mermaid
flowchart TD
    START[Ranked evidence + customer context<br/>rewritten query + selected history] --> CHECK{Evidence sufficient and permitted?}
    CHECK -- No --> SAFE[Return limitation or request clarification]
    CHECK -- Escalation required --> HUMAN[Route to advisor / human review]
    CHECK -- Yes --> ASSEMBLE[Assemble bounded prompt context<br/>relevant history, evidence, rules and format]
    ASSEMBLE --> BUDGET{Within input token budget?}
    BUDGET -- No --> TRIM[Trim unrelated history first,<br/>then low-ranked evidence]
    TRIM --> FIT{Required evidence and instructions fit?}
    FIT -- Yes --> REDACT
    FIT -- No --> SAFE
    BUDGET -- Yes --> REDACT[Minimize or mask sensitive data]
    REDACT --> MODEL[LLM generates structured draft]
    MODEL --> GENERATED{Draft generated successfully?}
    GENERATED -- Yes --> VALIDATE[Validate schema, policy constraints and groundedness]
    GENERATED -- Transient failure --> RETRY{Retry budget remains?}
    RETRY -- Yes --> BACKOFF[Bounded backoff]
    BACKOFF --> MODEL
    RETRY -- No --> SAFE
    VALIDATE --> CITATION[Resolve citations to document, version, section and page]
    CITATION --> PASS{Validation and citation checks pass?}
    PASS -- No --> SAFE
    PASS -- Yes --> FINAL[Return answer with citations and uncertainty]
    SAFE --> AUDIT[(Audit / trace record)]
    HUMAN --> AUDIT
    FINAL --> AUDIT
    AUDIT --> METRICS[Emit latency, quality, safety and cost telemetry]
```

This response flow invokes generation only when retrieved evidence is both authorized and sufficient. Bounded context assembly includes only a rewritten query, selected history, relevant evidence, and approved rules—not the full transcript. The token-budget gate reserves room for safety instructions and output; it trims unrelated history and then low-ranked evidence, and falls back to clarification/review if required evidence cannot fit intact. Data minimization reduces unnecessary exposure of customer information. Structured generation is followed by schema, policy, groundedness, and citation checks so unsupported or malformed answers do not reach users. The audit record links the request, selected history references, evidence, model/configuration, validation outcome, and citations; telemetry supports production monitoring and continuous evaluation.

---

## 11. Guardrails in the RAG Layer

The RAG pipeline must include safety and compliance checks.

### Input guardrails

- detect prompt injection
- sanitize user input
- prevent malicious retrieval manipulation

### Retrieval guardrails

- filter out documents the user is not authorized to read
- block stale or superseded policy versions
- remove irrelevant or low-quality documents before ranking

### Output guardrails

- enforce groundedness checks
- prevent unsupported claims
- require citation presence for factual or policy-based answers
- flag answers requiring human review

### Sensitive scenarios

- claim denial explanation
- legal interpretation of policy clauses
- premium disputes
- customer-specific coverage decisions
- ambiguous exclusions or regulatory concerns

These should often route to human review or advisor workflows.

---

## 12. Citation Strategy

Citations are not optional in an enterprise insurance assistant.

### Required citation components

- source document name
- section name or clause title
- document version
- page or chunk identifier
- time of policy validity

### Example

“Coverage is available for emergency hospitalization under the Health Plus policy, Section 4.3, Policy Version 2025.01 (source: Policy Doc 2025, chunk H-173).”

This helps with:

- auditability
- compliance review
- advisor trust
- customer transparency

---

## 13. RAG Evaluation Strategy

A production-grade RAG system needs systematic evaluation.

### Retrieval metrics

- recall@k
- ndcg@k
- MRR
- precision@k
- document hit rate

### Answer quality metrics

- groundedness
- citation correctness
- answer faithfulness
- refusal correctness
- usefulness for customer or advisor workflow

### Example evaluation scenarios

- claim eligibility question
- product comparison question
- claim procedure question
- policy exclusion question
- answer requiring routing to a human advisor

---

## 14. Failure Modes in RAG

### Retrieval failure
- no relevant chunks found
- wrong document version selected
- too many irrelevant results

### Answer failure
- the model hallucinates policy language
- the answer is too generic and not policy-specific
- the citation is missing or wrong

### Data failure
- stale document remains indexed
- OCR extraction is poor
- table data is misread
- metadata is incomplete

### Mitigation

- validation before indexing
- retrieval quality checks
- citation requirement as a hard gate
- reprocessing and version correction pipeline
- human review for domain-sensitive results

---

## 15. Security and Compliance in RAG

The RAG layer must enforce strict access rules.

### Key requirements

- user can only access authorized documents
- customer data access follows RBAC/ABAC policies
- documents with expired or superseded versions should not be used
- PII should be masked before model execution when necessary
- all retrievals and answers should be auditable

No answer should be generated from a document the user is not allowed to access, even if it exists in the index.

---

## 16. Performance and Scaling Considerations

RAG must scale across both user queries and ingestion workloads.

### Scaling techniques

- asynchronous ingestion workers
- horizontal scaling of retrieval and API services
- metadata filtering to reduce search cost
- vector index replication or sharding where needed
- Redis caching for common retrieval patterns and FAQ summaries
- query-time pruning of low-value documents

### Latency optimization

- do not retrieve full-document context for simple FAQ lookups
- precompute common retrieval patterns
- keep chunk sizes tight and context-aware
- use reranking only on a limited candidate set

---

## 17. RAG Architecture Decision Summary

The recommended architecture is:

- Qdrant for vector storage and retrieval
- metadata-aware filtering for policy relevance
- hybrid retrieval for semantic + lexical matching
- cross-encoder re-ranking for precision
- evidence-grounded answer generation with citations
- strict guardrails and access control
- async ingestion from source documents to vector index
- version-aware and auditable document processing

This architecture gives the best balance of accuracy, safety, enterprise readiness, and explainability for an insurance assistant.

---

## 18. Edge Case & Failure Handling

RAG must fail closed whenever source quality, authorization, version, or grounding cannot be verified. The ingestion, retrieval, and response diagrams above show the primary quarantine, retry, no-answer, validation, and escalation paths.

### Scenario: Duplicate upload or replayed ingestion event

- **Why it can happen:** Event brokers provide at-least-once delivery, admins may retry an upload, or an orchestrator may replay a job after timeout.
- **How the architecture detects it:** Use a stable document ID plus source checksum, version ID, and idempotency key; compare processing state and index records before writes.
- **How the system handles/recover from it:** Make workflow stages idempotent. Reuse completed extraction results when valid, or replace the same version atomically; do not create duplicate chunks or embeddings.
- **Fallback behavior:** Keep the last validated searchable version while the duplicate/replay is reconciled; quarantine conflicting content for operations review.
- **Impact on the user/system:** Normally none; unresolved conflicts can delay publication of the affected version.
- **Monitoring/alerting required:** Count duplicate events, idempotent no-ops, conflicting checksums, and replay attempts; alert on repeated conflicts or unusual event bursts.

### Scenario: OCR or layout extraction is incomplete or corrupt

- **Why it can happen:** Scanned or rotated pages, low-resolution images, handwritten annotations, unsupported tables, encrypted PDFs, or Document Intelligence outages.
- **How the architecture detects it:** Validate extraction completeness, page coverage, table structure, OCR confidence, required metadata, and document-level quality thresholds.
- **How the system handles/recover from it:** Retry transient service failures with bounded backoff; apply approved preprocessing or alternate extraction for supported cases; persist the original and extraction diagnostics; route low-quality documents to manual review.
- **Fallback behavior:** Do not index a document/version that failed required quality checks; retain an older version only if its validity period still applies.
- **Impact on the user/system:** Newly updated content may not be searchable; the assistant avoids giving answers from corrupted or partial text.
- **Monitoring/alerting required:** Track extraction latency, confidence, page/table coverage, retries, quarantine volume, and failure rates by source type.

### Scenario: Partial indexing or inconsistent document-version publication

- **Why it can happen:** Qdrant writes succeed while MongoDB metadata writes fail, indexing is interrupted, or an update is made searchable before all chunks are validated.
- **How the architecture detects it:** Compare expected and actual chunk counts, IDs, version metadata, and checksums across MongoDB and Qdrant before marking a version active.
- **How the system handles/recover from it:** Stage writes under a non-searchable version, retry idempotently, validate cross-store consistency, and activate the version only after validation. Remove or rebuild incomplete staged data.
- **Fallback behavior:** Continue serving the last validated version only when effective-date rules allow; otherwise return an unavailable-source result and escalate.
- **Impact on the user/system:** Index freshness may be delayed, but incomplete or mixed-version policy evidence is not served.
- **Monitoring/alerting required:** Alert on index/metadata count mismatch, activation delay, failed validation, orphan vectors, and active-version drift.

### Scenario: Unauthorized or cross-tenant retrieval/cache collision

- **Why it can happen:** Incorrect ACL metadata, a missing tenant filter, stale permissions, or a retrieval cache key that omits user/customer scope.
- **How the architecture detects it:** Enforce authorization before retrieval and again before context assembly; test tenant boundaries; include access scope and policy version in cache keys and verify cached payload metadata.
- **How the system handles/recover from it:** Reject the result, invalidate affected cache entries, prevent prompt assembly, and raise a security event for investigation. Restrict cache use if scope cannot be proven.
- **Fallback behavior:** Return a generic access failure without confirming the existence or contents of protected documents.
- **Impact on the user/system:** Affected requests fail safely; a confirmed leak is a security incident requiring containment and response.
- **Monitoring/alerting required:** Alert on ACL-filter failures, cache scope mismatches, cross-tenant test failures, and anomalous denied/allowed retrieval patterns.

### Scenario: No relevant, current, or sufficiently confident evidence

- **Why it can happen:** Query is ambiguous, a policy is not indexed, metadata is wrong, the requested coverage is not documented, or retrieval/reranking misses relevant clauses.
- **How the architecture detects it:** Apply minimum relevance and evidence coverage thresholds; check document effective dates, required source types, and expected citation availability.
- **How the system handles/recover from it:** Optionally reformulate or broaden the query within the same authorization scope and run a bounded second retrieval; preserve the original query and scores for audit.
- **Fallback behavior:** Do not fabricate an answer. Ask a clarifying question, state that supporting evidence was not found, or route to an advisor.
- **Impact on the user/system:** Reduced self-service resolution and possible human workload, but lower risk of unsupported insurance guidance.
- **Monitoring/alerting required:** Track no-hit and low-confidence rates by product, question intent, document version, and retrieval configuration; alert on regressions from baseline.

### Scenario: Grounding, citation, or model output validation fails

- **Why it can happen:** Model output is malformed, cites a nonexistent chunk, combines conflicting clauses, omits a required qualification, or a prompt/model update regresses quality.
- **How the architecture detects it:** Validate response schema, citation IDs against retrieved evidence, document version and page references, policy constraints, and groundedness thresholds.
- **How the system handles/recover from it:** Reject the draft; allow at most a bounded regeneration using the same validated evidence if policy permits; otherwise record the failed trace and route to review. Roll back a harmful prompt/model change.
- **Fallback behavior:** Return a safe limitation or human-review path, never the unvalidated draft.
- **Impact on the user/system:** The user may receive a delayed or limited answer; incorrect citations and unsupported coverage statements are withheld.
- **Monitoring/alerting required:** Monitor validation rejection, regeneration, citation mismatch, groundedness, safety escalation, and model/prompt-version quality metrics.

### Scenario: Follow-up reference is ambiguous or rewritten query changes intent

- **Why it can happen:** The user changes topics, multiple products or waiting periods were discussed, or the rewriter incorrectly resolves “it”, “that plan”, or another reference.
- **How the architecture detects it:** Compare the rewritten query and extracted constraints with the current message and relevant turn references; use a confidence threshold and detect competing antecedents.
- **How the system handles/recover from it:** Preserve the original query, reject an intent-changing rewrite, and ask a targeted clarification when the intended subject cannot be selected reliably.
- **Fallback behavior:** Do not search or answer against a guessed product or coverage context. The user may restate the subject or start a new topic.
- **Impact on the user/system:** One additional interaction may be needed; prevents retrieval against the wrong policy.
- **Monitoring/alerting required:** Measure rewrite confidence, intent-preservation failures, clarification rate, and retrieval/citation quality for follow-up questions.

### Scenario: Long conversation, stale summary, or summary-generation failure

- **Why it can happen:** The conversation exceeds the prompt budget, a topic changes, a summary omits a qualifier, or the summarizer/storage dependency fails.
- **How the architecture detects it:** Track token estimates and summary age/version; retain source-turn references; validate summary fields and compare resolved constraints with recent canonical turns.
- **How the system handles/recover from it:** Rebuild summaries from canonical turns, segment by topic, and retrieve only relevant scoped history. Apply the defined trimming order and never truncate required policy evidence or safety instructions.
- **Fallback behavior:** Use the bounded recent-turn window if the summary is unavailable; if required context no longer fits or cannot be verified, ask the user to restate it.
- **Impact on the user/system:** May add latency or require restatement, while keeping prompt size bounded and preventing lost qualifications from silently affecting the answer.
- **Monitoring/alerting required:** Track summary refresh failures/age, token-budget overruns, history-retrieval latency, context-trim counts, and clarification/restatement rates.

### Scenario: Conversation history is inaccessible, revoked, or belongs to another scope

- **Why it can happen:** Conversation ownership changes, session IDs are guessed or reused, permissions are revoked, cache keys omit scope, or history storage is unavailable.
- **How the architecture detects it:** Reauthorize every history read against user, tenant, conversation, and customer scope; validate cache metadata and reject missing ownership/version claims.
- **How the system handles/recover from it:** Fail closed for unauthorized history, invalidate affected cached context, and emit a security audit event. For storage outages, use only history already loaded and verified in the current request if policy permits.
- **Fallback behavior:** Treat the latest message as standalone or request the user to restate context; never borrow history from a different conversation or tenant.
- **Impact on the user/system:** Reduced continuity or a clarification step; no cross-session or cross-tenant disclosure.
- **Monitoring/alerting required:** Alert on denied history access, cache-scope mismatch, history-store outages, and anomalous conversation-ID access patterns.

---
