# Proposed Additional E2E Tests - Horus Platform

**Purpose**: Comprehensive list of ALL missing E2E test coverage based on deep platform analysis

📅 **Created**: 2025-11-17
📋 **Current Coverage**: 3 tests (~35% platform coverage)
🎯 **Target Coverage**: 13 total tests (100% platform coverage)

---

## Current Test Coverage (3 Tests)

### ✅ Test 01: File Upload E2E
- Organization/connection setup
- File upload (9 files)
- Document processing
- Weaviate ingestion

### ✅ Test 02: Google Drive E2E
- Airbyte source creation
- Sync job execution
- CDC ingestion
- Artifact routing

### ✅ Test 03: File Server Full Suite
- Parameter variations
- File type handling
- File operations (upload, get URL, update, delete)

**Coverage**: File upload, Airbyte CDC, file operations

---

## Proposed Additional Tests (10 Tests)

---

## 🔴 HIGH PRIORITY TESTS (4 tests)

---

### Test 04: RAG Query & Multi-Connection Retrieval

**Priority**: 🔴 HIGH (Core functionality)
**Duration**: 5-7 minutes
**Dependencies**: Test 01 or Test 02 (documents must be ingested)

#### Purpose
Validate the complete RAG query pipeline with multi-connection tenant queries, permission filtering, and reranking.

#### What It Tests
- ✅ RAG Manager multi-tenant queries (parallel tenant search)
- ✅ Permission-level filtering (private/restricted/public)
- ✅ Namespace filtering
- ✅ Orchestrator chat endpoints (V2 and V4)
- ✅ Cross-tenant reranking
- ✅ Source citation generation
- ✅ Phoenix trace collection

#### Test Flow
```
Phase 1: Setup (Use Test 01 organization)
  - Load test organization from Test 01
  - Verify documents ingested (3 collections)

Phase 2: Single-Connection Query
  - POST /orchestrator/chat
  - Query: "What are the main topics in my documents?"
  - Connection: single connection
  - Validate: Response, sources, citations

Phase 3: Multi-Connection Query
  - Create 2nd connection with different documents
  - Upload 5 new files to connection 2
  - Query across both connections
  - Validate: Results from both connections

Phase 4: Permission-Level Filtering
  - Upload file with permission_level: "private"
  - Upload file with permission_level: "public"
  - Query with user from different connection
  - Validate: Only public files returned

Phase 5: Namespace Filtering
  - Upload files to namespace: "sales"
  - Upload files to namespace: "legal"
  - Query with namespace filter: "sales"
  - Validate: Only sales files returned

Phase 6: Reranking Validation
  - Query with rerank: true
  - Check sources are reranked by relevance
  - Validate score ordering

Phase 7: Phoenix Trace Verification
  - Check Phoenix UI for traces
  - Validate multi-tenant context in traces
```

#### Success KPIs
- ✅ Chat response generated (200 OK)
- ✅ Sources returned with citations
- ✅ Multi-connection results merged
- ✅ Permission filtering correct (private not leaked)
- ✅ Namespace filtering correct
- ✅ Reranking applied (scores descending)
- ✅ Phoenix traces contain org/connection IDs
- ✅ Response time < 5 seconds

#### API Endpoints Tested
- `POST /orchestrator/chat` (V2)
- `POST /orchestrator/chat-v4` (V4 with permission filtering)
- `POST /rag/chunk-retriever` (multi-connection query)
- `GET /orchestrator/namespace-selection` (namespace list)

#### Test Data
- Documents from Test 01 (9 files)
- Additional 5 files for connection 2
- Files with different permission levels
- Files in different namespaces

---

### Test 05: Organization Lifecycle & Complete Deletion

**Priority**: 🔴 HIGH (Critical cleanup operation)
**Duration**: 3-5 minutes
**Dependencies**: None (creates temporary organization)

#### Purpose
Validate the complete organization deletion flow that removes ALL data across all services.

