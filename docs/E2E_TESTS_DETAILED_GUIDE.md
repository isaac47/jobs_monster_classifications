# E2E Tests Detailed Guide - Horus Platform

**Complete flow analysis, diagrams, and success KPIs for all E2E test cases**

📅 **Last Updated**: 2025-11-17
📋 **Test Count**: 3 comprehensive test suites
✅ **Status**: All tests verified and working

---

## Table of Contents

1. [Test 01: File Upload E2E](#test-01-file-upload-e2e)
2. [Test 02: Google Drive E2E](#test-02-google-drive-e2e)
3. [Test 03: File Server Full Suite](#test-03-file-server-full-suite)
4. [Appendix: Common Patterns](#appendix-common-patterns)

---

# Test 01: File Upload E2E

## Overview

**Purpose**: Validate the complete file upload pipeline from organization creation through to Weaviate multi-tenant ingestion.

**Location**: `e2e/01_file_upload_e2e/`
**Duration**: 2-3 minutes
**Test Files**: 9 diverse document types
**Services Tested**: LiteLLM Wrapper, File Server, Data Manager, DB Manager, RabbitMQ, MinIO, Weaviate

### What This Test Validates

This test ensures the entire document processing pipeline works end-to-end:

1. **Multi-tenant Architecture**: Organization and connection-level isolation
2. **Model Management**: LLM and embedding model configuration via LiteLLM wrapper
3. **File Upload**: Multipart file upload with comprehensive metadata
4. **Storage Routing**: Organization-centric MinIO bucket structure
5. **Async Processing**: RabbitMQ message queue handling
6. **Document Processing**: 13-stage Docling pipeline (chunking, embedding, AI enhancement)
7. **Vector Ingestion**: Multi-tenant Weaviate storage with 3 collection types
8. **Observability**: Phoenix tracing and spend tracking

---

## Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                       TEST 01: FILE UPLOAD E2E                      │
└─────────────────────────────────────────────────────────────────────┘

Phase 1: Organization Setup
─────────────────────────────
  User Script
      │
      ├─── POST /models/organizations/ ──→ LiteLLM Wrapper
      │                                         │
      │                                         ├─ Create org in PostgreSQL
      │                                         ├─ Auto-add 3 embedding models:
      │                                         │   • embedanything-minilm
      │                                         │   • bge-m3
      │                                         │   • jina
      │                                         │
      │    ←─── organization_id, models[] ─────┘
      │
      ✓ SUCCESS: Organization created with UUID

Phase 2: Model Configuration
─────────────────────────────
  User Script
      │
      ├─── POST /models/models/ (Claude Chat) ──→ LiteLLM Wrapper
      │    {                                           │
      │      model_name: "claude-37-sonnet-chat"      │
      │      capabilities: ["chat"]                   ├─ Register model
      │    }                                           │
      │    ←─── model_id ───────────────────────────┘
      │
      ├─── POST /models/models/ (Claude Vision) ──→ LiteLLM Wrapper
      │    {                                            │
      │      model_name: "claude-37-sonnet-vision"     │
      │      capabilities: ["chat", "vision"]          ├─ Register model
      │    }                                            │
      │    ←─── model_id ───────────────────────────┘
      │
      ├─── POST /models/models/ (OpenAI) ──→ LiteLLM Wrapper
      │    {                                      │
      │      model_name: "openai-gpt4o-mini"      │
      │      capabilities: ["chat", "vision"]     ├─ Register model
      │    }                                       │
      │    ←─── model_id ──────────────────────┘
      │
      ✓ SUCCESS: 3 LLM models + 3 embedding models available

Phase 3: Connection Team Setup
───────────────────────────────
  User Script
      │
      ├─── POST /models/teams/ ──→ LiteLLM Wrapper
      │    {                             │
      │      team_type: "connection"     │
      │      organization_uuid: "..."    ├─ Create connection team
      │    }                              │   - Generate API key
      │                                   │   - Link to organization
      │    ←─── team_id, api_key ────────┘
      │
      ✓ SUCCESS: Connection team with API key created

Phase 4: File Upload
─────────────────────
  User Script (9 files in multipart/form-data)
      │
      ├─── POST /file/upload-documents ──→ File Server (8020)
      │    Params (per file):                    │
      │      - namespace: "e2e-test-workspace"   │
      │      - permission_level: "private"       │
      │      - table_mode: "auto"                │
      │      - ocr_mode: "auto"                  │
      │      - includes_images: 1                │
      │    Global params:                        │
      │      - organization_id                   │
      │      - connection_id                     │
      │      - llm_models: [...]                 │
      │      - embedding_model                   │
      │      - team_api_key                      │
      │                                           │
      │                                           ├─ Validate files
      │                                           ├─ Compute MD5 hash
      │                                           ├─ Save to MinIO:
      │                                           │   org-{uuid}/uploads/
      │                                           │     conn-{uuid}/documents/
      │                                           │
      │    ←─── files[] (hash, status) ──────────┘
      │         {
      │           filename: "document.pdf"
      │           file_hash: "abc123..."
      │           status: "queued"
      │         }
      │
      ✓ SUCCESS: 9 files uploaded to MinIO

Phase 5: Async Message Queue
─────────────────────────────
  File Server
      │
      ├─── Publish Message ──→ RabbitMQ (queue: file_notifications)
      │    {                         │
      │      file_hash               │
      │      organization_id          │
      │      connection_id            │
      │      encrypted_url            │
      │      processing_params        │
      │    }                          │
      │                               │
      │                               ├─ Queue message
      │                               │
  Data Manager (Consumer) ←───────────┘
      │
      ├─ Consume from queue
      ├─ Decrypt file URL
      ├─ Download from MinIO
      │
      ✓ SUCCESS: Messages queued, consumers active

Phase 6: Document Processing (Data Manager)
────────────────────────────────────────────
  Data Manager (13-stage Docling pipeline)
      │
      ├─ Stage 1: Download file from MinIO
      ├─ Stage 2: Convert to Markdown (Docling)
      ├─ Stage 3: Extract tables (auto mode)
      ├─ Stage 4: Extract images (OCR if needed)
      ├─ Stage 5: Generate Table of Contents
      ├─ Stage 6: Semantic chunking
      ├─ Stage 7: Generate embeddings (embedanything-minilm)
      ├─ Stage 8: AI image description (vision model)
      ├─ Stage 9: AI table summarization (chat model)
      ├─ Stage 10: Document summarization (chat model)
      ├─ Stage 11: Sentence splitting (precision retrieval)
      ├─ Stage 12: Upload artifacts to MinIO:
      │             - images/ (extracted images)
      │             - tables/ (extracted tables)
      │             - chunks/ (JSON chunks)
      │             - doc_summaries/ (summaries)
      ├─ Stage 13: Send chunks to DB Manager
      │
      ✓ SUCCESS: All 9 files processed

Phase 7: Weaviate Ingestion (DB Manager)
─────────────────────────────────────────
  DB Manager
      │
      ├─── POST /database/document ──→ Weaviate (8080)
      │                                     │
      │                                     ├─ Create/Get Collections:
      │                                     │   • {Org}_FullText (multi-tenant)
      │                                     │   • {Org}_Sentence (multi-tenant)
      │                                     │   • {Org}_DocSummary (multi-tenant)
      │                                     │
      │                                     ├─ Create/Get Tenant:
      │                                     │   • conn_{connection_uuid}
      │                                     │   (hyphens removed for sanitization)
      │                                     │
      │                                     ├─ Insert Objects:
      │                                     │   - FullText chunks
      │                                     │   - Sentence chunks
      │                                     │   - Document summaries
      │                                     │
      │    ←─── success: true ──────────────┘
      │
      ✓ SUCCESS: All chunks ingested to Weaviate

Phase 8: Verification & Monitoring
───────────────────────────────────
  Test Script
      │
      ├─── Check MinIO Structure
      │    • org-{uuid}/uploads/conn-{uuid}/documents/ (9 files)
      │    • org-{uuid}/uploads/conn-{uuid}/images/ (extracted)
      │    • org-{uuid}/uploads/conn-{uuid}/tables/ (extracted)
      │
      ├─── Query Weaviate Collections
      │    GET /v1/schema → Find collections
      │    GET /v1/objects?class={Org}_FullText&tenant=conn_{uuid}&limit=100
      │
      ├─── Check Spend Tracking
      │    GET /models/analytics/spend/organization/{org_uuid}
      │    GET /models/analytics/spend/team/{team_id}
      │
      ├─── Check RabbitMQ Queue
      │    rabbitmqctl list_queues → 0 messages pending
      │
      ✓ SUCCESS: All validations passed
```

---

## Phase-by-Phase Breakdown

### Phase 1: Organization Setup

**Goal**: Create a multi-tenant organization with auto-provisioned embedding models.

**API Call**:
```bash
POST http://localhost:8100/models/organizations/
Content-Type: application/json

{
  "organization_uuid": "fileupload-test-20251117140000",
  "organization_alias": "File Upload E2E Test",
  "max_budget": 100.0,
  "metadata": {
    "purpose": "e2e-testing",
    "test_type": "file-upload"
  }
}
```

**Expected Response**:
```json
{
  "organization_id": "org_abc123...",
  "organization_uuid": "fileupload-test-20251117140000",
  "models": [
    "embedanything-minilm",
    "bge-m3",
    "jina"
  ],
  "created_at": "2025-11-17T14:00:00Z"
}
```

**What Happens**:
1. LiteLLM wrapper creates organization in PostgreSQL
2. Automatically provisions 3 default embedding models
3. Initializes spend tracking (0.00 initial spend)
4. Returns organization ID for subsequent operations

**Success Criteria**:
- ✅ `organization_id` is not null
- ✅ Exactly 3 embedding models auto-added
- ✅ Response time < 2 seconds

---

### Phase 2: Model Configuration

**Goal**: Register LLM models (Claude chat, Claude vision, OpenAI) for document processing.

**API Calls** (3 sequential):

**2.1 - Claude Chat Model**:
```bash
POST http://localhost:8100/models/models/
Content-Type: application/json

{
  "organization_uuid": "fileupload-test-20251117140000",
  "model": {
    "model_name": "claude-37-sonnet-chat",
    "litellm_params": {
      "model": "anthropic/claude-3-7-sonnet-20250219",
      "api_key": "sk-ant-api03-..."
    },
    "model_info": {
      "capabilities": ["chat"],
      "mode": "chat",
      "supports_function_calling": true,
      "supports_vision": false
    }
  }
}
```

**2.2 - Claude Vision Model**:
```json
{
  "model_name": "claude-37-sonnet-vision",
  "capabilities": ["chat", "vision"],
  "supports_vision": true
}
```

**2.3 - OpenAI GPT-4o-mini**:
```json
{
  "model_name": "openai-gpt4o-mini",
  "litellm_params": {
    "model": "openai/gpt-4o-mini"
  },
  "capabilities": ["chat", "vision"]
}
```

**What Happens**:
1. LiteLLM wrapper validates model configuration
2. Stores model credentials encrypted in PostgreSQL
3. Registers model with LiteLLM proxy server
4. Makes models available for organization

**Success Criteria**:
- ✅ All 3 models registered successfully
- ✅ Model IDs returned for each
- ✅ Total models = 6 (3 embedding + 3 LLM)

---

### Phase 3: Connection Team Setup

**Goal**: Create a connection-level team with API key for authentication.

**API Call**:
```bash
POST http://localhost:8100/models/teams/
Content-Type: application/json

{
  "team_uuid": "conn-upload-20251117140000",
  "organization_uuid": "fileupload-test-20251117140000",
  "team_type": "connection",
  "team_alias": "File Upload Connection",
  "max_budget": 50.0,
  "metadata": {
    "connection_type": "file-upload",
    "test_timestamp": "20251117140000"
  }
}
```

**Expected Response**:
```json
{
  "team_id": "team_xyz789...",
  "team_uuid": "conn-upload-20251117140000",
  "api_key": "sk-1aUwAnKM3NH9i-oZBGFRLg",
  "budget_remaining": 50.0,
  "created_at": "2025-11-17T14:00:05Z"
}
```

**What Happens**:
1. Creates connection team in PostgreSQL
2. Generates unique API key (sk-...)
3. Links team to organization
4. Inherits organization's models
5. Sets team-level budget (50.0 USD)

**Success Criteria**:
- ✅ `team_id` and `api_key` returned
- ✅ API key starts with "sk-"
- ✅ Budget correctly set

---

### Phase 4: File Upload

**Goal**: Upload 9 diverse files with comprehensive metadata.

**Test Files** (9 files, 25.4 MB total):
1. `photo of a a4 format with text.jpeg` (3.2 MB) - Standalone image
2. `Relevé de situation_20250515 (1).pdf` (0.8 MB) - Financial PDF
3. `one sheet table.csv` (2 KB) - Simple CSV
4. `multi page financial with image and table embed.pdf` (4.5 MB) - Complex PDF
5. `pelousi-annual-accounts-simple.html` (15 KB) - HTML document
6. `scan pdf of full text page.pdf` (1.2 MB) - Scanned PDF (OCR)
7. `simple md file.md` (1 KB) - Markdown text
8. `costs (1).csv` (3 KB) - CSV with special chars
9. `YCombinato-2a0506eb97 - Copie (1).pptx` (15.7 MB) - PowerPoint

**API Call**:
```bash
POST http://localhost:8020/file/upload-documents
Content-Type: multipart/form-data

# Files (9x)
--boundary
Content-Disposition: form-data; name="files"; filename="document.pdf"
Content-Type: application/pdf

[binary data]
--boundary

# Per-file parameters (9x each)
namespace=e2e-test-workspace
permission_level=private
table_mode=auto
ocr_mode=auto
includes_images=1

# Global parameters
organization_id=fileupload-test-20251117140000
connection_id=conn-upload-20251117140000
collection_name=FileUploadTest
source_provider=raw_documents
embedding_model=embedanything-minilm
llm_models=[{"model_id":"claude-37-sonnet-chat","capabilities":["chat"]},...]
team_api_key=sk-1aUwAnKM3NH9i-oZBGFRLg
```

**Expected Response**:
```json
{
  "files": [
    {
      "filename": "photo of a a4 format with text.jpeg",
      "clean_filename": "photo_of_a_a4_format_with_text.jpeg",
      "file_hash": "f61e8a479d2313f31bb638c98bb16803",
      "file_checksum_id": "f61e8a479d2313f31bb638c98bb16803",
      "status": "queued",
      "file_type": "image",
      "size_bytes": 3355648,
      "mime_type": "image/jpeg"
    },
    // ... 8 more files
  ],
  "total_files": 9,
  "organization_bucket": "org-fileupload-test-20251117140000",
  "connection_path": "uploads/conn-conn-upload-20251117140000"
}
```

**What Happens**:
1. **File Server validates parameters**:
   - Checks file count matches metadata count
   - Validates team API key
   - Checks organization exists
2. **Computes MD5 hash** for each file (deduplication)
3. **Saves to MinIO**:
   - Bucket: `org-fileupload-test-20251117140000`
   - Path: `uploads/conn-conn-upload-20251117140000/documents/`
   - Filename: `{hash}_{original_filename}`
4. **Publishes to RabbitMQ**:
   - Queue: `file_notifications`
   - Message: encrypted URL + metadata
5. **Returns upload response** with file hashes

**Success Criteria**:
- ✅ All 9 files return `status: "queued"`
- ✅ Unique MD5 hash for each file
- ✅ Files exist in MinIO bucket
- ✅ Response time < 10 seconds

---

### Phase 5: Async Message Queue

**Goal**: Verify RabbitMQ message queuing and consumption.

**RabbitMQ Message Format**:
```json
{
  "file_hash": "f61e8a479d2313f31bb638c98bb16803",
  "organization_id": "fileupload-test-20251117140000",
  "connection_id": "conn-upload-20251117140000",
  "namespace": "e2e-test-workspace",
  "collection_name": "FileUploadTest",
  "encrypted_url": "gAAAAABpEUCm-zGRvEKBzlsW4wdw...",
  "custom_parameters": {
    "table_mode": "auto",
    "ocr_mode": "auto",
    "includes_images": true
  },
  "llm_models": [...],
  "embedding_model": "embedanything-minilm"
}
```

**What Happens**:
1. **File Server publishes** 9 messages to queue
2. **Data Manager consumes** messages (3 workers: data-manager_1, _2, _3)
3. **Parallel processing** starts (3 files at a time)
4. **Message acknowledgment** after successful download

**Verification Commands**:
```bash
# Check queue status
docker exec rabbitmq-broker rabbitmqctl list_queues name messages consumers

# Expected output:
# file_notifications  0  3  (0 messages, 3 consumers)
```

**Success Criteria**:
- ✅ Queue `file_notifications` exists
- ✅ 3 consumers connected (data-manager_1, _2, _3)
- ✅ Messages processed (0 remaining after 30s)
- ✅ No messages in Dead Letter Queue (DLQ)

---

### Phase 6: Document Processing

**Goal**: Execute 13-stage Docling pipeline for all 9 files.

**Processing Pipeline** (per file):

**Stage 1-2: Download & Convert**
```python
# Data manager receives message
file_url = decrypt_url(message['encrypted_url'])
file_path = download_from_minio(file_url)
markdown_content = docling.convert(file_path)  # PDF → Markdown
```

**Stage 3-4: Extract Tables & Images**
```python
# table_mode: "auto" → detect tables automatically
tables = extract_tables(markdown_content)  # Using Docling table detection
images = extract_images(file_path)         # Extract embedded images

# Upload to MinIO
for table in tables:
    save_to_minio(table, "tables/table_001.csv")
for image in images:
    save_to_minio(image, "images/image_001.png")
```

**Stage 5-6: TOC & Chunking**
```python
# Generate Table of Contents
toc = generate_toc(markdown_content)

# Semantic chunking (Docling)
chunks = semantic_chunker(
    markdown_content,
    chunk_size=512,  # tokens
    overlap=50,       # token overlap
    respect_boundaries=True  # Don't split mid-sentence
)
```

**Stage 7: Embedding Generation**
```python
# Generate embeddings (embedanything-minilm)
for chunk in chunks:
    chunk['embedding'] = embedding_service.embed(
        chunk['text'],
        model="embedanything-minilm"
    )  # Returns 384-dim vector
```

**Stage 8-9: AI Enhancement**
```python
# Image description (vision model)
for image in images:
    image['description'] = llm_call(
        model="claude-37-sonnet-vision",
        prompt="Describe this image in detail",
        image=image['data']
    )

# Table summarization (chat model)
for table in tables:
    table['summary'] = llm_call(
        model="claude-37-sonnet-chat",
        prompt="Summarize this table",
        context=table['markdown']
    )
```

**Stage 10-11: Summarization & Sentences**
```python
# Document summary
doc_summary = llm_call(
    model="claude-37-sonnet-chat",
    prompt="Summarize this document in 2-3 sentences",
    context=full_markdown[:4000]  # First 4k chars
)

# Sentence splitting (for precision retrieval)
sentences = split_into_sentences(full_markdown)
# Each sentence gets its own embedding
```

**Stage 12: Artifact Upload**
```python
# Save chunks to MinIO
save_to_minio(chunks, "chunks/chunk_0000.json")
save_to_minio(doc_summary, "doc_summaries/summary.json")
save_to_minio(images_metadata, "images/image_metadata.json")
save_to_minio(tables_metadata, "tables/table_metadata.json")
```

**Stage 13: Send to DB Manager**
```python
# Send chunks to Weaviate
response = db_manager.ingest_chunks(
    chunks=chunks,
    organization_id=org_id,
    connection_id=conn_id,
    collection_name="FileUploadTest"
)
```

**Processing Time by File Type**:
- **TXT/MD**: 10-30 seconds
- **CSV/HTML**: 20-40 seconds
- **PDF (simple)**: 1-2 minutes
- **PDF (complex with images/tables)**: 2-3 minutes
- **PPTX**: 3-5 minutes
- **JPEG (standalone)**: 30-60 seconds

**Success Criteria**:
- ✅ All 9 files processed without errors
- ✅ Chunks saved to MinIO: `storage/{hash}/chunks/`
- ✅ Images extracted: `storage/{hash}/images/`
- ✅ Tables extracted: `storage/{hash}/tables/`
- ✅ Doc summaries created: `storage/{hash}/doc_summaries/`
- ✅ Data-manager logs show "Processing complete"

---

### Phase 7: Weaviate Ingestion

**Goal**: Ingest all chunks into multi-tenant Weaviate collections.

**Collection Architecture**:
```
Organization: Fileuploadtest20251117140000
├── FullText (multi-tenant)
│   └── Tenants:
│       └── conn_connupload20251117140000 (sanitized, hyphens removed)
├── Sentence (multi-tenant)
│   └── Tenants:
│       └── conn_connupload20251117140000
└── DocSummary (multi-tenant)
    └── Tenants:
        └── conn_connupload20251117140000
```

**DB Manager API Call**:
```bash
POST http://localhost:5000/database/document
Content-Type: application/json

{
  "organization_id": "fileupload-test-20251117140000",
  "connection_id": "conn-upload-20251117140000",
  "collection_name": "FileUploadTest",
  "chunks": [
    {
      "chunk_id": "chunk_001",
      "text": "This is the document text...",
      "embedding": [0.123, -0.456, ...],  # 384 dimensions
      "chunk_type": "text",
      "file_name": "document.pdf",
      "file_hash": "abc123...",
      "source_provider": "raw_documents",
      "permission_level": "private",
      "namespace": "e2e-test-workspace",
      "metadata": {
        "page_number": 1,
        "chunk_index": 0
      }
    }
    // ... more chunks
  ]
}
```

**What Happens**:
1. **Create/Get Collections**:
   - Check if `Fileuploadtest20251117140000_FullText` exists
   - If not, create with schema (text, embedding, metadata fields)
   - Repeat for `_Sentence` and `_DocSummary`

2. **Create/Get Tenant**:
   - Sanitize connection ID: `conn-upload-20251117140000` → `conn_connupload20251117140000`
   - Check if tenant exists in collection
   - If not, create tenant

3. **Batch Insert Objects**:
   - Insert FullText chunks (main document chunks)
   - Insert Sentence chunks (for precision retrieval)
   - Insert DocSummary chunks (document-level summaries)
   - Use batch size: 100 objects per request

**Weaviate Query Verification**:
```bash
# List all collections
curl http://localhost:8080/v1/schema | jq -r '.classes[].class' | grep Fileuploadtest

# Query FullText collection (MUST use limit!)
TENANT="conn_connupload20251117140000"
curl "http://localhost:8080/v1/objects?class=Fileuploadtest20251117140000_FullText&tenant=$TENANT&limit=100" | jq '.totalResults'

# Expected: 50-150 chunks (depends on file size)
```

**Success Criteria**:
- ✅ 3 collections created: `_FullText`, `_Sentence`, `_DocSummary`
- ✅ Tenant created: `conn_connupload20251117140000`
- ✅ Query returns > 0 objects (with `limit` parameter!)
- ✅ Sample object has correct metadata (org, connection, namespace)
- ✅ No ingestion errors in db-manager logs

---

### Phase 8: Verification & Monitoring

**Goal**: Validate complete end-to-end flow succeeded.

**8.1 - MinIO Structure Verification**:
```bash
# Check organization bucket
ls -R minio_mount/org-fileupload-test-20251117140000/

# Expected structure:
# uploads/
#   conn-conn-upload-20251117140000/
#     documents/
#       f61e8a479d2313f31bb638c98bb16803_photo_of_a_a4_format_with_text.jpeg
#       ... (9 files total)
#     images/
#       image_001.png
#       image_002.png
#       ... (extracted images from PDFs)
#     tables/
#       table_001.csv
#       ... (extracted tables from PDFs)
```

**8.2 - Weaviate Data Verification**:
```bash
# Find collections
COLLECTIONS=$(curl -s http://localhost:8080/v1/schema | jq -r '.classes[].class' | grep -i fileuploadtest)

# Query each collection
TENANT="conn_connupload20251117140000"

for COLLECTION in $COLLECTIONS; do
  echo "=== $COLLECTION ==="
  curl -s "http://localhost:8080/v1/objects?class=$COLLECTION&tenant=$TENANT&limit=100" | jq '{
    totalResults: .totalResults,
    objects: (.objects | length),
    sample: .objects[0].properties | {file_name, chunk_type, source_provider}
  }'
done
```

**8.3 - Spend Tracking Verification**:
```bash
# Organization spend
curl http://localhost:8100/models/analytics/spend/organization/fileupload-test-20251117140000 | jq '.'

# Expected:
{
  "organization_uuid": "fileupload-test-20251117140000",
  "total_spend": 0.05,  # Small amount from embeddings/LLM calls
  "spend_by_model": {
    "claude-37-sonnet-chat": 0.02,
    "claude-37-sonnet-vision": 0.01,
    "embedanything-minilm": 0.02
  }
}

# Team spend
curl http://localhost:8100/models/analytics/spend/team/team_xyz789 | jq '.'
```

**8.4 - RabbitMQ Queue Verification**:
```bash
# Check queue is empty (all processed)
docker exec rabbitmq-broker rabbitmqctl list_queues name messages consumers

# Expected:
# file_notifications  0  3
```

**8.5 - Service Logs Verification**:
```bash
# Data manager processing logs
for dm in horus_data-manager_1 horus_data-manager_2 horus_data-manager_3; do
  echo "=== $dm ==="
  docker logs $dm --since 5m | grep -E "(Processing file|Processing complete|Calling orchestrator)" | tail -5
done

# Expected: "Processing complete" for all 9 files
```

**Success Criteria**:
- ✅ MinIO bucket contains 9 uploaded files
- ✅ MinIO contains extracted images/tables
- ✅ Weaviate collections have > 0 objects
- ✅ Spend tracking shows non-zero usage
- ✅ RabbitMQ queue is empty (all messages processed)
- ✅ No errors in data-manager or db-manager logs

---

## Success KPIs Summary

### Critical KPIs (Must Pass)

| # | KPI | Expected Value | Validation Method |
|---|-----|----------------|-------------------|
| 1 | Organization created | `organization_id` not null | API response check |
| 2 | Embedding models auto-added | Exactly 3 models | Response array length |
| 3 | LLM models registered | 3 models (Claude chat, vision, OpenAI) | Model count check |
| 4 | Connection team created | `team_id` and `api_key` not null | API response check |
| 5 | Files uploaded | 9 files with status "queued" | Response file count |
| 6 | MinIO files stored | 9 files in org bucket | File count in MinIO |
| 7 | RabbitMQ messages queued | 9 messages → 0 after 30s | Queue count check |
| 8 | Document processing complete | All 9 files processed | Data-manager logs |
| 9 | Weaviate collections created | 3 collections (FullText, Sentence, DocSummary) | Schema query |
| 10 | Weaviate tenant created | Tenant ID exists | Tenant list check |
| 11 | Weaviate objects ingested | > 0 objects per collection | Query with `limit=100` |
| 12 | No processing errors | Zero errors in logs | Log grep for "ERROR" |
| 13 | Spend tracking initialized | Total spend > 0 | Analytics API |
| 14 | Queue empty | 0 messages in file_notifications | RabbitMQ status |
| 15 | Total execution time | < 5 minutes | Test duration check |

### Performance KPIs

| Metric | Expected Value | Notes |
|--------|----------------|-------|
| File upload latency | < 10 seconds for 9 files | Network + MD5 computation |
| MinIO save time | < 5 seconds | Depends on file sizes |
| RabbitMQ publish time | < 1 second | 9 messages |
| Processing time (TXT/MD) | 10-30 seconds | Simple files |
| Processing time (PDF complex) | 2-3 minutes | Images + tables |
| Processing time (PPTX) | 3-5 minutes | Largest file (15.7 MB) |
| Weaviate ingestion | 1-2 minutes | All collections |
| Total test duration | 2-3 minutes | End-to-end |

### Data Quality KPIs

| Metric | Expected Value | Validation |
|--------|----------------|------------|
| Chunk count (avg) | 10-20 per file | Depends on file length |
| Embedding dimension | 384 (embedanything-minilm) | Vector length |
| Tenant ID format | `conn_connupload{timestamp}` | Sanitized (no hyphens) |
| Collection name format | `Fileuploadtest{timestamp}_{Type}` | Sanitized |
| Permission level | "private" for all chunks | Metadata field |
| Source provider | "raw_documents" for all | Metadata field |
| Namespace | "e2e-test-workspace" for all | Metadata field |

---

## Common Failure Scenarios

### Failure 1: "Organization already exists"

**Symptom**: Setup script fails with organization creation error.

**Root Cause**: Timestamp-based UUID already used (script run twice in same second).

**Solution**:
```bash
# Delete existing organization
curl -X DELETE http://localhost:8100/models/organizations/fileupload-test-{timestamp}

# Or wait 1 second and re-run
```

---

### Failure 2: "No collections found" (Weaviate)

**Symptom**: Query returns 0 collections after 30s wait.

**Root Cause**: Processing still in progress, or data-manager crashed.

**Solution**:
```bash
# Check data-manager logs
docker logs horus_data-manager_1 --tail 50 | grep -i error

# If no errors, wait longer (processing complex PDFs)
sleep 60

# Then re-check
curl http://localhost:8080/v1/schema | jq -r '.classes[].class' | grep Fileuploadtest
```

---

### Failure 3: "Query returns 0 objects" (Weaviate)

**Symptom**: Collections exist but query returns `totalResults: 0`.

**Root Cause**: Missing `limit` parameter in query.

**Solution**:
```bash
# ❌ WRONG (returns 0)
curl "http://localhost:8080/v1/objects?class=Collection&tenant=conn_xxx"

# ✅ CORRECT (returns actual count)
curl "http://localhost:8080/v1/objects?class=Collection&tenant=conn_xxx&limit=100"
```

---

### Failure 4: "RabbitMQ queue stuck"

**Symptom**: Messages remain in queue after 5 minutes.

**Root Cause**: Data-manager consumers not running or crashed.

**Solution**:
```bash
# Check consumers
docker exec rabbitmq-broker rabbitmqctl list_queues name messages consumers

# If 0 consumers, restart data-managers
docker-compose restart data-manager

# Check logs for crash
docker logs horus_data-manager_1 --tail 100 | grep -i "error\|exception"
```

---

### Failure 5: "Files not in MinIO"

**Symptom**: Upload succeeds but files missing from MinIO.

**Root Cause**: File-server MinIO configuration issue.

**Solution**:
```bash
# Check MinIO health
curl http://localhost:9000/minio/health/live

# Check bucket exists
ls -la minio_mount/ | grep org-fileupload

# Restart file-server
docker-compose restart file-server
```

---

## Test Data Details

### File 1: `photo of a a4 format with text.jpeg`
- **Type**: Standalone image
- **Size**: 3.2 MB
- **Processing**: OCR (Tesseract) → text extraction → vision model description
- **Expected Chunks**: 1 text chunk + 1 image description
- **Processing Time**: 30-60 seconds

### File 2: `Relevé de situation_20250515 (1).pdf`
- **Type**: Financial PDF (2 pages)
- **Size**: 0.8 MB
- **Processing**: PDF → Markdown → chunking
- **Expected Chunks**: 3-5 chunks
- **Processing Time**: 1 minute

### File 3: `one sheet table.csv`
- **Type**: Simple CSV
- **Size**: 2 KB
- **Processing**: CSV → table extraction → summarization
- **Expected Chunks**: 1 table chunk
- **Processing Time**: 20 seconds

### File 4: `multi page financial with image and table embed.pdf`
- **Type**: Complex PDF (8 pages, images, tables)
- **Size**: 4.5 MB
- **Processing**: Full pipeline (images + tables + text)
- **Expected Chunks**: 10-15 chunks + 2-3 images + 1-2 tables
- **Processing Time**: 2-3 minutes

### File 5: `pelousi-annual-accounts-simple.html`
- **Type**: HTML document
- **Size**: 15 KB
- **Processing**: HTML → Markdown → chunking
- **Expected Chunks**: 2-3 chunks
- **Processing Time**: 30 seconds

### File 6: `scan pdf of full text page.pdf`
- **Type**: Scanned PDF (requires OCR)
- **Size**: 1.2 MB
- **Processing**: OCR (Tesseract) → text extraction → chunking
- **Expected Chunks**: 4-6 chunks
- **Processing Time**: 1-2 minutes

### File 7: `simple md file.md`
- **Type**: Markdown
- **Size**: 1 KB
- **Processing**: Direct chunking (already Markdown)
- **Expected Chunks**: 1 chunk
- **Processing Time**: 10 seconds

### File 8: `costs (1).csv`
- **Type**: CSV with special characters
- **Size**: 3 KB
- **Processing**: CSV → table extraction → summarization
- **Expected Chunks**: 1 table chunk
- **Processing Time**: 20 seconds

### File 9: `YCombinato-2a0506eb97 - Copie (1).pptx`
- **Type**: PowerPoint presentation (20 slides)
- **Size**: 15.7 MB
- **Processing**: PPTX → Markdown → images/tables → chunking
- **Expected Chunks**: 20-30 chunks + 5-10 images
- **Processing Time**: 3-5 minutes

---

## Next Steps After Test

1. **Verify data quality**: Query Weaviate and inspect chunk content
2. **Test RAG queries**: Use orchestrator to query uploaded documents
3. **Check Phoenix traces**: View processing traces at http://localhost:6006
4. **Cleanup**: Delete test organization to free resources
5. **Run test 02**: Google Drive E2E test

---

# Test 02: Google Drive E2E

## Overview

**Purpose**: Validate Airbyte CDC (Change Data Capture) integration with Google Drive, including artifact routing and reference file strategy.

**Location**: `e2e/02_google_drive_e2e/`
**Duration**: 3-4 minutes
**Files Synced**: 8 files from Google Drive folder
**Services Tested**: LiteLLM Wrapper, Airbyte Wrapper, File Server, Data Manager, DB Manager, MinIO, Weaviate

### What This Test Validates

This test ensures the Airbyte integration works correctly:

1. **Airbyte Source Creation**: Workspace, source, destination setup via wrapper
2. **Google Drive Connector**: Authentication, folder sync, file transfer mode
3. **Sync Job Execution**: Incremental sync, job monitoring, status polling
4. **CDC Ingestion**: Change detection, metadata tracking, reference file strategy
5. **Artifact Routing**: Images/tables routed to `airbyte/.../google-drive/` (not `uploads/`)
6. **MinIO Storage**: Organization-centric buckets with Airbyte-specific paths
7. **Weaviate Ingestion**: Multi-tenant storage with Airbyte connection tenant

---

## Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                   TEST 02: GOOGLE DRIVE E2E                         │
└─────────────────────────────────────────────────────────────────────┘

Phase 1: LiteLLM Wrapper Setup
───────────────────────────────
  User Script
      │
      ├─── POST /models/organizations/ ──→ LiteLLM Wrapper
      │    ←─── organization_id, models[] ───┘
      │
      ├─── POST /models/teams/ ──→ LiteLLM Wrapper
      │    ←─── team_id, api_key ──┘
      │
      ├─── POST /models/models/ (Chat) ──→ LiteLLM Wrapper
      ├─── POST /models/models/ (Vision) ──→ LiteLLM Wrapper
      │    ←─── model_ids ──────────────┘
      │
      ✓ SUCCESS: Org + Team + 5 models ready

Phase 2: Airbyte Source Creation
─────────────────────────────────
  User Script
      │
      ├─── POST /airbyte/create_source ──→ Airbyte Wrapper (4040)
      │    {                                     │
      │      workspace_name: "gdrive-test-..."  │
      │      user_source_name: "Google Drive"   ├─ Create workspace
      │      source_config: {                    ├─ Create source (Google Drive)
      │        source_type: "google-drive"       ├─ Create destination (Horus)
      │        folder_url: "https://..."         ├─ Create connection
      │        credentials: {...}                │
      │      }                                    │
      │    }                                      │
      │    ←─── workspace_id, source_id, ────────┘
      │         destination_id, connection_id
      │
      ✓ SUCCESS: Airbyte entities created

Phase 3: Sync Job Execution
────────────────────────────
  User Script
      │
      ├─── POST /airbyte/start_job_sync ──→ Airbyte Wrapper
      │    {                                     │
      │      workspace_id,                       │
      │      source_id,                          │
      │      destination_id,                     ├─ Start sync job
      │      source_type: "google-drive"         │   (Airbyte native)
      │      sync_mode: "incremental_append"     │
      │    }                                      │
      │    ←─── job_id ────────────────────────┘
      │
      ├─── Poll job status (30x, every 5s)
      │    POST /airbyte/job_status
      │    {job_id: "..."}
      │
      │    ←─── status: "running" → "succeeded"
      │
      ✓ SUCCESS: 8 files synced to MinIO

  (Background: Airbyte transfers files)
      │
      ├─ Airbyte Worker downloads from Google Drive
      ├─ Saves to MinIO: org-{uuid}/airbyte/conn-{airbyte_conn}/google-drive/items/{item_id}/
      ├─ Creates metadata: daily_results_{date}.json
      │
      ✓ FILES IN MINIO: 8 files

Phase 4: CDC Ingestion
───────────────────────
  User Script
      │
      ├─── POST /airbyte/ingest_data ──→ Airbyte Wrapper (4040)
      │    {                                    │
      │      connection_id: "{airbyte_conn}"   │
      │      organization_id,                   │
      │      connection_uuid: "{our_conn}",     ├─ Detect CDC changes:
      │      source_type: "google-drive",       │   • New files (created)
      │      namespace: [...],                  │   • Updated files (modified)
      │      collection_name: "GoogleDriveTest",│   • Deleted files (removed)
      │      llm_models: [...],                 │
      │      custom_parameters: {               ├─ For each file:
      │        table_mode: "auto",              │   → Call file-server:
      │        ocr_mode: "auto"                 │     POST /file/upload-documents-airbyte
      │      }                                   │
      │    }                                     │
      │    ←─── summary: {created, updated} ────┘
      │
      ✓ SUCCESS: CDC detected 8 new files

  Airbyte Wrapper → File Server (for each file)
      │
      ├─── POST /file/upload-documents-airbyte ──→ File Server
      │    {                                            │
      │      files: [{                                  │
      │        source_name: "doc.pdf",                  │
      │        source_uri: "minio://org-.../doc.pdf",  │
      │        item_id: "abc123"                        │
      │      }],                                        │
      │      organization_id,                           │
      │      connection_id,                             ├─ Reference file strategy:
      │      source_provider: "google-drive",           │   • Don't re-upload to MinIO
      │      airbyte_connection_id,                     │   • Use existing MinIO path
      │      ... (standard params)                      │   • Publish to RabbitMQ with
      │    }                                             │     MinIO reference URL
      │    ←─── files[] (status: queued) ───────────────┘
      │
      ✓ SUCCESS: 8 files queued for processing

Phase 5: Document Processing
─────────────────────────────
  RabbitMQ → Data Manager (same as Test 01)
      │
      ├─ Download from MinIO (reference URL)
      ├─ Docling processing (13-stage pipeline)
      ├─ Extract artifacts:
      │   → Images: org-{uuid}/airbyte/conn-{airbyte_conn}/google-drive/images/
      │   → Tables: org-{uuid}/airbyte/conn-{airbyte_conn}/google-drive/tables/
      ├─ Generate chunks, embeddings, summaries
      ├─ Send to DB Manager
      │
      ✓ SUCCESS: All files processed

Phase 6: Weaviate Ingestion & Verification
───────────────────────────────────────────
  DB Manager → Weaviate
      │
      ├─ Create collections: Googledriveetest_{FullText, Sentence, DocSummary}
      ├─ Create tenant: conn_{airbyte_connection_id} (sanitized)
      ├─ Insert chunks
      │
      ✓ SUCCESS: Data in Weaviate

  Test Script → Verification
      │
      ├─ Check MinIO artifact routing:
      │   ✓ Files in: airbyte/conn-.../google-drive/items/
      │   ✓ Images in: airbyte/conn-.../google-drive/images/
      │   ✓ Tables in: airbyte/conn-.../google-drive/tables/
      │   ✓ NOT in: uploads/ (WRONG location)
      │
      ├─ Query Weaviate:
      │   ✓ Collections created
      │   ✓ Tenant exists
      │   ✓ Objects ingested (totalResults > 0)
      │
      ✓ SUCCESS: All validations passed
```

---

## Phase-by-Phase Breakdown

### Phase 1: LiteLLM Wrapper Setup

**Goal**: Create organization, connection team, and LLM models (same as Test 01, but with Google Drive context).

**Key Differences from Test 01**:
- Organization alias: "Google Drive E2E Test"
- Connection team metadata: `connection_type: "google-drive"`
- Only 2 LLM models: chat + vision (no separate OpenAI model)

**API Calls**: Same as Test 01, phases 1-3.

**Success Criteria**:
- ✅ Organization created: `gdrive-test-{timestamp}`
- ✅ Connection team created: `conn-gdrive-{timestamp}`
- ✅ 5 models total: 3 embedding + 2 LLM (chat, vision)

---

### Phase 2: Airbyte Source Creation

**Goal**: Create Airbyte workspace, Google Drive source, Horus destination, and connection via wrapper.

**API Call**:
```bash
POST http://localhost:4040/airbyte/create_source
Content-Type: application/json

{
  "workspace_name": "gdrive-test-20251117150000",
  "user_source_name": "Google Drive E2E Test 20251117150000",
  "source_config": {
    "source_type": "google-drive",
    "configuration": {
      "sourceType": "google-drive",
      "streams": [
        {
          "name": "google_drive",
          "globs": ["**"],  # All files
          "format": {
            "filetype": "unstructured",
            "strategy": "auto",
            "processing": {"mode": "local"},
            "skip_unprocessable_files": true
          },
          "schemaless": false,
          "validation_policy": "Emit Record",
          "days_to_sync_if_history_is_full": 3
        }
      ],
      "folder_url": "https://drive.google.com/drive/folders/1cwka1_j0D_bJNG-DLRE6CKJRn2UoF60D",
      "credentials": {
        "auth_type": "Service",
        "service_account_info": "{...json...}"  # Google service account
      },
      "delivery_method": {
        "delivery_type": "use_file_transfer",
        "preserve_directory_structure": true
      }
    }
  }
}
```

**Expected Response**:
```json
{
  "workspaceId": "e76bf322-aff5-4d17-b60a-6101f02571e5",
  "sourceId": "66ede919-0035-4e18-b929-795a4379ae7c",
  "destinationId": "97acaaa2-5429-453e-a54f-bfcd92d942b2",
  "connectionId": "43c64530-0351-4e57-b245-fe30954cd7f6",
  "status": "success"
}
```

**What Happens**:
1. **Airbyte Wrapper** receives request
2. **Creates Airbyte Workspace**:
   - Calls Airbyte API: `POST /v1/workspaces`
   - Name: `gdrive-test-{timestamp}`
3. **Creates Google Drive Source**:
   - Calls Airbyte API: `POST /v1/sources`
   - Source type: `google-drive`
   - Credentials: service account JSON
4. **Creates Horus Destination** (file transfer):
   - Type: `file-transfer` (local MinIO)
   - Path: `minio_mount/org-{org_uuid}/airbyte/`
5. **Creates Airbyte Connection**:
   - Links source → destination
   - Sync mode: `incremental_append`
   - Schedule: manual (triggered by wrapper)

**Success Criteria**:
- ✅ All 4 IDs returned (workspace, source, destination, connection)
- ✅ Response status: "success"
- ✅ Response time < 10 seconds

---

### Phase 3: Sync Job Execution & Monitoring

**Goal**: Trigger Airbyte sync job and poll until completion.

**3.1 - Start Sync Job**:
```bash
POST http://localhost:4040/airbyte/start_job_sync
Content-Type: application/json

{
  "workspace_id": "e76bf322-aff5-4d17-b60a-6101f02571e5",
  "source_id": "66ede919-0035-4e18-b929-795a4379ae7c",
  "destination_id": "97acaaa2-5429-453e-a54f-bfcd92d942b2",
  "source_type": "google-drive",
  "selected_streams": ["google_drive"],
  "sync_mode": "incremental_append"
}
```

**Expected Response**:
```json
{
  "connectionId": "43c64530-0351-4e57-b245-fe30954cd7f6",
  "jobId": "915",
  "status": "running"
}
```

**3.2 - Poll Job Status** (every 5 seconds, max 30 times):
```bash
POST http://localhost:4040/airbyte/job_status
Content-Type: application/json

{
  "job_id": "915"
}
```

**Status Progression**:
```
Check 1/30:  status=pending,   rows=0,    bytes=0
Check 2/30:  status=running,   rows=0,    bytes=0
Check 3/30:  status=running,   rows=0,    bytes=1024
Check 4/30:  status=running,   rows=0,    bytes=2048
...
Check 20/30: status=succeeded, rows=0,    bytes=26214400  # 25 MB
```

**Final Response**:
```json
{
  "jobId": "915",
  "status": "succeeded",
  "rowsEmitted": 0,         # File transfer mode (no rows)
  "bytesEmitted": 26214400, # 25 MB transferred
  "completedAt": "2025-11-17T15:02:30Z"
}
```

**What Happens**:
1. **Airbyte Worker** starts sync job
2. **Connects to Google Drive** using service account
3. **Downloads files** from folder (8 files):
   - `costs.csv`
   - `YCombinato-2a0506eb97 - Copie.pptx`
   - `multi page financial with image and table embed.pdf`
   - `Relevé de situation_20250515 (1).pdf`
   - `scan pdf of full text page.pdf`
   - `one sheet table.csv`
   - `pelousi-annual-accounts-simple.html`
   - `CODE INSCRIPTION IMPOTS.jpeg`
4. **Saves to MinIO**:
   - Path: `minio_mount/org-gdrive-test-{timestamp}/airbyte/conn-{connection_id}/google-drive/`
   - Structure:
     ```
     items/
       {item_id_1}/
         {filename}.pdf
       {item_id_2}/
         {filename}.csv
     daily_results_2025-11-17.json  # Metadata file
     ```
5. **Job completes** with status: "succeeded"

**Success Criteria**:
- ✅ Job status transitions: pending → running → succeeded
- ✅ Final status: "succeeded" (not "failed")
- ✅ Bytes emitted > 0
- ✅ Poll completes within 150 seconds (30 checks * 5s)
- ✅ 8 files visible in MinIO

---

### Phase 4: CDC Ingestion

**Goal**: Trigger change detection and route new/updated files to file-server for processing.

**API Call**:
```bash
POST http://localhost:4040/airbyte/ingest_data
Content-Type: application/json

{
  "connection_id": "43c64530-0351-4e57-b245-fe30954cd7f6",  # Airbyte connection
  "organization_id": "gdrive-test-20251117150000",
  "connection_uuid": "conn-gdrive-20251117150000",  # Our connection (not Airbyte's)
  "source_type": "google-drive",
  "namespace": ["gdrive-e2e-test"],
  "collection_name": "GoogleDriveTest",
  "source_provider": "google-drive",
  "permission_level": "private",
  "embedding_model": "embedanything-minilm",
  "llm_models": [
    {"model_id": "gpt4mini-chat-{timestamp}", "capabilities": ["chat"]},
    {"model_id": "gpt4mini-vision-{timestamp}", "capabilities": ["chat", "vision"]}
  ],
  "custom_parameters": {
    "table_mode": "auto",
    "ocr_mode": "auto",
    "includes_images": true
  }
}
```

**Expected Response**:
```json
{
  "summary": {
    "created_count": 8,   # 8 new files detected
    "updated_count": 0,   # 0 files modified
    "deleted_count": 0,   # 0 files deleted
    "total_changes": 8
  },
  "files_processed": [
    {
      "source_name": "costs.csv",
      "item_id": "1abc123",
      "status": "queued",
      "change_type": "created"
    },
    // ... 7 more files
  ]
}
```

**What Happens**:

**4.1 - CDC Detection** (Airbyte Wrapper):
```python
# Airbyte wrapper reads sync metadata
metadata_file = f"minio://org-{org_id}/airbyte/conn-{airbyte_conn}/google-drive/daily_results_{today}.json"
sync_results = load_json(metadata_file)

# Detect changes
changes = {
    "created": [],   # New files not in previous sync
    "updated": [],   # Modified files (changed metadata)
    "deleted": []    # Removed files (in previous sync, not in current)
}

for item in sync_results['items']:
    if item['id'] not in previous_sync:
        changes['created'].append(item)
    elif item['modified_time'] > previous_sync[item['id']]['modified_time']:
        changes['updated'].append(item)

for prev_item in previous_sync:
    if prev_item['id'] not in current_sync:
        changes['deleted'].append(prev_item)
```

**4.2 - File Server Calls** (Airbyte Wrapper → File Server):

For each file (8 calls):
```bash
POST http://localhost:8020/file/upload-documents-airbyte
Content-Type: application/json

{
  "files": [{
    "source_name": "costs.csv",
    "source_uri": "minio://org-gdrive-test-{timestamp}/airbyte/conn-{airbyte_conn}/google-drive/items/1abc123/costs.csv",
    "item_id": "1abc123",
    "modified_time": "2025-11-17T10:30:00Z"
  }],
  "organization_id": "gdrive-test-20251117150000",
  "connection_id": "conn-gdrive-20251117150000",
  "namespace": "gdrive-e2e-test",
  "collection_name": "GoogleDriveTest",
  "source_provider": "google-drive",
  "permission_level": "private",
  "airbyte_connection_id": "43c64530-0351-4e57-b245-fe30954cd7f6",
  "llm_models": [...],
  "embedding_model": "embedanything-minilm",
  "table_mode": "auto",
  "ocr_mode": "auto",
  "includes_images": true
}
```

**4.3 - Reference File Strategy** (File Server):
```python
# File server receives upload request
for file in request.files:
    if file['source_uri'].startswith('minio://'):
        # File already in MinIO (from Airbyte sync)
        # Don't re-upload, just use existing path
        file_location = parse_minio_uri(file['source_uri'])

        # Publish to RabbitMQ with reference URL
        publish_message({
            'file_hash': compute_hash_from_minio(file_location),
            'encrypted_url': encrypt_minio_path(file_location),  # Reference to existing file
            'organization_id': org_id,
            'connection_id': conn_id,
            'source_provider': 'google-drive',
            # ... other metadata
        })
```

**Success Criteria**:
- ✅ CDC detects 8 new files (created_count: 8)
- ✅ All 8 files queued for processing
- ✅ File server uses reference strategy (no re-upload)
- ✅ RabbitMQ receives 8 messages

---

### Phase 5: Artifact Routing Verification

**Goal**: Ensure images/tables are routed to `airbyte/.../google-drive/` folder (NOT `uploads/`).

**Expected MinIO Structure**:
```
org-gdrive-test-20251117150000/
├── airbyte/
│   └── conn-43c64530-0351-4e57-b245-fe30954cd7f6/
│       └── google-drive/
│           ├── items/              # Original files from Airbyte
│           │   ├── 1abc123/
│           │   │   └── costs.csv
│           │   ├── 2def456/
│           │   │   └── document.pdf
│           │   └── ... (8 item folders)
│           │
│           ├── images/             # Extracted images (from data-manager)
│           │   ├── image_001.png
│           │   ├── image_002.png
│           │   └── ... (5-10 images)
│           │
│           ├── tables/             # Extracted tables (from data-manager)
│           │   ├── table_001.csv
│           │   └── ... (2-3 tables)
│           │
│           └── daily_results_2025-11-17.json  # Sync metadata
│
└── uploads/                        # Should be EMPTY (or have other uploads, not Google Drive)
    └── (NONE - Google Drive files go to airbyte/ folder)
```

**Validation Commands**:
```bash
ORG_BUCKET="org-gdrive-test-20251117150000"

# Check images in correct location
IMAGE_COUNT=$(find "minio_mount/$ORG_BUCKET" -path "*/google-drive/images/*" -type f | wc -l)
echo "✓ Images found: $IMAGE_COUNT"

# Check tables in correct location
TABLE_COUNT=$(find "minio_mount/$ORG_BUCKET" -path "*/google-drive/tables/*" -type f | wc -l)
echo "✓ Tables found: $TABLE_COUNT"

# Check WRONG location (should be 0)
WRONG_COUNT=$(find "minio_mount/$ORG_BUCKET/uploads" -type f | wc -l)
if [ "$WRONG_COUNT" -gt 0 ]; then
  echo "❌ WARNING: Found $WRONG_COUNT files in uploads/ folder (WRONG!)"
else
  echo "✓ No files in uploads/ folder (CORRECT!)"
fi
```

**Success Criteria**:
- ✅ Images found in: `airbyte/conn-.../google-drive/images/`
- ✅ Tables found in: `airbyte/conn-.../google-drive/tables/`
- ✅ Zero files in: `uploads/` folder
- ✅ Original files in: `airbyte/conn-.../google-drive/items/`

---

### Phase 6: Weaviate Verification

**Goal**: Verify Weaviate collections and tenant created, data ingested.

**6.1 - Find Collections**:
```bash
curl -s http://localhost:8080/v1/schema | jq -r '.classes[].class' | grep -i googledrive

# Expected collections:
# Googledriveetest_FullText
# Googledriveetest_Sentence
# Googledriveetest_DocSummary
```

**6.2 - Query Data**:
```bash
# Sanitize tenant ID (remove hyphens from Airbyte connection ID)
AIRBYTE_CONN_ID="43c64530-0351-4e57-b245-fe30954cd7f6"
TENANT_ID="conn_${AIRBYTE_CONN_ID//-/_}"  # conn_43c6453003514e57b245fe30954cd7f6

# Query FullText collection (MUST use limit!)
curl -s "http://localhost:8080/v1/objects?class=Googledriveetest_FullText&tenant=$TENANT_ID&limit=100" | jq '{
  totalResults: .totalResults,
  objects: (.objects | length),
  sample: .objects[0].properties | {
    file_name,
    chunk_type,
    source_provider,
    source_uri
  }
}'
```

**Expected Response**:
```json
{
  "totalResults": 45,  # 8 files → ~45 chunks
  "objects": 45,
  "sample": {
    "file_name": "costs.csv",
    "chunk_type": "table",
    "source_provider": "google-drive",
    "source_uri": "minio://org-gdrive-test-.../costs.csv"
  }
}
```

**Success Criteria**:
- ✅ 3 collections found (FullText, Sentence, DocSummary)
- ✅ Tenant ID sanitized correctly (hyphens removed)
- ✅ Query returns > 0 objects (totalResults > 0)
- ✅ Sample object has source_provider: "google-drive"
- ✅ Sample object has source_uri pointing to airbyte/ folder

---

## Success KPIs Summary

### Critical KPIs (Must Pass)

| # | KPI | Expected Value | Validation Method |
|---|-----|----------------|-------------------|
| 1 | Airbyte workspace created | `workspaceId` not null | API response |
| 2 | Google Drive source created | `sourceId` not null | API response |
| 3 | Horus destination created | `destinationId` not null | API response |
| 4 | Airbyte connection created | `connectionId` not null | API response |
| 5 | Sync job started | `jobId` not null | API response |
| 6 | Sync job succeeded | `status: "succeeded"` | Poll job status |
| 7 | Files synced to MinIO | 8 files in airbyte/ folder | MinIO file count |
| 8 | CDC detected changes | `created_count: 8` | Ingestion response |
| 9 | Files queued for processing | 8 files with status "queued" | File server response |
| 10 | Artifact routing correct | Images/tables in airbyte/google-drive/ | MinIO path check |
| 11 | No files in uploads/ | 0 files in uploads/ folder | MinIO path check |
| 12 | Weaviate collections created | 3 collections (FullText, Sentence, DocSummary) | Schema query |
| 13 | Weaviate tenant created | Tenant with sanitized ID | Tenant list |
| 14 | Weaviate data ingested | > 0 objects per collection | Query with limit |
| 15 | Total execution time | < 5 minutes | Test duration |

### Google Drive Specific KPIs

| Metric | Expected Value | Validation |
|--------|----------------|------------|
| Google Drive authentication | Service account valid | Sync job succeeds |
| Folder access | Folder readable | Files downloaded |
| File transfer mode | preserve_directory_structure: true | MinIO structure |
| Incremental sync | CDC detects only new/modified | created_count matches |
| Metadata tracking | daily_results JSON created | File exists in MinIO |
| Reference file strategy | No re-upload to MinIO | Upload-documents-airbyte uses source_uri |
| Artifact folder routing | Images/tables in google-drive/ subfolder | Path check |

---

## Common Failure Scenarios

### Failure 1: "Google Drive authentication failed"

**Symptom**: Sync job fails with "invalid_grant" or "permission_denied" error.

**Root Cause**: Service account credentials invalid or folder not shared.

**Solution**:
```bash
# Check service account JSON is valid
cat service_account.json | jq '.'

# Verify folder is shared with service account email
# 1. Open Google Drive folder
# 2. Share with: 1096946420084-compute@developer.gserviceaccount.com
# 3. Role: Viewer or Editor
```

---

### Failure 2: "Sync job stuck in 'running' status"

**Symptom**: Job status remains "running" for > 5 minutes.

**Root Cause**: Large files downloading slowly, or Airbyte worker issue.

**Solution**:
```bash
# Check Airbyte worker logs
docker logs airbyte-worker --tail 100

# Increase poll timeout (wait longer)
# Normal for large files (15.7 MB PPTX takes ~2 minutes)

# If truly stuck, restart Airbyte
docker-compose -f docker-compose.airbyteserver.yaml restart
```

---

### Failure 3: "Files in wrong location (uploads/ instead of airbyte/)"

**Symptom**: Artifact routing validation fails.

**Root Cause**: File server not using reference file strategy, or source_provider incorrect.

**Solution**:
```bash
# Check source_provider in upload request
grep "source_provider" /tmp/upload_request.json
# Should be: "google-drive" (not "raw_documents")

# Check file server code: upload-documents-airbyte endpoint
# Should use reference strategy for minio:// URIs
```

---

### Failure 4: "CDC ingestion returns created_count: 0"

**Symptom**: No files detected by CDC, even though sync succeeded.

**Root Cause**: Metadata file (daily_results) not created or corrupted.

**Solution**:
```bash
# Check metadata file exists
ls -la minio_mount/org-gdrive-test-*/airbyte/conn-*/google-drive/daily_results_*.json

# If missing, re-run sync job
curl -X POST http://localhost:4040/airbyte/start_job_sync -d '{...}'

# If corrupted, delete and re-sync
rm minio_mount/org-gdrive-test-*/airbyte/conn-*/google-drive/daily_results_*.json
```

---

## Files Synced from Google Drive

### Folder: "E2E Test Files"
**URL**: `https://drive.google.com/drive/folders/1cwka1_j0D_bJNG-DLRE6CKJRn2UoF60D`

1. `costs.csv` (3 KB) - Simple CSV table
2. `YCombinato-2a0506eb97 - Copie.pptx` (15.7 MB) - PowerPoint presentation
3. `multi page financial with image and table embed.pdf` (4.5 MB) - Complex PDF
4. `Relevé de situation_20250515 (1).pdf` (0.8 MB) - Financial PDF
5. `scan pdf of full text page.pdf` (1.2 MB) - Scanned PDF (OCR)
6. `one sheet table.csv` (2 KB) - CSV table
7. `pelousi-annual-accounts-simple.html` (15 KB) - HTML document
8. `CODE INSCRIPTION IMPOTS.jpeg` (3.2 MB) - Standalone image

**Total Size**: ~25 MB

---

## Next Steps After Test

1. **Verify artifact routing**: Check images/tables are in `airbyte/google-drive/` folder
2. **Test incremental sync**: Modify a file in Google Drive, re-run sync, verify CDC detects update
3. **Test file deletion**: Delete a file in Google Drive, re-run sync, verify CDC detects deletion
4. **Query Weaviate**: Perform RAG queries on Google Drive documents
5. **Cleanup**: Delete Airbyte workspace and test organization

---

# Test 03: File Server Full Suite

## Overview

**Purpose**: Comprehensive parameter coverage testing with 23 test cases validating file type handling, processing modes, and file operations.

**Location**: `e2e/03_file_server_full_suite/`
**Duration**: 30-45 minutes (if all 23 tests run)
**Test Cases**: 23 comprehensive scenarios
**Current Implementation**: Master test runner framework with 16 upload tests + 7 operation tests

### What This Test Validates

This test suite provides comprehensive coverage of:

1. **Parameter Variations**: All combinations of `includes_images`, `table_mode`, `ocr_mode`
2. **File Type Handling**: TXT, PDF, XLSX, CSV, JPEG, HTML, MD, PPTX
3. **Processing Modes**: disabled, fast, accurate, auto
4. **Special Cases**: Excel multi-sheet, CSV encoding, standalone images
5. **File Operations**: Upload, get URL, update metadata, delete
6. **Batch Operations**: Multiple files, deduplication
7. **Permission Levels**: private, restricted, public

---

## Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│              TEST 03: FILE SERVER FULL SUITE                        │
└─────────────────────────────────────────────────────────────────────┘

Master Test Runner
──────────────────
  User Script: run_test.sh
      │
      ├─ Load configuration from e2e/setup/test_config.env
      ├─ Check prerequisites (organization, connection setup)
      │
      ├─ Define test categories:
      │   • Upload & Processing Tests (01-15)
      │   • File Operation Tests (16-22)
      │   • Special Scenario Tests (23)
      │
      ├─ Execute tests sequentially:
      │   For each test case:
      │     ├─ Run test script: test_cases/{NN}_*/test.sh
      │     ├─ Capture output: reports/{timestamp}/test_{NN}.log
      │     ├─ Record result: PASSED / FAILED / SKIPPED
      │     └─ Wait 5s (avoid race conditions)
      │
      └─ Generate summary report: reports/summary.md

Test Case Flow (Individual Test)
─────────────────────────────────
  Test Script (e.g., 01_basic_upload/test.sh)
      │
      ├─── Phase 1: Upload File
      │    POST /file/upload-documents
      │    • file: test file
      │    • parameters: specific to test case
      │    • organization_id, connection_id from config
      │
      ├─── Phase 2: Verify MinIO
      │    Check file exists:
      │    minio_mount/org-{uuid}/uploads/conn-{uuid}/documents/{file}
      │
      ├─── Phase 3: Monitor Processing
      │    • Check RabbitMQ queue
      │    • Check data-manager logs
      │    • Wait for processing (30-120s)
      │
      ├─── Phase 4: Verify Weaviate
      │    • Query collection with tenant ID
      │    • Check totalResults > 0 (with limit!)
      │    • Verify chunk metadata
      │
      ├─── Phase 5: Verify Parameters
      │    • Check table_mode applied correctly
      │    • Check ocr_mode applied correctly
      │    • Check includes_images honored
      │
      └─── Report: PASS or FAIL with details

Summary Report Generation
──────────────────────────
  Master Test Runner
      │
      ├─ Collect results from all test cases
      ├─ Calculate metrics:
      │   • Total tests: 23
      │   • Passed: X
      │   • Failed: Y
      │   • Skipped: Z
      │   • Success rate: X/23 * 100%
      │
      ├─ Generate summary.md:
      │   • Overall results table
      │   • Results by category
      │   • Parameter coverage matrix
      │   • Validation coverage checklist
      │   • Issues found (if any)
      │   • Recommendations
      │
      └─ Copy to reports/summary.md
```

---

## Test Categories

### Category 1: Upload & Processing Tests (01-15)

**Test 01: Basic Upload**
- **File**: simple_text.txt
- **Parameters**: Default (table_mode: auto, ocr_mode: auto, includes_images: 1)
- **Validates**: Basic upload flow, MinIO storage, Weaviate ingestion

**Test 02: No Images Processing**
- **File**: document.pdf
- **Parameters**: includes_images: 0
- **Validates**: Images not extracted when disabled

**Test 03: With Images Processing**
- **File**: document_with_images.pdf
- **Parameters**: includes_images: 1
- **Validates**: Images extracted and described by vision model

**Test 04: Table Mode - Disabled**
- **File**: document_with_tables.pdf
- **Parameters**: table_mode: disabled
- **Validates**: Tables not extracted

**Test 05: Table Mode - Fast**
- **File**: document_with_tables.pdf
- **Parameters**: table_mode: fast
- **Validates**: Basic table extraction (no LLM summarization)

**Test 06: Table Mode - Accurate**
- **File**: document_with_tables.pdf
- **Parameters**: table_mode: accurate
- **Validates**: Full table extraction with LLM summarization

**Test 07: Table Mode - Auto**
- **File**: document_with_tables.pdf
- **Parameters**: table_mode: auto
- **Validates**: Automatic table detection and extraction

**Test 08: OCR Mode - Disabled**
- **File**: scanned_document.pdf
- **Parameters**: ocr_mode: disabled
- **Validates**: No OCR, scanned PDF returns minimal text

**Test 09: OCR Mode - Enabled**
- **File**: scanned_document.pdf
- **Parameters**: ocr_mode: enabled
- **Validates**: Tesseract OCR extracts text from scanned image

**Test 10: OCR Mode - Auto**
- **File**: scanned_document.pdf
- **Parameters**: ocr_mode: auto
- **Validates**: Automatic OCR when needed

**Test 11: Excel Multi-Sheet**
- **File**: workbook.xlsx (3 sheets)
- **Parameters**: table_mode: auto
- **Validates**: All sheets extracted as separate tables

**Test 12: CSV File**
- **File**: data.csv
- **Parameters**: table_mode: auto
- **Validates**: CSV parsed as table, summarized

**Test 13: Standalone Image**
- **File**: photo.jpeg
- **Parameters**: includes_images: 1
- **Validates**: Image uploaded, OCR + vision description

**Test 14: HTML Document**
- **File**: webpage.html
- **Parameters**: Default
- **Validates**: HTML → Markdown conversion

**Test 15: PowerPoint**
- **File**: presentation.pptx
- **Parameters**: includes_images: 1, table_mode: auto
- **Validates**: Slides processed, images/tables extracted

---

### Category 2: File Operation Tests (16-22)

**Test 16: Get File URL**
- **Validates**: Presigned URL generation, URL expiration (default 3600s)

**Test 17: Update File Metadata**
- **Validates**: Update namespace, permission_level, tags

**Test 18: Delete Single File**
- **Validates**: File removed from MinIO + Weaviate

**Test 19: Delete Multiple Files**
- **Validates**: Batch deletion

**Test 20: Duplicate File Upload** (Deduplication)
- **Validates**: Same file (same MD5 hash) not re-processed

**Test 21: Batch Upload**
- **Validates**: Multiple files uploaded in single request

**Test 22: Permission Levels**
- **Validates**: private, restricted, public levels stored correctly

---

### Category 3: Special Scenarios (23)

**Test 23: Webhook Notification**
- **Validates**: Webhook called on processing completion

---

## Parameter Coverage Matrix

| Parameter | Values Tested | Test Cases |
|-----------|---------------|------------|
| `includes_images` | 0, 1 | 02, 03, 13, 15 |
| `table_mode` | disabled, fast, accurate, auto | 04, 05, 06, 07, 11, 12 |
| `ocr_mode` | disabled, enabled, auto | 08, 09, 10, 13 |
| File Types | TXT, PDF, XLSX, CSV, JPEG, HTML, PPTX | 01-15 |
| Special Cases | Multi-sheet Excel, standalone image, scanned PDF | 11, 12, 13 |
| Operations | Upload, get URL, update, delete | 16-19 |
| Batch Operations | Multiple files, deduplication | 20, 21 |
| Permission Levels | private, restricted, public | 22 |

---

## Master Test Runner

**Script**: `e2e/03_file_server_full_suite/run_test.sh`

**Key Features**:
1. **Configuration Loading**: Loads `e2e/setup/test_config.env` for org/connection
2. **Sequential Execution**: Runs tests one by one (avoids race conditions)
3. **Result Tracking**: Tracks PASSED/FAILED/SKIPPED for each test
4. **Timestamped Reports**: Creates `reports/{timestamp}/` folder for each run
5. **Summary Generation**: Creates markdown summary with pass/fail table
6. **Error Handling**: Continues on failure, reports all results

**Usage**:
```bash
cd e2e/03_file_server_full_suite
bash run_test.sh

# View summary
cat reports/summary.md

# View specific test log
cat reports/{timestamp}/test_01.log
```

**Expected Output**:
```
=========================================
E2E File Server Test Suite
=========================================
Start Time: 2025-11-17 16:00:00

Configuration:
  Organization: e2e-fileserver-1763419471
  Connection: connection-1763419471
  Model: gpt4mini-1763419471

=========================================
Phase 1: Upload & Processing Tests (01-15)
=========================================

=========================================
Running Test 01: 01_basic_upload
=========================================
✅ PASSED: 01_basic_upload

=========================================
Running Test 02: 02_no_images
=========================================
✅ PASSED: 02_no_images

... (13 more tests)

=========================================
Phase 2: File Operation Tests (16-22)
=========================================

=========================================
Running Test 16: 16_get_file_url
=========================================
✅ PASSED: 16_get_file_url

... (6 more tests)

=========================================
Test Suite Complete
=========================================
End Time: 2025-11-17 16:35:00

Results:
  Total:   23
  ✅ Passed:  23
  ❌ Failed:  0
  ⚠️ Skipped: 0

Reports:
  Summary: e2e/03_file_server_full_suite/reports/summary.md
  Details: e2e/03_file_server_full_suite/reports/20251117_160000/

✅ ALL TESTS PASSED
```

---

## Success KPIs Summary

### Overall Suite KPIs

| Metric | Target Value | Validation |
|--------|--------------|------------|
| Total tests | 23 | Test count |
| Pass rate | 100% (23/23) | All tests pass |
| Execution time | < 45 minutes | Full suite duration |
| Coverage | 100% parameter combinations | Matrix completion |

### Per-Test KPIs

Each test must satisfy:
- ✅ File uploaded to MinIO
- ✅ RabbitMQ message processed
- ✅ Document processing completed (data-manager logs)
- ✅ Weaviate collection created
- ✅ Weaviate objects ingested (totalResults > 0)
- ✅ Parameters applied correctly (verified in chunk metadata)
- ✅ No errors in service logs

### Parameter Validation KPIs

| Parameter | Expected Behavior | Validation |
|-----------|-------------------|------------|
| `includes_images: 0` | No images extracted | Image count = 0 |
| `includes_images: 1` | Images extracted + described | Image count > 0, descriptions exist |
| `table_mode: disabled` | No tables extracted | Table count = 0 |
| `table_mode: fast` | Tables extracted, no LLM summary | Tables exist, summary = null |
| `table_mode: accurate` | Tables extracted + LLM summary | Tables exist, summary != null |
| `table_mode: auto` | Automatic detection | Tables extracted if detected |
| `ocr_mode: disabled` | No OCR on scanned PDFs | Minimal text extracted |
| `ocr_mode: enabled` | Tesseract OCR applied | Text extracted from scanned images |
| `ocr_mode: auto` | OCR when needed | OCR applied to scanned PDFs only |

---

## Summary Report Structure

**File**: `e2e/03_file_server_full_suite/reports/summary.md`

**Sections**:
1. **Overall Results**: Pass/fail counts, success rate
2. **Test Results by Category**: Table of test results
3. **Parameter Coverage**: Matrix of tested parameters
4. **Validation Coverage**: Checklist of validations
5. **Issues Found**: List of failed tests with log references
6. **Recommendations**: Next steps (if failures)

**Example Summary** (all tests passed):
```markdown
# E2E File Server Test Suite - Summary Report

**Test Run:** 20251117_160000
**Generated:** 2025-11-17 16:35:00

## Overall Results

| Metric | Count |
|--------|-------|
| **Total Tests** | 23 |
| **✅ Passed** | 23 |
| **❌ Failed** | 0 |
| **⚠️ Skipped** | 0 |
| **Success Rate** | 100.0% |

## Test Results by Category

### Upload & Processing Tests (01-15)

| # | Test Name | Status |
|---|-----------|--------|
| 01 | 01_basic_upload | ✅ PASSED |
| 02 | 02_no_images | ✅ PASSED |
| 03 | 03_with_images | ✅ PASSED |
... (15 total)

### File Operation Tests (16-22)

| # | Test Name | Status |
|---|-----------|--------|
| 16 | 16_get_file_url | ✅ PASSED |
... (7 total)

## Parameter Coverage

| Parameter | Values Tested | Tests |
|-----------|---------------|-------|
| `includes_images` | 0, 1 | 02, 03 |
| `table_mode` | disabled, fast, accurate, auto | 04, 05, 06, 07 |
| `ocr_mode` | disabled, enabled, auto | 08, 09, 10 |
| File Types | TXT, PDF, XLSX, CSV, JPEG | All |

## Issues Found

✅ **No issues found** - All tests passed!

## Recommendations

✅ All validation checks passed. File server and document processing pipeline functioning correctly.

System validated:
- Multi-tenant architecture (V2.1)
- Organization-centric MinIO buckets
- Connection tenant isolation
- Parameter variations (includes_images, table_mode, ocr_mode)
- Special file type handling (Excel, CSV, images)
```

---

## Next Steps After Test

1. **Review failed tests** (if any): Check logs in `reports/{timestamp}/test_NN.log`
2. **Verify parameter handling**: Ensure all modes (disabled, fast, accurate, auto) work
3. **Test edge cases**: Large files (>100 MB), corrupted files, unsupported formats
4. **Performance testing**: Measure processing time for each file type
5. **Cleanup**: Delete test organization and files

---

# Appendix: Common Patterns

## Pattern 1: Multi-Tenant Weaviate Queries

**Always use `limit` parameter**:
```bash
# ❌ WRONG - Returns 0 even if data exists
curl "http://localhost:8080/v1/objects?class=Collection&tenant=conn_xxx"

# ✅ CORRECT - Returns actual count
curl "http://localhost:8080/v1/objects?class=Collection&tenant=conn_xxx&limit=100"
```

---

## Pattern 2: Tenant ID Sanitization

**Connection IDs have hyphens removed**:
```bash
# Connection ID
CONN_ID="conn-upload-20251117140000"

# Tenant ID (sanitized)
TENANT_ID="conn_connupload20251117140000"
         # ^^^^^^ hyphens removed
```

---

## Pattern 3: MinIO Bucket Structure

**Organization-centric buckets**:
```
org-{organization_uuid}/
├── uploads/                    # User-uploaded files
│   └── conn-{connection_uuid}/
│       ├── documents/
│       ├── images/
│       └── tables/
│
└── airbyte/                    # Airbyte CDC files
    └── conn-{airbyte_connection_id}/
        └── {source_type}/      # e.g., google-drive, confluence
            ├── items/
            ├── images/
            └── tables/
```

---

## Pattern 4: RabbitMQ Message Format

**Queue**: `file_notifications`

**Message**:
```json
{
  "file_hash": "abc123...",
  "organization_id": "org-uuid",
  "connection_id": "conn-uuid",
  "namespace": "workspace",
  "collection_name": "Collection",
  "encrypted_url": "gAAAAAB...",
  "custom_parameters": {
    "table_mode": "auto",
    "ocr_mode": "auto",
    "includes_images": true
  },
  "llm_models": [...],
  "embedding_model": "embedanything-minilm"
}
```

---

## Pattern 5: Service Health Checks

**Check all services before running tests**:
```bash
# LiteLLM Wrapper
curl http://localhost:8100/health

# File Server
curl http://localhost:8020/file/

# Orchestrator
curl http://localhost:4092/orchestrator/

# Weaviate
curl http://localhost:8080/v1/.well-known/ready

# Airbyte Wrapper
curl http://localhost:4040/health

# MinIO
curl http://localhost:9000/minio/health/live

# RabbitMQ
docker exec rabbitmq-broker rabbitmqctl status
```

---

## Pattern 6: Processing Time Expectations

**By file type**:
- TXT/MD: 10-30 seconds
- CSV: 20-40 seconds
- HTML: 30-60 seconds
- PDF (simple): 1-2 minutes
- PDF (complex): 2-3 minutes
- JPEG (standalone): 30-60 seconds
- PPTX: 3-5 minutes
- XLSX (multi-sheet): 1-2 minutes

**Add buffer**: +30-60 seconds for Weaviate ingestion lag

---

## Pattern 7: Error Debugging Commands

**Data Manager Logs**:
```bash
# Check for errors
docker logs horus_data-manager_1 --tail 100 | grep -i error

# Check processing status
docker logs horus_data-manager_1 --tail 50 | grep "Processing file"
```

**RabbitMQ Queue Status**:
```bash
# Check queue depth
docker exec rabbitmq-broker rabbitmqctl list_queues name messages consumers

# Check for DLQ messages
docker exec rabbitmq-broker rabbitmqctl list_queues | grep dlq
```

**MinIO File Check**:
```bash
# List files in organization bucket
ls -R minio_mount/org-{uuid}/

# Count files in connection path
find minio_mount/org-{uuid}/uploads/conn-{uuid} -type f | wc -l
```

**Weaviate Schema Check**:
```bash
# List all collections
curl http://localhost:8080/v1/schema | jq -r '.classes[].class'

# List tenants for a collection
curl http://localhost:8080/v1/schema/{CollectionName}/tenants | jq -r '.[].name'
```

---

**End of E2E Tests Detailed Guide**

For quick start instructions, see [QUICKSTART.md](QUICKSTART.md)
For complete platform documentation, see [../CLAUDE.md](../CLAUDE.md)
