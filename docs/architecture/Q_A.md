# Insurance RAG Architecture --- Production Interview Q&A

> **Basis:** Answers are tailored to the
> `rag-architecture-multi-agent-architecture-hardened.md` design: a
> supervisor-orchestrated LangGraph multi-agent workflow, FastAPI,
> approved document ingestion, MongoDB metadata/versioning, Qdrant
> retrieval, optional BM25 hybrid retrieval, BGE reranking, grounded LLM
> synthesis, deterministic validation, OpenTelemetry/Azure Monitor, and
> offline evaluation.
>
> **Interview framing:** Treat the architecture as the production design
> being discussed, but distinguish established design contracts from
> choices the architecture itself marks as requiring approval or
> benchmarking. Do not claim a particular queue, durable checkpoint
> store, claim classifier, numeric SLO, or semantic claim-support
> validator is implemented unless it has actually been selected, built,
> and tested.

## 1. A user asks a question and the LLM gives a wrong answer. How do you find whether the problem is retrieval, prompt, context, or LLM behavior?

**Answer**

I debug the request as a traceable pipeline, not by changing the prompt
immediately. I follow one `request_id` across the supervisor, specialist
nodes, retrieval, reranking, context assembly, model call, and
deterministic validation. I compare the answer against the exact
evidence and configuration used for that request.

### Investigation sequence

1.  **Reproduce and preserve the case.** Record the original question,
    route selected by the Supervisor, timestamp, document/index
    generation, model and prompt versions, and the incorrect answer.
    Redact or minimize PII.
2.  **Check routing and authorization.** Confirm the Supervisor selected
    the correct workflow and that trusted tenant, ACL, product,
    jurisdiction, effective-date, and active-version filters were
    applied. A missing or overly broad filter is a routing/security
    defect, not an LLM issue.
3.  **Inspect first-stage retrieval.** Review candidate chunk IDs,
    document/version IDs, scores, ranks, and source offsets. Ask whether
    the correct document and the correct passage were present in the
    initial candidate set.
4.  **Inspect reranking.** If the correct passage was retrieved but
    moved down or excluded after BGE reranking, compare its pre-rerank
    and post-rerank rank and inspect competing passages. If hybrid
    retrieval is enabled, also inspect dense and BM25 candidate lists
    and the fusion result.
5.  **Inspect context assembly.** Confirm the right chunks survived
    deduplication, truncation, context-budget limits, ordering, and any
    optional compression. The correct chunk can be retrieved but omitted
    from the final prompt.
6.  **Inspect the exact model input.** Review the rendered
    system/developer instructions, user question, evidence passages,
    citation identifiers, output schema, model/version, generation
    parameters, and token budget. Apply approved redaction controls; do
    not log unrestricted sensitive payloads by default.
7.  **Classify the failure.**
    -   Correct evidence never retrieved: retrieval/indexing/filtering
        problem.
    -   Correct document but wrong passage ranked or selected: chunking,
        retrieval, or reranking problem.
    -   Correct passage retrieved but omitted or distorted in assembled
        context: context construction/budget problem.
    -   Correct context supplied but the prompt permits unsupported
        inference or gives conflicting instructions: prompt/contract
        problem.
    -   Correct, clear evidence and instructions supplied, but the model
        still misstates it: generation/model behavior problem.
    -   Answer text is plausible but citations, policy version, or
        required fields are invalid: output validation/contract problem.
8.  **Fix the responsible layer and add a regression case.** Do not tune
    prompts to compensate for bad retrieval, and do not increase
    retrieval `top_k` blindly. Each can hide the symptom while adding
    cost or introducing irrelevant context.

**What I would say in an interview:** "I locate the earliest stage where
the expected evidence or behavior diverges from the trace. That lets me
distinguish retrieval recall, ranking, context assembly, prompt
contract, model generation, and validation failures with evidence rather
than guesswork."

------------------------------------------------------------------------

## 2. How exactly would you log the problem? How would you do it without LangSmith or Ragas?