#### What It Tests
- ✅ Organization creation (full setup)
- ✅ Complete deletion cascade (orchestrator endpoint)
- ✅ LiteLLM wrapper cleanup (teams, keys, spend logs)
- ✅ PostgreSQL cleanup (organizations, teams, api keys)
- ✅ Phoenix cleanup (projects, traces)
- ✅ Weaviate cleanup (collections, tenants)
- ✅ MinIO cleanup (buckets, files)
- ✅ Airbyte cleanup (workspaces, connections)

#### Test Flow
```
Phase 1: Full Organization Setup
  - Create organization (org-delete-test-{timestamp})
  - Add 3 LLM models
  - Create 2 connection teams
  - Upload 5 files to connection 1
  - Upload 3 files to connection 2
  - Create Airbyte workspace + Google Drive source

Phase 2: Verify Data Exists
  - PostgreSQL: Organizations, teams, keys exist
  - Weaviate: 3 collections exist with 2 tenants
  - MinIO: Organization bucket with 8 files
  - Phoenix: Project exists with traces
  - Airbyte: Workspace exists

Phase 3: Complete Deletion (Single API Call)
  - DELETE /orchestrator/organization/{org_uuid}?hard_delete=true
  - Monitor deletion progress (parallel execution)

Phase 4: Verify Complete Cleanup
  - PostgreSQL: Organization row deleted
  - PostgreSQL: All teams deleted
  - PostgreSQL: All API keys deleted
  - PostgreSQL: Spend logs deleted
  - Weaviate: All collections deleted
  - MinIO: Organization bucket deleted (force)
  - Phoenix: Project deleted
  - Phoenix: All traces deleted
  - Airbyte: Workspace deleted

Phase 5: Orphan Check
  - Search PostgreSQL for any orphaned records
  - Search Weaviate for orphaned collections
  - Search MinIO for orphaned buckets
  - Validate: 0 orphans found
```

#### Success KPIs
- ✅ Organization deleted in < 30 seconds
- ✅ All PostgreSQL records removed (0 orphans)
- ✅ All Weaviate collections removed (0 orphans)
- ✅ MinIO bucket removed (force delete worked)
- ✅ Phoenix project removed
- ✅ Airbyte workspace removed
- ✅ No errors during deletion
- ✅ Parallel execution completed successfully

#### API Endpoints Tested
- `DELETE /orchestrator/organization/{org_uuid}`
- `DELETE /models/organizations/{org_uuid}` (LiteLLM wrapper)
- `DELETE /database/collection/{collection_name}` (DB Manager)
- `DELETE /file/bucket/{bucket_name}?force=true` (File Server)
- `DELETE /airbyte/workspaces/{workspace_id}` (Airbyte wrapper)

#### Critical Validation
- **MUST verify**: No orphaned data in ANY service
- **MUST test**: Force deletion of non-empty buckets
- **MUST validate**: Cascade deletion works correctly

---

### Test 06: File Operations Complete Lifecycle

**Priority**: 🔴 HIGH (Core file management)
**Duration**: 4-6 minutes
**Dependencies**: Test 01 setup

#### Purpose
Validate all file operation endpoints: upload, get URL, update metadata, delete, batch operations, deduplication.

#### What It Tests
- ✅ File upload (single + batch)
- ✅ Presigned URL generation (with expiration)
- ✅ File metadata update (namespace, tags, permission)
- ✅ File deletion (single + batch)
- ✅ Deduplication (same MD5 hash)
- ✅ File viewing (encrypted URL)
- ✅ Source sync (delete + recreate)

