# Implementation Plan: Node.js/React Validation Agent with Component-Based Architecture

**Stack:** Node.js (Express) + MongoDB + Pinecone/Vector DB + React.js  
**Approach:** API-First → DB-Second → Web-Third  
**Design Principle:** Small, testable components; single responsibility; stateless where possible

---

## Overview

Break the validation system into **independently testable components** following strict API-first design. Each class has one job. Express middleware handles auth → validation endpoints → orchestrator → vector retrieval → LLM → response formatting. No complex architecture; standard patterns only.

---

## Phase 1: API Layer (Core Validation Engine)

### 1.1 Auth & Request Handling

**Class: `ApiKeyMiddleware`**
- Validates API keys from request headers
- Rejects unauthorized requests before reaching handlers
- Returns 401 with clear error messages

**Class: `RequestValidator`**
- Validates POST /validate payload schema
- Ensures story object and CR ID array are present
- Returns 400 with validation error details

**Class: `ErrorHandler`**
- Centralized error response format
- Maps internal errors to HTTP status codes
- Includes request ID for tracing
- Logs errors with context

**Class: `RateLimiter`**
- Throttles requests by API key
- Prevents abuse; tracks quota per caller
- Returns 429 when limit exceeded

---

### 1.2 Validation Orchestrator (Core Engine)

**Class: `ValidationOrchestrator`**
- Orchestrates the complete 7-phase validation flow
- Accepts: story object (title, description, acceptance criteria), selected CR IDs
- Returns: structured `ValidationResult` object (readiness_score, risk_level, gaps, evidence, etc.)
- **Method: `validateStory(story, crIds) → Promise<ValidationResult>`**
  - Phase 1: Embed story at runtime
  - Phase 2: Retrieve CR context from Vector DB
  - Phase 3: Expand semantic context
  - Phase 4: Execute LLM validation prompts
  - Phase 5: Detect gaps and compute scores
  - Phase 6: Generate HTML report reference
  - Phase 7: Return structured result
- Stateless; fully testable with mocked dependencies
- No state retention between calls

---

### 1.3 Story & CR Retrieval

**Class: `StoryRetriever`**
- Fetches user story from Azure DevOps (external API call)
- Returns: story object with id, title, description, acceptance_criteria
- **Method: `getStoryById(storyId) → Promise<Story>`**
- Handles API errors gracefully

**Class: `CRVectorRetriever`**
- Performs semantic search in Vector DB
- Accepts: embedding vector, CR ID list, top-K count
- Returns: ranked list of CR text chunks with relevance scores
- **Methods:**
  - `retrieveByCRIds(crIds, topK=5) → Promise<CRChunk[]>`
  - `expandContext(chunks) → Promise<ExpandedContext>` (adds related tech docs, NFR references)
  - `filterByRelevance(chunks, threshold=0.7) → CRChunk[]`
- Each chunk carries metadata: (docId, version, sectionId, projectId, sourceType, checksum)

---

### 1.4 Embedding & LLM Services

**Class: `EmbeddingService`**
- Embeds story text at runtime using external API (OpenAI, Azure OpenAI, etc.)
- **Method: `embedText(text) → Promise<number[]>`**
- Returns: fixed-size vector (e.g., 1536 dims for OpenAI)
- Caches embeddings briefly to avoid re-embedding identical text in same request
- Simple, no-frills; tested with mock API

**Class: `PromptBuilder`**
- Constructs phase-specific structured prompts for LLM
- No monolithic prompt; each phase builds its own prompt template
- **Methods:**
  - `buildFunctionalAlignmentPrompt(story, crContext) → string`
  - `buildACGapPrompt(story, acceptanceCriteria, crContext) → string`
  - `buildBusinessRulePrompt(story, crContext) → string`
  - `buildNFRValidationPrompt(story, crContext) → string`
  - `buildAmbiguityDetectionPrompt(story, crContext) → string`
  - `buildRiskClassificationPrompt(story, gaps) → string`
  - `buildReadinessScoringPrompt(story, allAnalysis) → string`
  - `buildEvidenceEnforcementPrompt(story, crContext) → string`