**Answer**

Specialized LLM tooling is optional. I would implement structured
application telemetry with OpenTelemetry and Azure Monitor/Application
Insights, using a common `request_id` and trace/span IDs. LangSmith may
be added only if its privacy and residency requirements are approved;
Ragas is an offline evaluation tool, not a prerequisite for operational
logging.

### What I log

  ------------------------------------------------------------------------------
  Area                                Fields to capture
  ----------------------------------- ------------------------------------------
  Request                             `request_id`, timestamp, route/intent,
                                      latency, outcome/status, tenant-safe
                                      identifier, correlation/trace ID

  Supervisor / graph                  Selected route, node names, node start/end
                                      times, status, attempts, retry count,
                                      deadline/budget remaining, failure reason

  Retrieval                           Index generation, query/rewritten-query
                                      version, filter summary, dense/BM25 mode,
                                      candidate chunk IDs, document/version IDs,
                                      scores, ranks, retrieval latency

  Reranking                           Reranker model/version, input candidate
                                      IDs/ranks, output ranks/scores, latency,
                                      timeout/fallback status

  Context assembly                    Evidence IDs included in the final
                                      context, order, token count,
                                      truncation/compression decisions, dropped
                                      evidence IDs and reasons

  LLM call                            Provider/model ID and version,
                                      prompt-template version/hash, generation
                                      parameters, input/output token counts,
                                      latency, retry count, finish reason, error
                                      class

  Validation                          Schema/citation/provenance/applicability
                                      checks, claim-support check status if
                                      implemented, reason codes, final
                                      pass/fail/abstention

  Ingestion lineage                   Source document ID/version/hash, ingestion
                                      job ID, extraction/chunking/embedding
                                      versions, index generation,
                                      reconciliation/activation status

  Cost and quality                    Token/cost estimate, end-to-end latency,
                                      feedback/incident label, evaluation case
                                      ID when applicable
  ------------------------------------------------------------------------------

### How I would do it without LangSmith or Ragas

1.  Instrument each FastAPI request and LangGraph node with
    OpenTelemetry spans.
2.  Emit structured JSON logs and metrics to the organization-approved
    backend, such as Azure Monitor/Application Insights.
3.  Store retrieval trace details as structured events linked by
    `request_id`. Keep large evidence payloads out of ordinary logs;
    retain only chunk IDs and provenance by default.
4.  For sensitive debugging, provide an access-controlled diagnostic
    view that can resolve chunk IDs to source passages under the
    operator's permissions. Audit diagnostic access.
5.  Create a small reviewed evaluation dataset in version control or an
    approved test-data store. A custom Python test harness can run fixed
    questions and assert retrieval, citation, answer, abstention,
    latency, and cost expectations. Ragas can be introduced later, but
    it is not required to begin.
6.  Alert on operational signals such as rising no-hit rates,
    reranker/provider timeouts, validation failures, retry exhaustion,
    ingestion reconciliation mismatches, and latency/cost regressions.

**Privacy rule:** Do not indiscriminately log raw prompts, full customer
records, or entire policy documents. Apply PII/DLP controls, retention
rules, access controls, and redaction before telemetry leaves the
service. If the approved policy prohibits storing prompt content, log
hashes, evidence IDs, configuration versions, and reason codes instead,
with a separately controlled debugging path.

------------------------------------------------------------------------

## 3. How would you know or inspect which chunks were retrieved from the documents?

**Answer**

Every chunk needs a stable identity and source lineage. The retrieval
trace should record the chunk IDs and their source metadata, not merely
the final answer or a list of document names.

For each candidate, I inspect:

-   `chunk_id` and stable source/document ID;
-   document version and index generation;
-   page, section, clause, or source offsets;
-   product, jurisdiction, effective-date, approval/status, and ACL
    metadata used for filtering;
-   retrieval method (dense, BM25, or hybrid), raw score, rank, and
    retrieval timestamp;