#### Test Flow
```
Phase 1: Single File Upload
  - Upload file: document.pdf
  - Verify: File in MinIO
  - Verify: Processing queued

Phase 2: Get Presigned URL
  - POST /file/get-url (file_hash)
  - Get presigned URL (expires in 3600s)
  - Download file using URL
  - Verify: File content matches original
  - Wait for expiration
  - Verify: URL expired (403 Forbidden)

Phase 3: Update File Metadata
  - POST /file/update-document
  - Update namespace: "sales" → "legal"
  - Update permission_level: "private" → "public"
  - Add tags: {"department": "legal", "year": "2025"}
  - Verify: Metadata updated in Weaviate

Phase 4: Batch Upload
  - Upload 5 files in single request
  - Verify: All 5 files uploaded
  - Verify: All 5 in MinIO
  - Verify: All 5 processing

Phase 5: Deduplication Test
  - Upload same file again (same MD5 hash)
  - Verify: Not re-uploaded to MinIO
  - Verify: Not re-processed (existing chunks returned)
  - Verify: Status: "already_exists"

Phase 6: View Encrypted File
  - GET /file/view/?encrypted_url={url}
  - Verify: File downloaded
  - Verify: Decryption worked

Phase 7: Delete Single File
  - POST /file/delete-documents (1 file)
  - Verify: File removed from MinIO
  - Verify: Chunks removed from Weaviate

Phase 8: Delete Batch Files
  - POST /file/delete-documents (5 files)
  - Verify: All 5 removed from MinIO
  - Verify: All chunks removed from Weaviate

Phase 9: Source Sync (Delete + Recreate)
  - POST /file/source-sync
  - Delete all files from source "raw_documents"
  - Re-upload all files
  - Verify: Old versions deleted, new versions created
```

#### Success KPIs
- ✅ Single upload: 200 OK, file in MinIO
- ✅ Batch upload: All files uploaded
- ✅ Presigned URL: Valid, expires correctly
- ✅ Metadata update: Changes reflected in Weaviate
- ✅ Deduplication: Same hash not re-processed
- ✅ Delete single: File + chunks removed
- ✅ Delete batch: All files + chunks removed
- ✅ Source sync: Old deleted, new created

#### API Endpoints Tested
- `POST /file/upload-documents`
- `POST /file/get-url`
- `GET /file/view/`
- `POST /file/update-document`
- `POST /file/delete-documents`
- `POST /file/source-sync`

---

### Test 07: Orchestrator AI Endpoints Complete Suite

**Priority**: 🔴 HIGH (Core AI functionality)
**Duration**: 8-10 minutes
**Dependencies**: Test 01 (documents ingested)

#### Purpose
Validate all orchestrator AI endpoints: chat, summarization, image description, conversation management, suggestions, metrics.

#### What It Tests
- ✅ Chat endpoints (V2, V4)
- ✅ Text summarization (chunks, documents)
- ✅ Image description generation
- ✅ Conversation management (title, save, delete)
- ✅ Prompt suggestions
- ✅ Session suggestions
- ✅ Metrics tracking (conversation quality)
- ✅ Source highlighting
- ✅ Chart visualization (PandasAI)

#### Test Flow
```
Phase 1: Chat V2 (Basic RAG)
  - POST /orchestrator/chat
  - Query: "What are the main topics?"
  - Validate: Response, sources, citations

Phase 2: Chat V4 (Permission Filtering)
  - POST /orchestrator/chat-v4
  - Query with connection-level filtering
  - Upload files with different permissions
  - Validate: Only authorized files returned

Phase 3: Text Summarization
  - POST /orchestrator/summarize-text
  - Summarize chunks from document
  - Validate: Summary length < original
  - Validate: Key points captured

Phase 4: Image Description
  - POST /orchestrator/describe-image
  - Upload image or reference image from MinIO
  - Validate: Description generated
  - Validate: Vision model used

Phase 5: Conversation Management
  - POST /orchestrator/conversation (create)
  - POST /orchestrator/chat (add messages)
  - POST /orchestrator/conversation-title (generate title)
  - GET /orchestrator/conversations (list)
  - DELETE /orchestrator/conversation/{id}

Phase 6: Prompt Suggestions
  - POST /orchestrator/suggestions
  - Get suggestions based on organization context
  - Validate: Relevant suggestions returned

Phase 7: Session Suggestions
  - POST /orchestrator/session-suggestions
  - Get suggestions for current session
  - Validate: Context-aware suggestions

Phase 8: Metrics Tracking
  - POST /orchestrator/conversation-metrics
  - Track conversation quality rating
  - POST /orchestrator/qa-ratings
  - Rate Q&A pairs
  - GET /orchestrator/metrics/conversations

Phase 9: Source Highlighting
  - POST /orchestrator/highlight-sources
  - Get highlighted excerpts from sources
  - Validate: Relevant excerpts returned

Phase 10: Chart Visualization (PandasAI)
  - POST /orchestrator/chart
  - Query: "Create a bar chart of sales by month"
  - Validate: Chart data returned
  - Validate: PandasAI executed
```