- Each returns: template + context variables (story, CR chunks, previous results)
- Versioned for reproducibility and audit

**Class: `LLMValidator`**
- Executes structured validation prompts against LLM (OpenAI API, etc.)
- Parses LLM responses into structured objects
- **Methods (one per validation phase):**
  - `validateFunctionalAlignment(story, crContext) → Promise<AlignmentResult>`
    - Returns: alignment_score, coverage_gaps, missing_features
  - `detectAcceptanceCriteriaGaps(story, crContext) → Promise<ACGapResult>`
    - Returns: gaps (missing AC from CR), covered_ac (explicit AC from CR)
  - `validateBusinessRules(story, crContext) → Promise<BusinessRuleResult>`
    - Returns: rule_gaps, conflicting_rules, missing_context
  - `validateNFR(story, crContext) → Promise<NFRResult>`
    - Returns: implied_NFR, missing_NFR (performance, security, scalability, etc.)
  - `detectAmbiguities(story, crContext) → Promise<AmbiguityResult>`
    - Returns: ambiguous_phrases, unclear_acceptance_criteria
  - `classifyRisks(story, gaps) → Promise<RiskResult>`
    - Returns: technical_risks, business_risks, schedule_risks with severity
  - `scoreReadiness(story, allAnalysis) → Promise<ReadinessScoreResult>`
    - Returns: dimension_scores, overall_readiness_score, justification
- Each method returns structured object (JSON-serializable)
- Includes cite-with-evidence requirement: LLM must reference specific CR section IDs
- Handles LLM errors (timeouts, API failures) with fallback logic

---

### 1.5 Scoring Engine

**Class: `ScoringCalculator`**
- Calculates weighted readiness score from dimension-level scores
- **Dimensions & weights:**
  - Functional Alignment: 25%
  - Acceptance Criteria: 15%
  - Business Rules: 15%
  - NFR (Non-Functional Requirements): 15%
  - Ambiguity: 10% (lower is better)
  - Risk: 10% (lower is better)
  - Traceability: 10%
- **Method: `calculateReadinessScore(dimensionScores: Map<string, number>) → ReadinessScore`**
  - Returns: { overall_score: 0-100, weighted_breakdown: {dimension: score}, rationale: string }
  - All dimensions on 0-100 scale
  - Negative dimensions (ambiguity, risk) inverted before weighting

**Class: `RiskClassifier`**
- Maps readiness scores to risk bands
- **Method: `classifyRisk(readinessScore: number) → RiskBand`**
  - LOW: 80–100
  - MEDIUM: 60–79
  - HIGH: <60
- Returns: risk_band, justification

---

### 1.6 Report Generation

**Class: `HTMLReportGenerator`**
- Builds HTML report from structured validation result
- Output persisted to file storage (local, S3, or Azure Blob Storage)
- **Method: `generateReport(validationResult, story, crContext) → Promise<ReportPath>`**
- Sections:
  1. Executive Summary (readiness score, risk band, key findings)
  2. Alignment Summary (functional coverage percentage)
  3. Functional Gaps (list of missing features)
  4. Acceptance Criteria Gaps (missing or unclear AC)
  5. Business Rule Gaps
  6. NFR Gaps (performance, security, scalability, etc.)
  7. Risk Indicators (technical, business, schedule)
  8. Suggested Improvements
  9. Traceability Matrix (story ID → CR sections → evidence)
  10. Evidence Citations (links to specific CR chunks)
  11. Readiness Score Dashboard (dimension breakdown, visualization)
- File naming: `{storyId}_{validationId}_{timestamp}.html`
- Self-contained HTML; no external CDN dependencies (embed CSS, minimal JS)
- Accessible and printable