-   reranker input rank, output rank, and reranker score;
-   whether the chunk made it into the final LLM context and, if not,
    why it was dropped.

A restricted diagnostic endpoint or internal UI can resolve a chunk ID
to its source passage and surrounding section. It must enforce the same
authorization rules as retrieval and audit access. Do not expose chunk
text in broadly accessible logs.

### Example diagnostic record

``` json
{
  "request_id": "req-example",
  "index_generation": "generation-2026-10-09-01",
  "retrieval_mode": "dense",
  "candidates": [
    {
      "chunk_id": "policy-123-v4-page-18-chunk-03",
      "document_id": "policy-123",
      "document_version": "v4",
      "page": 18,
      "section": "Exclusions",
      "retrieval_rank": 2,
      "rerank_rank": 1,
      "included_in_llm_context": true
    }
  ]
}
```

This is an illustrative schema, not a claim that these exact fields or
IDs already exist in the implementation. The architecture should
preserve equivalent provenance.

**Key distinction:** A document ID tells me which document was found; a
chunk ID and source offsets tell me which specific passage was found and
used.

------------------------------------------------------------------------

## 4. The correct document was retrieved, but the wrong paragraph was selected. How would you investigate?

**Answer**

I would follow the candidate passage through chunking, retrieval,
reranking, and context assembly. "Correct document retrieved" does not
prove that the correct evidence passage was selected.

1.  **Locate the expected passage in the source.** Identify the exact
    page, section/clause, table row, and source offsets. Confirm the
    source version and effective date are applicable to the question.
2.  **Inspect chunk boundaries.** Check whether the paragraph was split
    from its heading, definition, exception, table row, or qualifying
    sentence. Chunking must preserve meaningful insurance structure and
    traceable offsets.
3.  **Check metadata and filters.** Verify that the expected chunk has
    correct product, jurisdiction, approval/status, effective-date,
    tenant/ACL, and version metadata. A filter can exclude the right
    passage even when another passage from the same document appears.
4.  **Compare initial retrieval candidates.** If the expected chunk is
    absent, investigate query formulation, embedding model/version,
    vector generation compatibility, indexing completeness, and
    retrieval limits. For exact clause numbers, defined terms, or
    product names, evaluate optional BM25 hybrid retrieval.
5.  **Inspect reranker ordering.** If the chunk was retrieved but ranked
    too low, compare BGE cross-encoder scores and competing passages.
    Evaluate candidate count and timeout/fallback behavior rather than
    assuming the reranker is always correct.
6.  **Inspect context assembly.** If the correct chunk ranked well but
    was dropped, examine token budgets, truncation, deduplication, chunk
    ordering, and any optional compression. Keep MMR and compression
    disabled unless evaluation demonstrates a benefit without
    regressions on important exclusions and qualifiers.
7.  **Fix the narrowest responsible layer.** Possible fixes include
    structure-aware chunking, metadata repair/reindexing, query rewrite,
    hybrid retrieval, candidate-limit tuning, reranker evaluation, or
    context-budget changes. Choose based on the trace and test set, not
    intuition alone.
8.  **Add a regression test.** Assert that the required passage is
    retrieved and included, its citation points to the right source
    location, and the answer preserves relevant exclusions/qualifiers.

**Example:** If a policy paragraph says a benefit is covered "except in
the following circumstances," retrieving only the benefit sentence while
dropping the exception is a serious context-selection failure. The
answer must not be accepted merely because it cites the correct policy
document.

------------------------------------------------------------------------

## 5. How do you prove the problem is fixed? Do you only fix the issue, or improve the system too?

**Answer**

I do both, in that order: first establish a minimal fix for the reported
failure, then determine whether a systemic improvement is justified. I
prove the fix with a reproducible regression test and a broader
non-regression evaluation, not by checking one successful answer.

### Verification plan

1.  **Capture the failing case before the change.** Save the question,
    expected evidence, expected answer constraints, incorrect behavior,
    and trace/configuration versions in an approved test dataset.