#### Success KPIs
- ✅ Chat V2: Response with sources
- ✅ Chat V4: Permission filtering works
- ✅ Summarization: Concise summary
- ✅ Image description: Vision model output
- ✅ Conversations: CRUD operations work
- ✅ Suggestions: Context-aware
- ✅ Metrics: Tracked and queryable
- ✅ Highlighting: Relevant excerpts
- ✅ Charts: PandasAI generates visualization

#### API Endpoints Tested
- `POST /orchestrator/chat`
- `POST /orchestrator/chat-v4`
- `POST /orchestrator/summarize-text`
- `POST /orchestrator/describe-image`
- `POST /orchestrator/conversation`
- `POST /orchestrator/conversation-title`
- `GET /orchestrator/conversations`
- `DELETE /orchestrator/conversation/{id}`
- `POST /orchestrator/suggestions`
- `POST /orchestrator/session-suggestions`
- `POST /orchestrator/conversation-metrics`
- `POST /orchestrator/qa-ratings`
- `POST /orchestrator/highlight-sources`
- `POST /orchestrator/chart`

---

## 🟡 MEDIUM PRIORITY TESTS (4 tests)

---

### Test 08: Airbyte CDC Complete Coverage (All Sources)

**Priority**: 🟡 MEDIUM (Extended CDC testing)
**Duration**: 15-20 minutes
**Dependencies**: None

#### Purpose
Validate ALL Airbyte sources (Confluence, GitHub, GitLab, Jira, GNews) with full CDC workflows.

#### What It Tests
- ✅ Confluence connector (pages, attachments)
- ✅ GitHub connector (issues, pull requests)
- ✅ GitLab connector (issues, merge requests)
- ✅ Jira connector (issues, comments)
- ✅ GNews connector (news articles)
- ✅ Incremental sync (CDC change detection)
- ✅ Update detection (modified items)
- ✅ Delete detection (removed items)

#### Test Flow
```
Phase 1: Confluence Integration
  - Create Confluence source (Atlassian Cloud)
  - Sync workspace pages
  - Ingest via /airbyte/ingest_data
  - Verify: Pages in Weaviate
  - Modify a page in Confluence
  - Re-sync
  - Verify: Update detected

Phase 2: GitHub Integration
  - Create GitHub source (repository)
  - Sync issues and PRs
  - Ingest via /airbyte/ingest_data
  - Verify: Issues in Weaviate

Phase 3: GitLab Integration
  - Create GitLab source
  - Sync issues and MRs
  - Ingest and verify

Phase 4: Jira Integration
  - Create Jira source (Cloud)
  - Sync issues
  - Ingest and verify

Phase 5: GNews Integration
  - Create GNews source
  - Sync news articles
  - Ingest and verify

Phase 6: CDC Update Test
  - Update item in source system
  - Re-run sync
  - Verify: CDC detects update
  - Verify: Weaviate updated

Phase 7: CDC Delete Test
  - Delete item in source system
  - Re-run sync
  - Verify: CDC detects deletion
  - Verify: Weaviate chunks removed
```

#### Success KPIs
- ✅ All 5 sources create successfully
- ✅ All sync jobs succeed
- ✅ All items ingested to Weaviate
- ✅ CDC detects updates (modified_time changed)
- ✅ CDC detects deletions (deleted_count > 0)
- ✅ Artifacts routed to `airbyte/.../` folder
- ✅ Reference file strategy used

#### API Endpoints Tested
- `POST /airbyte/create_source` (5x for each source)
- `POST /airbyte/start_job_sync` (5x)
- `POST /airbyte/job_status` (polling)
- `POST /airbyte/ingest_data` (5x)

---

### Test 09: Multi-User Workflow & Conversation Management

**Priority**: 🟡 MEDIUM (Multi-user scenarios)
**Duration**: 6-8 minutes
**Dependencies**: Test 01

#### Purpose
Validate multi-user scenarios with conversation isolation, sharing, and collaboration.

#### What It Tests
- ✅ User-level isolation (conversations, suggestions)
- ✅ Conversation save/load
- ✅ Conversation sharing (future: permissions)
- ✅ Concurrent user queries (parallel requests)
- ✅ User-specific suggestions
- ✅ Conversation history retrieval

