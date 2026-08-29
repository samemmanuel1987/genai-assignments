# Phase-Wise Implementation Plan

## Goal

Deliver a multi-tenant AI Test Case Generator that ingests QA artifacts and connected-tool data, retrieves project-scoped evidence through hybrid RAG, and produces reviewable manual and API test cases.

## Phase 0: Project Foundation

**Outcome:** A runnable TypeScript monorepo with local development services.

### Work

- Create npm workspaces for `apps/web`, `apps/api`, `apps/worker`, `packages/shared`, `packages/connectors`, and `packages/rag`.
- Configure TypeScript, ESLint, formatting, environment validation, test runners, and shared API contracts.
- Add local Docker Compose services for MongoDB, Redis, and object-storage emulation.
- Add CI for install, lint, type checking, unit tests, and build.

### Deliverables

- Monorepo structure and root scripts.
- React application shell.
- Express API health endpoint.
- Background worker startup.
- Shared validation and error-response conventions.

### Exit Criteria

- Developers can start the web app, API, worker, MongoDB, Redis, and object storage locally with one documented command.
- CI passes lint, type checks, tests, and builds.

## Phase 1: Identity, Organizations, and Access Control

**Outcome:** Secure, isolated workspaces for QA, developer, and product users.

### Work

- Implement organizations, memberships, projects, and role-based access control.
- Implement email/password registration, login, refresh tokens, and logout.
- Implement OAuth/OIDC SSO adapter and provider configuration.
- Add authorization middleware and tenant-scoped repository helpers.
- Encrypt integration credentials and write audit events for security-sensitive actions.

### API

- `POST /api/v1/auth/register`
- `POST /api/v1/auth/login`
- `POST /api/v1/auth/refresh`
- `POST /api/v1/auth/logout`
- `GET /api/v1/auth/oauth/:provider/start`
- `GET /api/v1/auth/oauth/:provider/callback`
- Organization membership endpoints.

### Exit Criteria

- A user can create an organization, invite a member, and create a project.
- A user from one organization cannot read, update, or search another organization's records.
- Audit events capture login, role, and credential changes.

## Phase 2: Project and Source Management

**Outcome:** Users can manage projects and upload source artifacts.

### Work

- Build project CRUD and project-member access.
- Add source catalog, source versions, processing status, and source metadata.
- Support signed uploads for BRD/user stories, UI specifications, technical documentation, release notes, Excel, PDF, transcripts, and OpenAPI/Swagger files.
- Support audio/video recording uploads and transcript uploads.
- Store originals in object storage; store metadata and references in MongoDB.
- Build source-management screens in React.

### API

- `GET, POST /api/v1/projects`
- `GET, PATCH, DELETE /api/v1/projects/:projectId`
- `GET, POST /api/v1/projects/:projectId/sources`
- `POST /api/v1/projects/:projectId/uploads/presign`
- `GET, DELETE /api/v1/projects/:projectId/sources/:sourceId`

### Exit Criteria

- An authorized user can upload an artifact and see its processing status.
- Files remain isolated by organization and project.
- Invalid file types and oversized files are rejected.

## Phase 3: Asynchronous Ingestion Pipeline

**Outcome:** Uploaded artifacts become searchable, cited knowledge.

### Work

- Introduce Redis-backed queues, worker processors, retry policies, and dead-letter handling.
- Extract text and structured content from PDF, DOCX, XLSX, Markdown, text, and OpenAPI/Swagger inputs.
- Transcribe supported audio/video recordings asynchronously.
- Normalize source metadata, split content into traceable chunks, and retain source locations.
- Generate Mistral embeddings and upsert chunks to MongoDB Atlas Vector Search.
- Add idempotency, progress events, manual re-ingestion, and operational status screens.

### API

- `POST /api/v1/projects/:projectId/sources/:sourceId/ingest`
- `GET /api/v1/projects/:projectId/ingestion-jobs`
- `GET /api/v1/ingestion-jobs/:jobId`
- `GET /api/v1/events`

### Exit Criteria

- A valid uploaded source reaches `indexed` state through the worker queue.
- A retry does not duplicate chunks or embeddings.
- Each chunk links back to its original source and excerpt location.

## Phase 4: Core RAG Retrieval

**Outcome:** The system retrieves the best project-scoped source evidence for a query.

### Work

- Create Atlas Search indexes for BM25 text search and vector similarity search.
- Implement query normalization, abbreviation expansion, and synonym expansion.
- Run hybrid BM25 and vector search filtered by organization and project.
- Merge and rerank candidates, remove duplicates, summarize selected context, and preserve citations.
- Build retrieval preview for users and quality evaluation datasets for the team.

### API

- `POST /api/v1/projects/:projectId/retrieval/search`
- `POST /api/v1/projects/:projectId/retrieval/preview`