2.  **Write explicit assertions.** Depending on the failure, assert
    expected chunk retrieval, correct source/version, inclusion in the
    model context, citation correctness, preservation of material
    qualifiers, correct structured response, or safe abstention.
3.  **Run the targeted regression.** Re-run the same case with the fix
    and verify the expected stage changed. A different answer alone is
    not proof.
4.  **Run the relevant evaluation slice.** For retrieval changes,
    measure recall@k and ranking quality on related cases. For
    generation changes, evaluate correctness, claim support, citation
    precision, and abstention behavior. For operational changes, test
    retries, timeouts, idempotency, latency, and cost.
5.  **Run the full regression suite.** Include normal questions,
    unsupported questions, conflicting/old policy versions, missing
    evidence, authorization boundaries, prompt injection, exclusions,
    and claims scenarios.
6.  **Compare before and after.** Record metrics, sample size,
    configuration, evaluation dataset version, and any regressions.
    Avoid relying only on an LLM-as-judge; use deterministic checks and
    insurance SME review for material policy answers.
7.  **Release safely.** Use a staged rollout/canary when feasible,
    monitor quality and operational signals, and define rollback
    criteria before release.

### Fix versus improve

-   **Fix:** make the smallest change that corrects the demonstrated
    root cause.
-   **Improve:** if multiple cases reveal the same systemic issue,
    improve the shared component---for example, chunking, retrieval
    filters, traceability, or validation.
-   **Do not overfit:** changing prompts or retrieval settings to pass
    one question can degrade other policy types. The full test suite is
    the guardrail.

The architecture calls for reviewed insurance scenarios, agent-level
contract tests, and full graph-path evaluation covering unsupported
questions, conflicting clauses, stale documents, cross-tenant attempts,
prompt injection, missing facts, dependency outages, timeouts, and
citation failures. Track correctness, retrieval recall, citation
precision, safe abstention, latency, and cost. Numeric release
thresholds should be agreed with the product/risk owners rather than
invented during an incident.

------------------------------------------------------------------------

## 6. How would you implement retries for only the failed node instead of restarting other nodes or the whole workflow?

**Answer**

I would use LangGraph's explicit node boundaries and typed
request-scoped state to isolate retryable work. Each node has a clear
input/output contract, a timeout, an error classification, and a bounded
retry policy. A transient failure should retry that node or its
idempotent operation, not re-run already successful unrelated work.

### Design

1.  **Persist or retain node state appropriately.** Keep validated
    outputs from completed nodes in the current graph state. If retrying
    across process crashes is required, configure an approved LangGraph
    checkpoint/persistence store with encryption, retention, access
    control, and replay rules. The architecture treats durable
    checkpoint persistence as a decision requiring approval; do not
    assume request-local memory survives a process restart.
2.  **Classify failures.** Retry transient timeouts, throttling, and
    selected 5xx failures with bounded exponential backoff and jitter.
    Do not retry invalid input, authorization denial, unsupported
    operations, source/version integrity failures, or deterministic
    business-rule rejection as if they were transient.
3.  **Retry the smallest safe unit.** If BGE reranking times out, retry
    reranking using the same validated candidate set. Do not rerun
    document ingestion or customer API calls that already succeeded. If
    a failed node's inputs depend on an invalid upstream result,
    repair/recompute the dependency deliberately rather than blindly
    retrying downstream.
4.  **Use a deadline and budget.** Enforce per-provider timeouts,
    maximum attempts, a request/job deadline, and an overall
    model-call/token budget. Stop retrying when the remaining budget
    cannot safely complete the workflow.
5.  **Preserve idempotent outputs.** Give each node execution an
    operation key derived from the workflow/request, node, input
    version/hash, and attempt-independent business operation identity.
    Record status and output so a repeated execution can detect
    completed work.
6.  **Handle exhaustion safely.** If a mandatory policy, customer,
    catalog, rules, or evidence dependency remains unavailable, return a
    safe unavailable result, clarify, abstain, or hand off. Skip an
    optional specialist only if the response contract remains valid
    without it.