**Class: `JSONFormatter`**
- Structures complete validation result as JSON for API response
- **Method: `formatAsJSON(validationResult) → object`**
- Returns:
  ```json
  {
    "validation_id": "...",
    "story_id": "...",
    "readiness_score": 75,
    "risk_level": "MEDIUM",
    "alignment_summary": { ... },
    "gaps": { functional: [...], ac: [...], business_rules: [...], nfr: [...] },
    "risks": [ ... ],
    "evidence": [ ... ],
    "traceability": [ ... ],
    "html_report_path": "/reports/...",
    "created_at": "2026-02-25T..."
  }
  ```

---

### 1.7 API Endpoints

**Class: `ValidationController`**

**Endpoint: `POST /api/v1/validate`**
- **Payload:**
  ```json
  {
    "story_id": "US-12345",
    "story": {
      "title": "User can login with SSO",
      "description": "...",
      "acceptance_criteria": ["...", "..."]
    },
    "cr_ids": ["CR-001", "CR-002"]
  }
  ```
- **Handler:**
  1. Validate request schema
  2. Call `ValidationOrchestrator.validateStory(story, crIds)`
  3. Persist result to DB (save to `validations` collection)
  4. Return 202 (Accepted) or 200 (if sync) with:
     ```json
     {
       "validation_id": "VAL-uuid",
       "status": "completed",
       "readiness_score": 75,
       "risk_level": "MEDIUM",
       "html_report_path": "/reports/US-12345_VAL-uuid_timestamp.html",
       "created_at": "..."
     }
     ```
- **Error codes:** 400 (bad request), 401 (unauthorized), 429 (rate limited), 500 (server error)

**Endpoint: `GET /api/v1/validation/{validation_id}`**
- **Handler:**
  1. Retrieve validation from DB
  2. Return full result (same JSON structure as above)
- **Uses:** `ValidationRepository.findById(id)`

**Endpoint: `GET /api/v1/health`**
- **Handler:** Returns 200 with { "status": "ok", "timestamp": "..." }
- Quick dependency check (DB connectivity, etc.) optional

---

## Phase 2: Database Layer (Data Persistence)

### 2.1 MongoDB Collections & Schemas

**Collection: `validations`**
- Document structure:
  ```javascript
  {
    _id: ObjectId,
    validation_id: "VAL-uuid", // unique
    story_id: "US-12345",
    project_id: "PROJ-001",
    readiness_score: 75,
    risk_level: "MEDIUM",
    prompt_version: "1.0",
    model_version: "gpt-4",
    cr_versions_used: ["CR-001:v2", "CR-002:v1"],
    html_path: "/reports/US-12345_VAL-uuid_timestamp.html",
    json_result: { ... }, // full validation result (denormalized)
    created_at: ISODate(),
    updated_at: ISODate()
  }
  ```
- **Indexes:**
  - `story_id, created_at` (most queries)
  - `validation_id` (unique lookup)
  - `project_id` (for project-level reporting)

**Collection: `validation_scores`**
- Document structure:
  ```javascript
  {
    _id: ObjectId,
    validation_id: "VAL-uuid",
    dimension: "functional_alignment", // enum: functional_alignment, ac, business_rules, nfr, ambiguity, risk, traceability
    score: 85,
    rationale: "Story covers 85% of CR requirements",
    evidence: ["CR-001:section-2.1", "CR-002:section-1.3"]
  }
  ```
- **Indexes:** `validation_id` (group by validation)

**Collection: `validation_comments`**
- Document structure:
  ```javascript
  {
    _id: ObjectId,
    validation_id: "VAL-uuid",
    user_id: "user-456",
    comment_text: "Reviewed; needs clarification on performance targets",
    timestamp: ISODate(),
    resolved: false
  }
  ```
- **Indexes:** `validation_id`

