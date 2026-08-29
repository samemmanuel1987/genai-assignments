## Plan: AI Test Case Generator

Build a multi-tenant React and Node.js/TypeScript application that ingests QA knowledge from files and external tools, stores metadata and Mistral embeddings in MongoDB Atlas Vector Search, retrieves project-scoped evidence with a hybrid RAG pipeline, and generates reviewable manual and API test cases. Run the API, worker, and scheduled jobs as independently scalable managed cloud containers.

### Architecture

React SPA -> Express REST API/BFF -> MongoDB Atlas, object storage, Redis-backed queue, external connectors, and LLM gateway. A worker processes ingestion and export jobs. All database documents, object keys, queue payloads, search filters, and audit events include organizationId and projectId. Backend services are split by domain rather than by provider: identity, organizations/projects, sources/integrations, ingestion, retrieval, generation, reviews, exports, and observability.

Use an LLM gateway adapter with OpenAI as primary, Groq as first fallback, and Anthropic as second fallback. Mistral creates document embeddings. Store chunks with source metadata and Atlas vector indexes; use Atlas Search BM25 plus vector similarity, merge scores, rerank candidates, deduplicate source passages, summarize context, and provide cited evidence to structured-output generation.

### Decisions

1. Primary users: all roles equally, with role-based access.
2. Authentication: email/password JWT sessions and enterprise OAuth/OIDC SSO.
3. Tenant model: multiple organizations with strictly isolated data.
4. Sources: file uploads plus direct integrations, configured manually by URL/API credentials.
5. Ingestion timing: manual runs, scheduled syncs, and event-driven webhooks where supported.
6. Recordings: audio/video upload with transcription and direct transcript upload.
7. Storage: cloud object storage for originals; MongoDB for metadata and vectors.
8. Ingestion execution: separate background workers through a job queue.
9. Test output: structured manual and API test cases.
10. LLM strategy: platform-managed fallback chain of OpenAI, Groq, then Anthropic.
11. Review workflow: generated cases begin as drafts and require per-item approval.
12. Review actions: approve, reject, edit, regenerate with comments, and assign another reviewer.
13. Confidence: composite relevance, coverage, and validation score with organization-configurable thresholds.
14. Export: CSV, Excel, PDF, and direct TestRail, Xray, and Zephyr publishing.
15. Traceability: cited source excerpts with bidirectional links to Jira/ADO work items and defects.
16. Retrieval scope: all indexed sources in the current project.
17. Voice chat: deferred; retain an extension path through the conversation/generation service.
18. Deployment: managed cloud container platform.
19. Observability: logs, metrics, traces, audit history, and LLM provider cost tracking.
20. API: REST API/BFF for React; add WebSockets or server-sent events for ingestion and export job status.

### Steps

1. Bootstrap a TypeScript monorepo with npm workspaces: apps/web for React, apps/api for Express, apps/worker for background work, and packages/shared for contracts and validation. Add Docker Compose only for local MongoDB, Redis, and object-storage emulation.
2. Implement identity and tenancy first: organizations, membership roles, local credentials, OAuth/OIDC identities, JWT refresh flow, authorization middleware, tenant-safe query helpers, encrypted integration secrets, and audit events. All remaining services depend on this.
3. Add project and source management. Support uploads of BRD/user-story files, UI specifications, technical docs, release notes, Excel, PDFs, transcripts, OpenAPI/Swagger files, and recordings. Build connector interfaces and implementations for Jira, Azure DevOps, TestRail, Xray, Zephyr, Confluence/wiki, Git repositories, Swagger endpoints, and defect databases.
4. Build the asynchronous ingestion pipeline. Persist source versions and objects, enqueue work, extract text and structured records, transcribe recordings, normalize metadata, chunk content with source pointers, create Mistral embeddings, and upsert searchable chunks. Add idempotency keys, retries, dead-letter handling, progress reporting, scheduled syncs, and signed webhook verification.
5. Configure MongoDB Atlas Search and Vector Search indexes for tenant/project filters, text search, and vector similarity. Implement query normalization, acronym and synonym expansion, hybrid candidate retrieval, reranking, deduplication, context summarization, and cited-context assembly.
6. Implement a provider-neutral LLM gateway with structured schemas for manual and API test cases. Route calls through OpenAI, then Groq, then Anthropic on eligible failures; record provider, token use, latency, and cost. Validate generated output and calculate a composite confidence score from retrieval relevance, evidence coverage, and schema/quality checks.
7. Create the generation and review lifecycle: create a draft generation job, display sources and confidence, edit the draft, approve/reject it, comment and regenerate, assign a reviewer, retain immutable revision history, and link cases to Jira/ADO items and defects.
8. Add export/publishing adapters for CSV, Excel, PDF, TestRail, Xray, and Zephyr. Use asynchronous export jobs, save export receipts and remote IDs, and synchronize bidirectional links where provider APIs allow it.
9. Build a responsive operational React UI: organization/project selection, source catalog and ingestion status, integration management, generation workspace, evidence side panel, test-case review queue, export history, and administrator settings. Subscribe to job-progress events.
10. Add production readiness: rate limits, file-type and malware scanning integration point, secret management, encryption in transit/at rest, retention/deletion workflows, OpenTelemetry tracing, logs/metrics/dashboards, error tracking, audit exploration, provider cost alerts, and CI/CD to a managed cloud container platform.