7.  **Observe every attempt.** Record node name, attempt number, failure
    class, backoff, latency, and final outcome.

**Important nuance:** "Retry only the failed node" is safe only when its
inputs remain valid and its side effects are idempotent or otherwise
protected. Some failures require restarting a dependent subgraph, but
not the entire request. If checkpointing is not enabled, the service may
only be able to retry within the current process/request; durable
recovery needs an explicitly selected persistence design.

------------------------------------------------------------------------

## 7. If the failed node already called an external API before failing, how do you prevent duplicate processing during retries?

**Answer**

A timeout does not tell me whether the external system completed the
operation. I treat this as an **unknown outcome**, not proof that the
API call failed. I use idempotency and reconciliation rather than simply
issuing the same request again.

### Controls

1.  **Use an idempotency key for side-effecting operations.** Generate a
    stable key for the logical business action---not a new key for every
    retry---and send it to the external API if supported. The external
    service should persist the key and return the original result for
    duplicates.
2.  **Use a durable operation record.** Before invoking the API, record
    an operation ID, request hash, target system, idempotency key, and
    status such as `PENDING`. After a confirmed result, record
    `SUCCEEDED` plus the external reference. Use conditional state
    transitions to prevent concurrent workers from executing the same
    logical operation.
3.  **Separate read operations from side effects.** Retrying an
    authorized GET is usually safer than retrying a payment, claim
    submission, document publication, or other write. Still apply
    timeouts, rate limits, and the provider's retry contract.
4.  **On an ambiguous timeout, reconcile first.** Query the external
    system by idempotency key or business reference. If it succeeded,
    persist the result locally and continue. If it definitively did not
    happen, retry with the same idempotency key. If status cannot be
    established, stop automatic retries and send the operation for
    controlled reconciliation or human review.
5.  **Use transactional outbox/inbox patterns when appropriate.** For
    database state plus event publication, an outbox can avoid losing
    the event between committing state and publishing it. Consumers
    should deduplicate using a stable event/operation ID. This is not a
    substitute for idempotency at the external system.
6.  **Do not claim exactly-once side effects without support.** A queue
    or workflow engine alone cannot guarantee exactly-once execution
    across an arbitrary external API. The practical target is
    at-least-once delivery with idempotent handling, deduplication, and
    reconciliation.

### Example

A claim-submission node calls an external claims API, then times out
before LangGraph records success. On retry, it checks the operation
record and queries the claims API using the same idempotency key. If the
claim already exists, it records the existing claim reference and
continues. It does not create a second claim.

For ingestion, apply the same principle with stable
document/version/job/chunk IDs. Replaying a job should upsert the same
logical records, not create duplicate chunks or mix vectors from
incompatible index generations.

------------------------------------------------------------------------

## 8. If there are 500,000 SOPs in the vector database, how do you identify the type of claim?

**Answer**

I would not search all 500,000 SOPs and expect the LLM to discover the
claim type from a huge context. I separate **claim-type classification**
from **evidence retrieval**, and I use metadata and staged retrieval to
narrow the search before generation.

### Request-time flow

1.  **Normalize the user's request.** Extract the stated claim category,
    policy/product reference, jurisdiction, event date, and relevant
    identifiers where available. Do not infer missing authoritative
    facts.
2.  **Resolve claim type.** Use an approved deterministic taxonomy/rules
    mapping when structured claim type is already available from the
    claims system or is explicitly stated. If it is absent or ambiguous,
    use a validated classifier (rules-based or a separately evaluated
    model) to propose candidate classes with confidence. The classifier
    is a routing aid, not the authoritative claim record.
3.  **Clarify when necessary.** If several claim types remain plausible
    and the distinction changes the procedure, ask a targeted question
    or retrieve only the relevant high-level taxonomy/SOP index to
    disambiguate. Do not silently choose a low-confidence class.