#### Test Flow
```
Phase 1: User 1 - Create Conversation
  - Create user 1 context
  - POST /orchestrator/conversation
  - Add 5 messages
  - Generate conversation title
  - Save conversation

Phase 2: User 2 - Create Separate Conversation
  - Create user 2 context
  - Create new conversation
  - Verify: User 1's conversation NOT visible

Phase 3: Concurrent Queries (Load Test)
  - Simulate 10 concurrent users
  - Each sends chat query
  - Verify: All get responses
  - Verify: No cross-user data leakage

Phase 4: Conversation Retrieval
  - User 1 GET /orchestrator/conversations
  - Verify: Only user 1's conversations returned
  - User 2 GET /orchestrator/conversations
  - Verify: Only user 2's conversations returned

Phase 5: User-Specific Suggestions
  - User 1 POST /orchestrator/session-suggestions
  - Upload documents to user 1's connection
  - Verify: Suggestions based on user 1's documents
  - User 2 suggestions should be different
```

#### Success KPIs
- ✅ Conversations isolated by user
- ✅ No cross-user data leakage
- ✅ Concurrent queries succeed (10 parallel)
- ✅ Suggestions are user-specific
- ✅ Conversation history correct per user

---

### Test 10: Webhook & Processing Status Notifications

**Priority**: 🟡 MEDIUM (Status tracking)
**Duration**: 5-7 minutes
**Dependencies**: Test 01

#### Purpose
Validate webhook registration, processing status updates, and real-time notifications.

#### What It Tests
- ✅ Webhook registration
- ✅ Processing status events
- ✅ Webhook callbacks (HTTP POST to registered URL)
- ✅ Status query endpoint
- ✅ WebSocket status streaming (future)

#### Test Flow
```
Phase 1: Webhook Registration
  - POST /file/register-webhook
  - URL: http://webhook-test-server:5000/callback
  - Events: ["processing.started", "processing.completed", "processing.failed"]
  - Verify: Webhook registered

Phase 2: Upload File & Monitor Webhooks
  - Upload document.pdf
  - Start webhook test server (Flask)
  - Wait for callbacks
  - Verify: "processing.started" callback received
  - Verify: "processing.completed" callback received

Phase 3: Processing Status Query
  - GET /file/status/{file_hash}
  - Verify: Status returned (processing, completed, failed)
  - Verify: Step and percentage updated

Phase 4: Batch Status Query
  - GET /file/status/connection/{connection_id}
  - Verify: All files for connection returned
  - Verify: Each file has status

Phase 5: Webhook Deactivation
  - DELETE /file/webhooks/{webhook_id}
  - Verify: Webhook deactivated
  - Upload new file
  - Verify: No callback received

Phase 6: Failed Processing Webhook
  - Upload corrupted file
  - Verify: "processing.failed" callback
  - Verify: Error message included
```

#### Success KPIs
- ✅ Webhook registered successfully
- ✅ All events trigger callbacks
- ✅ Callback payload contains file metadata
- ✅ Status query returns current state
- ✅ Batch status returns all files
- ✅ Deactivated webhooks don't fire

#### API Endpoints Tested
- `POST /file/register-webhook`
- `GET /file/webhooks/{org_id}/{conn_id}`
- `DELETE /file/webhooks/{webhook_id}`
- `GET /file/status/{file_hash}`
- `GET /file/status/connection/{connection_id}`

---

### Test 11: LiteLLM Wrapper Analytics & Spend Tracking

**Priority**: 🟡 MEDIUM (Cost tracking)
**Duration**: 4-6 minutes
**Dependencies**: Test 01 (LLM calls made)

#### Purpose
Validate spend tracking, analytics, budget limits, and cost reporting across all levels.

#### What It Tests
- ✅ Organization-level spend tracking
- ✅ Team-level spend tracking
- ✅ User-level spend tracking
- ✅ Model-level spend breakdown
- ✅ Budget limit enforcement
- ✅ Spend analytics queries
- ✅ Cost projection