**Collection: `audit_logs`**
- Document structure:
  ```javascript
  {
    _id: ObjectId,
    event_type: "validation_created|validation_updated|comment_added|report_generated",
    validation_id: "VAL-uuid",
    actor: "system|user-456",
    metadata: { ... },
    timestamp: ISODate()
  }
  ```
- **Indexes:** `(validation_id, timestamp)` (audit trail queries)

---

### 2.2 Mongoose Schema Classes

**Class: `ValidationSchema`**
- Defines validation document structure
- Validation rules: story_id required, readiness_score 0-100, risk_level enum
- Methods: `statics.findByStoryId()`, `statics.findRecent(limit)`, `instance.addComment()`

**Class: `ScoreSchema`**
- Defines score document structure
- Validation: dimension is enum, score 0-100

**Class: `CommentSchema`**
- Defines comment structure
- Validation: timestamp auto-set, resolved default false

**Class: `AuditLogSchema`**
- Defines audit log structure
- Validation: event_type is enum

---

### 2.3 Repository Pattern (Data Access Abstraction)

**Class: `ValidationRepository`**
- Abstracts all validation read/write logic
- Decouples business logic from MongoDB details
- **Methods:**
  - `save(validationData) → Promise<ValidationDocument>`
  - `findById(validationId) → Promise<ValidationDocument>`
  - `findByStoryId(storyId) → Promise<ValidationDocument[]>`
  - `updateScore(validationId, dimension, score) → Promise<void>`
  - `addComment(validationId, userId, commentText) → Promise<void>`
  - `findRecent(projectId, limit=10) → Promise<ValidationDocument[]>`
- All methods return promises
- Error handling: throws `DatabaseError` on failure

**Class: `AuditRepository`**
- Appends audit events (no updates, only inserts)
- **Method: `log(eventType, validationId, actor, metadata) → Promise<void>`**
- Ensures immutability of audit trail

---

### 2.4 Vector DB Layer (Pinecone / Weaviate / Milvus)

**Class: `VectorDBConnector`**
- Abstracts vector DB implementation (Pinecone, Weaviate, Milvus, etc.)
- **Methods:**
  - `indexDocument(documentId, vector, metadata) → Promise<void>`
    - metadata includes: version, sectionId, projectId, sourceType (CR|TechDoc|NFR|Defect|Release), checksum
  - `search(queryVector, topK=5, filters={}) → Promise<SearchResult[]>`
    - Filters: by CR ID, by sourceType, by projectId
    - Returns: chunks with relevance scores and metadata
  - `deleteByDocId(documentId) → Promise<void>`
  - `updateDocument(documentId, newVector, newMetadata) → Promise<void>`
- Handles connection pooling, retries, timeouts

**Class: `CRIngestionPipeline`**
- Batch pre-embedding of CR documents
- Runs as separate scheduled job (not on API pathway)
- **Methods:**
  - `ingestCRs(crDocuments) → Promise<IngestionReport>`
    - Calculates checksum for each document
    - Skips re-embedding if checksum matches indexed version
    - Updates Vector DB with new/changed docs
  - `ingestTechDocs(techDocuments) → Promise<IngestionReport>`
  - `ingestNFRDocs(nfrDocuments) → Promise<IngestionReport>`
  - `ingestReleaseNotes(releaseNotes) → Promise<IngestionReport>`
- Stateless; can run independently from API service
- Logs ingestion events to audit trail

---

## Phase 3: Web Layer (React Dashboard)

### 3.1 React Components

**Component: `ValidationForm`**
- Renders form to submit story and select CRs
- **State:**
  - `storyInput` (title, description, AC textarea)
  - `selectedCRs` (multi-select checkboxes)
  - `loading` (boolean)
  - `error` (string)
- **Methods:**
  - `handleStoryChange(field, value)` → update local state
  - `handleCRToggle(crId)` → add/remove from selectedCRs
  - `handleSubmit()` → call `ValidationService.submitValidation(...)` → redirect to results