4.  **Apply metadata pre-filters.** Filter retrieval using trusted
    access scope plus claim type/category, jurisdiction, product,
    document status, effective date, and active version. Missing or
    inconsistent security/applicability metadata must not broaden
    access.
5.  **Retrieve in stages.** Search a small candidate set in the filtered
    corpus using dense retrieval, with optional BM25 for exact SOP IDs,
    clause numbers, defined terms, and product names. Apply BGE
    reranking to the candidates, not to all 500,000 SOPs.
6.  **Validate the result.** Verify source status, version, effective
    date, and provenance. If evidence is weak, conflicting, or not
    applicable, ask for clarification or abstain.
7.  **Measure the classifier and retrieval separately.** Track
    claim-type precision/recall and confusion matrix, retrieval recall@k
    within the correct type, reranker quality, wrong-route rate,
    abstention/clarification rate, latency, and cost. Evaluate across
    rare claim categories and overlapping SOPs.

### Example

If the customer says, "My car was damaged in a flood," the system may
propose a motor/comprehensive-damage route, but it should confirm
relevant policy and jurisdiction context and retrieve applicable
approved SOPs. If the customer says only, "I need to file a claim," the
system should ask what type of claim or use authorized claims-system
context rather than guessing.

### Scale considerations

-   Store searchable metadata/payload indexes for the filters used at
    query time.
-   Maintain stable document/chunk IDs and versioned index generations.
-   Use bounded candidate counts and reranking; benchmark Qdrant
    indexing, filtering, latency, and recall on representative data.
-   If claim types map to separate collections or partitions, choose
    that based on measured isolation and performance needs. Metadata
    filtering is often simpler; separate collections are not
    automatically required.
-   Do not assume the architecture already defines a claim-type
    classifier or has benchmarked 500,000 SOPs. The design should
    explicitly select and evaluate the taxonomy, classifier, filter
    schema, and capacity targets.

**Interview summary:** "I classify or resolve the claim type first, then
retrieve from the authorized, applicable subset of SOPs. The LLM
explains retrieved evidence; it does not replace the claims system or
the classification evaluation."

------------------------------------------------------------------------

## 9. How did you manage document updates and deletes? Do you allow deletion or retain versions?

**Answer**

I separate **source-document lifecycle**, **version history**,
**search-index lifecycle**, and **legal/data-retention deletion**.
Updating a document should not overwrite evidence that previous answers
relied on, and a deleted or superseded document must not remain
retrievable just because its vectors still exist.

### Document update flow

1.  **Create an immutable version record.** Assign a stable document ID
    and a new version ID/hash; store source location, approval/status,
    product/jurisdiction, effective dates, and lineage in metadata.
    Preserve the approved source in the governed document store.
2.  **Ingest into staging.** Extract text/layout/tables/OCR, run quality
    checks, chunk with stable source offsets, generate embeddings with a
    recorded model/version, and write the new chunks/vectors into a
    non-active index generation.
3.  **Validate completeness and consistency.** Reconcile expected
    document/chunk IDs and versions between metadata and Qdrant. Verify
    filters, provenance, vector dimensions/model configuration, and
    document quality. Do not expose a partially indexed version.
4.  **Publish through the logical activation boundary.** The
    architecture does not assume a cross-store atomic transaction
    between MongoDB and Qdrant. Use one authoritative active-generation
    reference (proposed as a conditional metadata update, subject to
    platform review). Retrieval resolves that reference and filters to
    the selected generation.
5.  **Retire the prior version from normal retrieval.** Mark it
    superseded or otherwise not current for applicable queries. Preserve
    it for audit/history only under approved retention and access
    policies. A prior version may remain valid for a historical claim if
    its effective-date rules say it applied at the relevant time.
6.  **Rollback safely if activation fails.** Keep the prior valid
    generation active if staging or reconciliation fails. If a
    post-activation integrity problem appears, stop serving the affected
    generation and follow the documented rollback/reconciliation
    procedure. Conditional updates/version checks prevent an older
    ingestion job from replacing a newer approved generation.