#### Test Flow
```
Phase 1: Organization Spend Tracking
  - Create organization with budget: $100
  - Upload 10 documents (triggers LLM calls)
  - GET /models/analytics/spend/organization/{org_uuid}
  - Verify: Spend > 0 (embeddings + LLM summarization)
  - Verify: Spend breakdown by model

Phase 2: Team Spend Tracking
  - Create team with budget: $50
  - Make 100 chat queries
  - GET /models/analytics/spend/team/{team_id}
  - Verify: Spend tracked per team
  - Verify: Within team budget

Phase 3: User Spend Tracking
  - Create 3 users
  - Each user makes 10 queries
  - GET /models/analytics/spend/user/{user_id}
  - Verify: Spend tracked per user
  - Verify: User attribution correct

Phase 4: Model Breakdown
  - GET /models/analytics/spend/model/{model_id}
  - Verify: Spend for gpt-4o-mini tracked
  - Verify: Token counts correct
  - Verify: Cost calculation accurate

Phase 5: Budget Limit Enforcement
  - Create team with budget: $1.00
  - Make queries until budget exceeded
  - Verify: 429 Too Many Requests returned
  - Verify: Error message: "Budget exceeded"

Phase 6: Spend Analytics
  - GET /models/analytics/spend/organization/{org_uuid}?start_date=...&end_date=...
  - Verify: Time-range filtering works
  - Verify: Aggregation correct

Phase 7: Cost Projection
  - GET /models/analytics/spend/projection/{org_uuid}
  - Verify: Monthly projection calculated
  - Verify: Based on current usage trends
```

#### Success KPIs
- ✅ Organization spend tracked accurately
- ✅ Team spend isolated correctly
- ✅ User spend attributed correctly
- ✅ Model breakdown shows token usage
- ✅ Budget limits enforced (429 when exceeded)
- ✅ Time-range queries work
- ✅ Cost projection reasonable

#### API Endpoints Tested
- `GET /models/analytics/spend/organization/{org_uuid}`
- `GET /models/analytics/spend/team/{team_id}`
- `GET /models/analytics/spend/user/{user_id}`
- `GET /models/analytics/spend/model/{model_id}`
- `GET /models/analytics/spend/projection/{org_uuid}`

---

## 🟢 LOW PRIORITY TESTS (2 tests)

---

### Test 12: Phoenix Observability & Trace Validation

**Priority**: 🟢 LOW (Monitoring/debugging)
**Duration**: 3-5 minutes
**Dependencies**: Test 01 (LLM calls made)

#### Purpose
Validate Phoenix trace collection, multi-tenant context, and trace query capabilities.

#### What It Tests
- ✅ Phoenix trace collection (all services)
- ✅ Multi-tenant context in traces (org, connection, user)
- ✅ Trace query via Phoenix API
- ✅ Span attributes (model, tokens, latency)
- ✅ Trace filtering by organization/connection
- ✅ Phoenix UI functionality

#### Test Flow
```
Phase 1: Trace Collection Verification
  - Make 10 chat queries
  - Wait 10 seconds (trace flush interval)
  - Query Phoenix API: GET /v1/traces
  - Verify: 10 traces collected

Phase 2: Multi-Tenant Context
  - Query traces for organization
  - Verify: All traces have organization_id attribute
  - Verify: All traces have connection_id attribute
  - Verify: User_id attribute present

Phase 3: Span Attributes
  - Query trace details
  - Verify: Model name (e.g., "gpt-4o-mini")
  - Verify: Token counts (input, output)
  - Verify: Latency (ms)
  - Verify: Status (OK, ERROR)

Phase 4: Trace Filtering
  - Filter traces by organization_id
  - Verify: Only org's traces returned
  - Filter traces by connection_id
  - Verify: Only connection's traces returned

Phase 5: Phoenix UI Validation
  - Open Phoenix UI: http://localhost:6006
  - Verify: Project exists (org_uuid)
  - Verify: Traces visible in UI
  - Verify: Spans expandable
  - Verify: Attributes visible
```

#### Success KPIs
- ✅ All LLM calls traced
- ✅ Multi-tenant context in every trace
- ✅ Span attributes complete
- ✅ Filtering works correctly
- ✅ Phoenix UI functional