- **Validation:**
  - Story title required, description required
  - At least one CR selected
  - Show validation errors before submit

**Component: `ValidationResults`**
- Displays validation results from API
- **Props:** `validationId` (from URL params)
- **Fetches:** GET /api/v1/validation/{validationId}
- **Sections:**
  - Readiness score card (large, color-coded: green/yellow/red)
  - Risk band indicator
  - Alignment summary (% coverage)
  - Tabs for: Gaps, Risks, Evidence, Traceability
  - Download HTML report button
- **Async states:** loading spinner, error message, no-data state

**Component: `GapsList`**
- Displays functional, AC, business rule, NFR gaps
- **Props:** `gaps` (array of gap objects)
- **Renders:** collapsible list with gap description, severity, suggested fix
- **Interaction:** expand/collapse, hover for related CR references

**Component: `TraceabilityMatrix`**
- Visualizes story ID → CR sections → evidence mapping
- **Props:** `traceability` (array of trace objects)
- **Renders:** table or tree view showing coverage
- **Interaction:** hover to highlight related cells

**Component: `ReadinessDashboard`**
- Shows dimension breakdown (functional, AC, business, NFR, ambiguity, risk, traceability)
- **Renders:** horizontal bar chart or radar chart
- **Props:** `dimensionScores` (object)

**Component: `ReportViewer`**
- Embeds or links to HTML report
- **Props:** `htmlPath` (server path to report)
- **Options:** inline embed (iframe) or download link
- **Download functionality:** fetch HTML and save as file

---

### 3.2 API Client Service

**Class: `ValidationService`**
- Encapsulates all HTTP calls to backend API
- Injects auth header (API key) on all requests
- **Methods:**
  - `submitValidation(story, crIds) → Promise<SubmitResponse>`
    - POST /api/v1/validate
    - Returns: validation_id, status, readiness_score, html_path
  - `getValidation(validationId) → Promise<ValidationResult>`
    - GET /api/v1/validation/{validationId}
  - `getHealth() → Promise<HealthStatus>`
    - GET /api/v1/health
- **Error handling:** catch and re-throw with user-friendly messages
- **Timeout:** 60s per request

---

### 3.3 State Management (Context API / Zustand)

**Store/Context: `ValidationStore`**
- Holds current validation session state
- **State:**
  - `currentValidation` (full validation result)
  - `loading` (boolean)
  - `error` (string)
  - `submittedStory` (story data from form)
- **Methods:**
  - `setValidation(data)` → update current validation
  - `setLoading(bool)` → set loading state
  - `clearError()` → clear error
- **Simple approach:** no Redux; keep it lightweight

---

### 3.4 Pages / Routes

- **Page: `/`** → ValidationForm
- **Page: `/results/{validationId}`** → ValidationResults (main dashboard)
- **Page: `/history`** → List of past validations (optional for MVP)

---

## Testing Strategy

### Unit Tests

**File: `test/api/ValidationOrchestrator.test.js`**
- Test each phase in isolation
- Mock: `StoryRetriever`, `CRVectorRetriever`, `EmbeddingService`, `LLMValidator`, `ScoringCalculator`
- Cases:
  - Happy path: story + CRs → all phases run → result returned
  - Missing CRs: no CR context → result with warning
  - LLM timeout: LLM fails → fallback response
  - Empty story: invalid input → error thrown

**File: `test/scorer/ScoringCalculator.test.js`**
- Test weighted score calculation
- Cases:
  - All dimensions 100 → readiness_score 100
  - All dimensions 50 → readiness_score 50
  - Ambiguity at 100 (high) → inverted to 0 in weighting
  - Risk at 20 (high) → inverted to 80 in weighting
  - Verify weight percentages sum correctly

**File: `test/db/ValidationRepository.test.js`**
- Mock MongoDB
- Cases:
  - Save validation → returns saved doc with _id
  - Find by story_id → returns array of validations
  - Add comment → comment appended to validation
  - Update score → dimension score updated