### Delete flow

"Delete" must have an explicit meaning:

-   **Remove from search:** immediately mark the document/version as
    deleted, revoked, or non-searchable in authoritative metadata and
    ensure retrieval filters reject it. Propagate removal/tombstones to
    Qdrant and caches; do not rely on eventual vector cleanup alone for
    access control.
-   **Retain for audit/legal obligations:** if policy permits and
    retention is required, keep the source/version in a restricted
    archive with no normal retrieval access and a documented retention
    period.
-   **Permanently erase:** if a valid legal/privacy/retention
    requirement demands erasure, delete the source and derived artifacts
    (chunks, vectors, cached copies, and eligible replicas/backups
    according to the approved deletion policy) and record a minimal
    deletion audit event that does not recreate the sensitive content.
    Exact scope and backup treatment must follow legal and platform
    policy.

### Preventing stale answers

-   Every retrieval must check approval/status, effective dates,
    version, jurisdiction/product, ACL/tenant, and the selected index
    generation.
-   Cache entries must include version/generation and scope;
    invalidation events must be defined, and authorization must be
    checked at read time.
-   Reconcile MongoDB metadata and Qdrant payloads after
    updates/deletes. If they disagree, do not activate or continue
    serving an unsafe generation.
-   Maintain provenance in answers and audit records so an investigation
    can identify which source version was used.
-   Historical validity is not the same as current searchability: a
    superseded version might be relevant to a claim from an earlier
    coverage period, but it must only be retrieved under an explicit
    historical-date rule and authorized workflow.

**My recommendation:** Support versioning and soft deactivation from
ordinary search as the normal update/revocation mechanism, with
controlled permanent deletion for legal/privacy requirements. Do not
expose an unrestricted "delete forever" operation without authorization,
audit, retention checks, and cleanup of derived index/cache data.

------------------------------------------------------------------------

## Quick interview recap

-   **Wrong answer:** trace the earliest divergence across retrieval,
    reranking, context, prompt, generation, and validation.
-   **Logging:** structured OpenTelemetry spans/logs with request IDs,
    evidence IDs, model/prompt/index versions, metrics, and safe
    redaction; LangSmith/Ragas are not prerequisites.
-   **Retrieved chunks:** inspect stable chunk IDs, source offsets,
    ranks/scores, filters, and whether each chunk entered the final
    context.
-   **Wrong paragraph:** inspect chunk boundaries, metadata, candidate
    ranks, reranker, and context truncation.
-   **Proof of fix:** targeted regression plus broader non-regression
    evaluation and measured before/after results.
-   **Retries:** retry the smallest safe, transiently failed node within
    deadlines and budgets; preserve valid prior outputs.
-   **Duplicate side effects:** stable idempotency keys, durable
    operation status, reconciliation, and deduplication; do not assume
    exactly-once external effects.
-   **500,000 SOPs:** resolve/classify claim type, apply trusted
    metadata filters, retrieve bounded candidates, rerank, and validate.
-   **Updates/deletes:** immutable versions, staging, reconciliation,
    controlled activation, retrieval deactivation, and policy-governed
    permanent erasure.

## Architecture decisions to confirm before claiming implementation

The architecture document itself marks some details as open decisions.
Before describing them as implemented in an interview, confirm the
actual selected configuration for:

-   Durable LangGraph checkpoint store and replay/retention controls.
-   Initial ingestion queue/event mechanism and ownership of retries,
    dead-letter handling, and replay.
-   Claim-type taxonomy/classifier and its measured evaluation results.
-   Claim-to-evidence support validation beyond citation membership.
-   Exact SLOs, capacity targets, and tested performance at the intended
    SOP volume.
-   Approved document retention, legal deletion, archive, and
    backup-erasure policies.
-   Model/provider IDs, versions, retry policies, and any distinct
    deployment decisions.

A strong production answer is explicit about both the control design and
what has actually been implemented and verified.