---

### Test 13: Advanced Features Suite

**Priority**: 🟢 LOW (Edge cases & future features)
**Duration**: 10-15 minutes
**Dependencies**: Test 01

#### Purpose
Validate advanced features, edge cases, and experimental functionality.

#### What It Tests
- ✅ Large file uploads (>100 MB)
- ✅ Special characters in filenames
- ✅ Unsupported file types (graceful failure)
- ✅ Concurrent uploads (10 parallel)
- ✅ Network interruption handling (retry)
- ✅ Encryption/decryption correctness
- ✅ Rate limiting (429 handling)
- ✅ Model fallback (primary fails → fallback)

#### Test Flow
```
Phase 1: Large File Upload
  - Upload 150 MB PDF
  - Verify: Upload succeeds (max 100 MB may fail, validate error)
  - Verify: Processing handles large file

Phase 2: Special Characters
  - Upload file: "document (copy) [final] - v2.0.pdf"
  - Verify: Filename sanitized correctly
  - Verify: Clean filename generated

Phase 3: Unsupported File Type
  - Upload file: executable.exe
  - Verify: 400 Bad Request
  - Verify: Error message: "Unsupported file type"

Phase 4: Concurrent Uploads
  - Upload 10 files in parallel (10 threads)
  - Verify: All 10 succeed
  - Verify: No race conditions

Phase 5: Network Interruption
  - Start upload
  - Simulate network failure (kill connection)
  - Verify: Retry logic kicks in
  - Verify: Upload eventually succeeds

Phase 6: Encryption Validation
  - Upload file
  - Get encrypted URL
  - Decrypt URL
  - Verify: Decrypted path matches original

Phase 7: Rate Limiting
  - Make 1000 requests in 10 seconds
  - Verify: 429 Too Many Requests
  - Verify: Retry-After header

Phase 8: Model Fallback
  - Configure primary model: gpt-4o (expensive)
  - Configure fallback: gpt-4o-mini (cheap)
  - Simulate primary model failure (disable API key)
  - Make chat query
  - Verify: Fallback model used
  - Verify: Response still generated
```

#### Success KPIs
- ✅ Large file handling validated
- ✅ Special characters sanitized
- ✅ Unsupported types rejected gracefully
- ✅ Concurrent uploads succeed
- ✅ Retry logic works
- ✅ Encryption/decryption correct
- ✅ Rate limiting enforced
- ✅ Model fallback functional

---

## Summary: All Proposed Tests

| # | Test Name | Priority | Duration | Coverage Area |
|---|-----------|----------|----------|---------------|
| **Current Tests** | | | | |
| 01 | File Upload E2E | ✅ | 2-3 min | Upload, processing, ingestion |
| 02 | Google Drive E2E | ✅ | 3-4 min | Airbyte CDC, Google Drive |
| 03 | File Server Full Suite | ✅ | 30-45 min | Parameter variations, file ops |
| **HIGH Priority (Must Have)** | | | | |
| 04 | RAG Query & Multi-Connection Retrieval | 🔴 | 5-7 min | RAG, chat, multi-tenant queries |
| 05 | Organization Lifecycle & Deletion | 🔴 | 3-5 min | Org management, cleanup |
| 06 | File Operations Complete Lifecycle | 🔴 | 4-6 min | File ops, presigned URLs |
| 07 | Orchestrator AI Endpoints Suite | 🔴 | 8-10 min | Chat, summarization, AI features |
| **MEDIUM Priority (Should Have)** | | | | |
| 08 | Airbyte CDC Complete Coverage | 🟡 | 15-20 min | All Airbyte sources |
| 09 | Multi-User Workflow | 🟡 | 6-8 min | User isolation, conversations |
| 10 | Webhook & Status Notifications | 🟡 | 5-7 min | Webhooks, status tracking |
| 11 | LiteLLM Analytics & Spend | 🟡 | 4-6 min | Cost tracking, budgets |
| **LOW Priority (Nice to Have)** | | | | |
| 12 | Phoenix Observability | 🟢 | 3-5 min | Tracing, monitoring |
| 13 | Advanced Features Suite | 🟢 | 10-15 min | Edge cases, experimental |