### Exit Criteria

- Search returns only content from the current project and organization.
- Each retrieval result includes relevance data and a source citation.
- Evaluation tests establish an agreed baseline for retrieval quality.

## Phase 5: Test Case Generation and Confidence

**Outcome:** Users can generate structured manual and API test-case drafts with evidence.

### Work

- Define validated schemas for manual and API test cases.
- Build a provider-neutral LLM gateway: OpenAI primary, Groq fallback, Anthropic second fallback.
- Compose prompts from user intent, retrieval context, project settings, and output schema.
- Validate model output, capture provider usage/cost, and persist generation records.
- Calculate confidence from retrieval relevance, evidence coverage, and output validation.
- Build the generation workspace with input controls, cited evidence, confidence, and draft results.

### API

- `POST /api/v1/projects/:projectId/generations`
- `GET /api/v1/projects/:projectId/generations`
- `GET /api/v1/generations/:generationId`
- `POST /api/v1/generations/:generationId/regenerate`

### Exit Criteria

- A user can generate manual or API test-case drafts from indexed project sources.
- All generated cases include citations, a confidence score, and a `draft` state.
- Provider fallback works on configured retryable provider failures.

## Phase 6: Human Review and Traceability

**Outcome:** Reviewers can control publication quality and explain every test case.

### Work

- Implement draft, approved, rejected, and regeneration-requested states.
- Support edits, reviewer assignments, comments, approve/reject actions, and regeneration with comments.
- Preserve immutable revision history for generated and manually edited content.
- Associate test cases with source excerpts, Jira/ADO work items, and defects.
- Build a review queue and a test-case detail view.

### API

- `GET /api/v1/projects/:projectId/test-cases`
- `GET, PATCH /api/v1/test-cases/:testCaseId`
- `POST /api/v1/test-cases/:testCaseId/approve`
- `POST /api/v1/test-cases/:testCaseId/reject`
- `POST /api/v1/test-cases/:testCaseId/assignments`
- `POST /api/v1/test-cases/:testCaseId/comments`
- `GET /api/v1/test-cases/:testCaseId/traceability`

### Exit Criteria

- No test case can be exported or synchronized before approval.
- Reviewers can see evidence, edit content, add comments, request regeneration, and inspect revisions.
- Traceability identifies the exact evidence supporting each test case.

## Phase 7: External Integrations and Publishing

**Outcome:** Teams can ingest and publish through their established tools.

### Work

- Create a common connector interface with credential, sync, webhook, transform, and retry contracts.
- Implement connectors for Jira, Azure DevOps, Confluence/wiki, Git repositories, Swagger API sources, and defect databases.
- Implement publishing adapters for TestRail, Xray, and Zephyr.
- Add CSV, Excel, and PDF exports.
- Implement scheduled synchronization, provider-supported webhooks, export jobs, receipts, remote IDs, and bidirectional work-item/defect links.

### API

- `GET, POST /api/v1/projects/:projectId/integrations`
- `PATCH, DELETE /api/v1/projects/:projectId/integrations/:integrationId`
- `POST /api/v1/projects/:projectId/integrations/:integrationId/sync`
- `POST /api/v1/webhooks/:connector`
- `POST /api/v1/projects/:projectId/exports`
- `GET /api/v1/exports/:exportId`

### Exit Criteria

- At least one configured connector successfully imports data on demand and on schedule.
- Approved cases export to CSV/Excel/PDF and publish to configured test-management tools.
- Webhook signatures are verified and sync/export operations are idempotent.

## Phase 8: Production Readiness and Release

**Outcome:** A secure, observable deployment on a managed cloud container platform.

### Work

- Containerize web, API, and worker applications.
- Provision managed MongoDB/Atlas, Redis, object storage, secret management, and container hosting.
- Add rate limiting, secure headers, file-scanning integration point, encryption, data retention, and deletion workflows.
- Add OpenTelemetry traces, structured logs, metrics, error tracking, audits, and LLM cost dashboards/alerts.
- Add environment-specific CI/CD, database migrations, smoke tests, backup/recovery checks, and release runbooks.

### Exit Criteria

- Production deployment is repeatable through CI/CD.
- Dashboards show API errors, queue health, ingestion latency, retrieval/generation performance, and provider cost.
- Security, recovery, and operational acceptance checks are complete.

## Deferred Enhancements

- Real-time voice conversation.
- Automated test-code generation for Playwright or Cypress.
- SIEM integration.
- Self-hosted customer deployments.
- Automated approval based only on confidence score.

## Recommended Delivery Order

Deliver Phases 0 through 5 as the first usable release: upload sources, index them, retrieve evidence, and generate cited test-case drafts. Add Phase 6 before broad production adoption so every output has a controlled approval path. Complete Phases 7 and 8 for enterprise integration and production rollout.