### API Endpoints

Identity and tenant management:
- POST /api/v1/auth/register
- POST /api/v1/auth/login
- POST /api/v1/auth/refresh
- POST /api/v1/auth/logout
- GET /api/v1/auth/oauth/:provider/start
- GET /api/v1/auth/oauth/:provider/callback
- GET /api/v1/organizations
- POST /api/v1/organizations
- GET /api/v1/organizations/:organizationId/members
- POST /api/v1/organizations/:organizationId/members
- PATCH /api/v1/organizations/:organizationId/members/:memberId
- DELETE /api/v1/organizations/:organizationId/members/:memberId

Projects and source ingestion:
- GET, POST /api/v1/projects
- GET, PATCH, DELETE /api/v1/projects/:projectId
- GET, POST /api/v1/projects/:projectId/sources
- POST /api/v1/projects/:projectId/uploads/presign
- POST /api/v1/projects/:projectId/sources/:sourceId/ingest
- GET /api/v1/projects/:projectId/sources/:sourceId
- DELETE /api/v1/projects/:projectId/sources/:sourceId
- GET /api/v1/projects/:projectId/ingestion-jobs
- GET /api/v1/ingestion-jobs/:jobId
- POST /api/v1/webhooks/:connector

Integrations and retrieval:
- GET, POST /api/v1/projects/:projectId/integrations
- PATCH, DELETE /api/v1/projects/:projectId/integrations/:integrationId
- POST /api/v1/projects/:projectId/integrations/:integrationId/sync
- POST /api/v1/projects/:projectId/retrieval/search
- POST /api/v1/projects/:projectId/retrieval/preview

Generation, review, traceability, and export:
- POST /api/v1/projects/:projectId/generations
- GET /api/v1/projects/:projectId/generations
- GET /api/v1/generations/:generationId
- POST /api/v1/generations/:generationId/regenerate
- GET /api/v1/projects/:projectId/test-cases
- GET, PATCH /api/v1/test-cases/:testCaseId
- POST /api/v1/test-cases/:testCaseId/approve
- POST /api/v1/test-cases/:testCaseId/reject
- POST /api/v1/test-cases/:testCaseId/assignments
- POST /api/v1/test-cases/:testCaseId/comments
- GET /api/v1/test-cases/:testCaseId/traceability
- POST /api/v1/projects/:projectId/exports
- GET /api/v1/exports/:exportId
- GET /api/v1/events for authenticated job-status streaming

### Proposed Folder Structure

```text
TestCase_Generator/
  apps/
    web/src/{app,features,components,lib}
    api/src/{config,modules,middleware,queues,realtime,observability}
    worker/src/{jobs,processors,schedulers}
  packages/
    shared/src/{contracts,schemas,types,constants}
    connectors/src/{base,jira,azure-devops,testrail,xray,zephyr,confluence,git,openapi,defects}
    rag/src/{extraction,transcription,chunking,embeddings,retrieval,reranking,prompts,providers}
  infra/{docker,cloud,monitoring}
  plan.md
```

### Relevant Files to Create

- docs/architecture.md - durable architecture document based on this plan.
- apps/api/src/app.ts - Express composition root and REST/BFF routes.
- apps/worker/src/index.ts - queue worker startup.
- packages/rag/src/retrieval/hybrid-retriever.ts - hybrid RAG orchestration.
- packages/shared/src/schemas/test-case.ts - validated manual/API test-case contracts.

### Verification

1. Unit-test tenant filtering, role checks, connector transforms, chunking, retrieval scoring, provider fallback, output validation, confidence calculation, and review state transitions.
2. Run integration tests with MongoDB, Redis, object-storage emulation, and mocked provider/connector APIs; prove one organization cannot read another organization's records or vectors.
3. Run contract tests for every connector and export adapter, including idempotent retry behavior and webhook-signature rejection.
4. Run end-to-end browser tests covering upload to generated draft, reviewer edit/approval, traceability inspection, job progress, and TestRail/Xray/Zephyr export.
5. Perform load tests for concurrent ingestion and generation; verify queue backpressure, rate limiting, retry behavior, provider fallback, and cost/audit telemetry.

### Scope Boundaries

Included: multi-tenant RBAC, local and SSO authentication, all named upload and integration categories, hybrid RAG, manual/API test generation, evidence citations, confidence, review workflow, exports/publishing, observability, and cloud-container deployment.

Excluded from the first release: real-time voice conversation, automated-test code generation, SIEM integration, self-hosted customer deployments, and fully automated approval based on confidence. Voice chat and automated test-code generation should be introduced as later modules without changing core retrieval or test-case contracts.