**Total Tests**: 13
**Total Estimated Time**: ~100-130 minutes (full suite)
**Current Coverage**: 3 tests (~35%)
**Target Coverage**: 13 tests (100%)

---

## Coverage Analysis

### Current Coverage (35%)
- ✅ File upload (Test 01)
- ✅ Airbyte CDC - Google Drive only (Test 02)
- ✅ File operations - basic (Test 03)

### Missing Critical Coverage (HIGH Priority)
- ❌ RAG queries & multi-connection retrieval
- ❌ Organization deletion & cleanup
- ❌ Complete file operations lifecycle
- ❌ Orchestrator AI endpoints (chat, summarization, image description)

### Missing Important Coverage (MEDIUM Priority)
- ❌ Other Airbyte sources (Confluence, GitHub, GitLab, Jira)
- ❌ Multi-user workflows
- ❌ Webhooks & status notifications
- ❌ Spend tracking & analytics

### Missing Nice-to-Have Coverage (LOW Priority)
- ❌ Phoenix observability validation
- ❌ Advanced features & edge cases

---

## Recommended Implementation Order

### Phase 1: Core Functionality (Weeks 1-2)
1. **Test 04**: RAG Query & Multi-Connection Retrieval
2. **Test 05**: Organization Lifecycle & Deletion
3. **Test 06**: File Operations Complete Lifecycle
4. **Test 07**: Orchestrator AI Endpoints Suite

**Rationale**: These tests cover critical user-facing functionality that is used in production daily.

### Phase 2: Extended Coverage (Weeks 3-4)
5. **Test 08**: Airbyte CDC Complete Coverage
6. **Test 09**: Multi-User Workflow
7. **Test 10**: Webhook & Status Notifications
8. **Test 11**: LiteLLM Analytics & Spend

**Rationale**: These tests validate important but less frequently used features.

### Phase 3: Monitoring & Edge Cases (Week 5)
9. **Test 12**: Phoenix Observability
10. **Test 13**: Advanced Features Suite

**Rationale**: These tests ensure system reliability and handle edge cases.

---

## Test Infrastructure Requirements

### Additional Test Data Needed
- **Test 04**: Documents with different permission levels, multiple namespaces
- **Test 05**: Temporary organization (auto-deleted)
- **Test 07**: Images for vision model, CSV for PandasAI charts
- **Test 08**: Credentials for Confluence, GitHub, GitLab, Jira, GNews
- **Test 10**: Webhook test server (Flask/FastAPI)

### Services Required
- All current services (already running)
- Webhook test server (new, for Test 10)
- Load test harness (for Test 09, Test 13)

### API Keys & Credentials
- ✅ OpenAI API key (existing)
- ✅ Anthropic API key (existing)
- 🔴 Confluence credentials (Test 08)
- 🔴 GitHub token (Test 08)
- 🔴 GitLab token (Test 08)
- 🔴 Jira credentials (Test 08)
- 🔴 GNews API key (Test 08)

---

## Success Metrics

### Test Suite Health
- **Pass Rate**: 100% (all tests must pass)
- **Execution Time**: < 2 hours (full suite)
- **Flakiness**: < 1% (tests should be deterministic)

### Platform Coverage
- **Endpoint Coverage**: 100% of public-facing endpoints
- **Service Coverage**: 100% of microservices
- **Feature Coverage**: 100% of documented features

### Quality Metrics
- **No orphaned data**: Deletion tests verify complete cleanup
- **No data leakage**: Multi-tenant tests verify isolation
- **No performance regressions**: Tests track response times

---

## Next Steps

### For User Review
Please review this document and indicate:
1. ✅ Which tests to implement first (HIGH priority recommended)
2. ✅ Any tests to add/remove/modify
3. ✅ Any additional validation requirements
4. ✅ Timeline expectations

### For Implementation
Once approved, I will:
1. Create test scripts for approved tests
2. Set up test infrastructure (webhook server, etc.)
3. Create test data and credentials
4. Update E2E README with new tests
5. Add to CI/CD pipeline (future)

---

**Document Version**: 1.0
**Author**: Claude (Anthropic)
**Review Status**: ⏳ Pending User Review