**File: `test/reporting/HTMLReportGenerator.test.js`**
- Mock file system
- Cases:
  - Generate HTML → valid HTML structure
  - All sections present (executive summary, gaps, traceability, etc.)
  - File saved to correct path
  - HTML is self-contained (no external CDN calls)

**File: `test/embedding/EmbeddingService.test.js`**
- Mock embedding API
- Cases:
  - Embed text → returns vector of correct size
  - Same text → returns cached result on second call
  - API timeout → throws error

**File: `test/llm/PromptBuilder.test.js`**
- Cases:
  - Build functional alignment prompt → includes story and CR context
  - Build AC gap prompt → includes acceptance criteria section
  - Verify templates are valid (no missing variables)

---

### Integration Tests

**File: `test/integration/ValidationAPI.test.js`**
- Start Express server with mocked orchestrator
- Tests:
  - POST /api/v1/validate → 200 with valid JSON
  - GET /api/v1/validation/{id} → returns saved validation
  - Invalid request → 400 error
  - Missing auth → 401 error

**File: `test/integration/DBRoundtrip.test.js`**
- Use real MongoDB (in-memory or test instance)
- Tests:
  - Save validation → retrieve by ID → all fields match
  - Save multiple → find recent → returns in correct order
  - Add comment → comment visible in retrieved validation

---

### Manual / E2E Tests

1. **Submit story via API:**
   - POST /api/v1/validate with valid payload
   - Verify: response is 200, validation_id returned, readiness_score is 0-100
2. **Retrieve validation:**
   - GET /api/v1/validation/{id}
   - Verify: all sections present, HTML path accessible
3. **Check audit trail:**
   - Query audit_logs for validation_id
   - Verify: validation_created event logged, timestamp correct
4. **Load React dashboard:**
   - Navigate to /results/{validationId}
   - Verify: readiness score displayed, gaps visible, download works
5. **Download HTML report:**
   - Click download button
   - Verify: file downloads, opens in browser, content matches API response

---

## Design Decisions & Rationale

| Decision | Rationale |
|----------|-----------|
| **Component-based classes** | Each class has single responsibility; easier to test, debug, extend |
| **Orchestrator as state machine** | 7 phases explicit; clear flow; easier to trace what's happening |
| **Mock LLM/Vector DB in unit tests** | Avoid external API dependency in local tests; fast feedback |
| **MongoDB (document DB)** | Flexibility for versioning, audit fields; easy schema changes |
| **Separate Vector DB** | Semantic search decoupled from transactional DB; scales independently |
| **Repository pattern** | Decouples business logic from persistence; easier to work with different storage later |
| **React Context API (not Redux)** | Minimal overhead; adequate for this scope; avoid over-engineering |
| **HTML reports self-contained** | Offline access; email-friendly; no external dependencies |
| **Ingestion pipeline separate** | Runs outside API request path; no latency impact on validation endpoint |
| **Immutable audit logs** | Compliance; tamper-proof; append-only |
| **Simple Express, no containers yet** | Start small; scale only when needed; Phase 2 can add Docker/orchestration |

---

## Summary: What Gets Built First

**API Layer (Phase 1):**
1. Auth middleware + request validation
2. Validation orchestrator + core validation classes (embedding, LLM, scoring)
3. Repository layer + MongoDB schemas
4. API endpoints (POST /validate, GET /validate/{id}, GET /health)
5. Unit tests for each component
6. Integration tests (API + DB roundtrip)

**DB Layer (Phase 2):**
1. MongoDB setup + schema definitions
2. Indexes (story_id, validation_id, etc.)
3. Ingestion pipeline for CR pre-embedding

**Web Layer (Phase 3):**
1. React form component
2. Results dashboard
3. Report viewer
4. API client service

**All:** Follows test-first mindset; each component testable in isolation; no dependencies on Phase 3 to validate Phase 1 logic.
