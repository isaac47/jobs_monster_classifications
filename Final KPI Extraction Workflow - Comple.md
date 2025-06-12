# Final KPI Extraction Workflow - Complete Architecture

## System Overview
The KPI extraction system processes 1-3 financial documents per analyze_id through a sequential pipeline. The system uses simplified status management with only 3 analyze states: **processing**, **complete**, and **failed**.

## Complete Workflow

### Stage 1: File Upload & Initial Processing

#### 1.1 File Upload Trigger Lambda (`file-upload-handler`)
**Trigger**: S3 ObjectCreated event
**Purpose**: File validation and processing initiation

**Process Flow**:
1. Extract metadata from S3 event (file_id, analyze_id, file_name, file_category)
2. Initialize analyze status as "processing" if first file for this analyze_id
3. Validate file integrity by attempting to read the file
4. Create file status record with status="uploaded"
5. Check if all expected files for analyze_id are uploaded
6. If all files uploaded and analyze not failed → Call docling endpoint for each file

**Failure Scenarios**:
- **File corruption/unreadable** → Mark file as "failed" + Mark analyze as "failed"
- **Invalid file format** → Mark file as "failed" + Mark analyze as "failed"
- **S3 access errors** → Mark analyze as "failed"

**Status Updates**:
- File Status: Create with "uploaded"
- Analyze Status: Initialize as "processing"

**Error Handling Strategy**: Any file-level failure immediately fails the entire analysis

---

### Stage 2: Document Processing via Docling

#### 2.1 Docling Service (SageMaker Endpoint)
**Trigger**: Direct API call from file-upload-handler
**Purpose**: Extract text, structure, and generate chunks

**Process Flow**:
1. Receive file_id and analyze_id in request body
2. **Critical Check**: Query analyze_id status → If "failed", skip and return
3. Update file status to "docling_processing"
4. Read file from S3 using stored path
5. Process document through docling:
   - Extract text content and document structure
   - Identify tables, images, formatted sections
   - For images: Call Bedrock Vision API to generate descriptions
   - Generate semantic chunks with context preservation
   - Extract metadata (page numbers, section types, confidence scores)
6. Store all chunks in `kpi_document_chunks` DynamoDB table
7. Update file status to "docling_complete"

**Image Processing**:
- Detect embedded charts/graphs in financial documents
- Use Bedrock Claude 3 Vision to generate descriptive text
- Integrate descriptions into relevant text chunks
- Preserve visual context for KPI extraction

**Failure Scenarios**:
- **File read errors** → Mark file as "failed" + Mark analyze as "failed"
- **Docling processing errors** → Mark analyze as "failed"
- **Bedrock Vision API errors** → Continue without descriptions, log warning
- **DynamoDB storage errors** → Mark analyze as "failed"

**Status Updates**:
- File Status: "uploaded" → "docling_processing" → "docling_complete" or "failed"

---

#### 2.2 Status Handler Lambda (`status-handler`)
**Trigger**: DynamoDB Stream on file status changes to "docling_complete"
**Purpose**: Coordinate progression to embedding stage

**Process Flow**:
1. **Critical Check**: Query analyze_id status → If "failed", skip processing
2. For each file that reached "docling_complete" → Send message to embedding queue
3. Monitor overall progress for the analyze_id

---

### Stage 3: Embedding Generation

#### 3.1 Embedding Generator Lambda (`embedding-processor`)
**Trigger**: SQS message from status-handler
**Purpose**: Generate vector embeddings for semantic search

**Process Flow**:
1. Receive file_id and analyze_id from SQS message
2. **Critical Check**: Query analyze_id status → If "failed", skip processing
3. Update file status to "embedding_processing"
4. Query all chunks for file_id from `kpi_document_chunks`
5. Process chunks in batches through Bedrock Embedding API:
   - Use appropriate model based on bank_is_french flag
   - Process in batches of 25 texts maximum
   - Implement rate limiting and retry logic
6. Store embedding vectors back to chunk records
7. Update file status to "embedding_complete"
8. Send message to retrieval queue

**Embedding Strategy**:
- **French banks**: Use multilingual or French-optimized embedding model
- **Non-French banks**: Use English-optimized embedding model
- Batch processing for efficiency
- Store embeddings as dense vectors in DynamoDB

**Failure Scenarios**:
- **Bedrock API failures** → Retry with exponential backoff
- **Rate limiting** → Implement queue delays and retry
- **Persistent failures** → Mark file as "failed" + Mark analyze as "failed"

**Status Updates**:
- File Status: "docling_complete" → "embedding_processing" → "embedding_complete" or "failed"

---

### Stage 4: Retrieval Processing

#### 4.1 Hybrid Retriever Lambda (`retrieval-processor`)
**Trigger**: SQS message from embedding-processor
**Purpose**: Retrieve relevant chunks for KPI extraction

**Process Flow**:
1. Receive file_id and analyze_id from SQS message
2. **Critical Check**: Query analyze_id status → If "failed", skip processing
3. Update file status to "retrieval_processing"
4. Retrieve analyze parameters (kpi_list, taxonomy) from `kpi_analyze_status`
5. Construct retrieval queries:
   - Generate queries for each KPI in kpi_list
   - Include synonyms from taxonomy
   - Create detail-level specific queries
6. Execute hybrid retrieval strategy:
   - **Semantic search** (70% weight): Use embeddings for similarity
   - **Keyword search** (30% weight): Use BM25/lexical matching
   - Combine and rank results
7. Retrieve top-k chunks (10-15 per KPI)
8. Store results in `kpi_retrieval_context`
9. Update file status to "retrieval_complete"
10. Send message to KPI generation queue

**Retrieval Enhancement**:
- Query expansion using KPI taxonomy
- Financial context prioritization
- Duplicate chunk filtering
- Relevance score thresholding

**Failure Scenarios**:
- **Vector search errors** → Fall back to keyword-only search
- **No relevant chunks found** → Continue with empty context, log warning
- **DynamoDB errors** → Mark file as "failed" + Mark analyze as "failed"

**Status Updates**:
- File Status: "embedding_complete" → "retrieval_processing" → "retrieval_complete" or "failed"

---

### Stage 5: KPI Response Generation

#### 5.1 KPI Response Generator Lambda (`kpi-generator`)
**Trigger**: SQS message from retrieval-processor
**Purpose**: Extract structured KPI data using LLM

**Process Flow**:
1. Receive file_id and analyze_id from SQS message
2. **Critical Check**: Query analyze_id status → If "failed", skip processing and stop
3. Update file status to "kpi_processing"
4. Retrieve context chunks from `kpi_retrieval_context` table
5. Retrieve prompt template from DynamoDB prompt store
6. Construct comprehensive prompt:
   - Include file category context
   - Add KPI definitions and expected formats
   - Provide examples based on bank_is_french
   - Include detail_level hierarchies
   - Specify output JSON schema
7. Call Bedrock LLM API (Claude 3) with structured prompt
8. Parse and validate LLM response:
   - Verify JSON schema compliance
   - Validate extracted values and formats
   - Check KPI names against kpi_list
   - Ensure detail_levels match hierarchy
9. Store raw and parsed responses in `kpi_responses` table
10. Update file status to "kpi_complete"
    - If status update fails → Mark analyze as "failed"

**Failure Scenarios**:
- **LLM API timeouts** → Retry, then mark analyze as "failed"
- **Invalid JSON responses** → Retry, then mark analyze as "failed"
- **DynamoDB errors** → Mark analyze as "failed"
- **Any persistent failure** → Mark analyze as "failed"

**Status Updates**:
- File Status: "retrieval_complete" → "kpi_processing" → "kpi_complete" or "failed"

---

#### 5.2 Status Handler Lambda (`status-handler`) - KPI Completion Monitor
**Trigger**: DynamoDB Stream on file status change to "kpi_complete"
**Purpose**: Monitor KPI completion and trigger final output when all files are done

**Process Flow**:
1. Detect file status change to "kpi_complete"
2. **Critical Check**: Query analyze_id status → If "failed", skip processing and stop
3. Count how many KPI responses exist in `kpi_responses` table for this analyze_id
4. Get total expected document count for this analyze_id from `kpi_analyze_status`
5. **Completion Check**:
   - If count ≠ total expected documents → Stop (will be triggered again for next response)
   - If count = total expected documents → All files completed KPI extraction
6. When all complete → Send notification to output generation queue
7. Update processing status (mark analyze as "failed" if update fails)

**Logic**:
- This lambda triggers multiple times (once per file completing KPI extraction)
- Only proceeds to final output when ALL files have completed KPI extraction
- Implements a counting mechanism to ensure completeness

**Status Updates**:
- Monitors completion, triggers next stage when ready

---

### Stage 6: Final Output Generation

#### 6.1 Output Generator Lambda (`output-generator`)
**Trigger**: SQS message from status-handler when all KPI extractions complete
**Purpose**: Combine results and generate final output

**Process Flow**:
1. **Critical Check**: Query analyze_id status → If "failed", skip processing and stop
2. Retrieve all KPI responses for analyze_id from `kpi_responses` table
3. Retrieve original parameters and metadata from `kpi_analyze_status` table
4. Process and combine results:
   - Merge KPI extractions from multiple files
   - Resolve duplicate KPIs using confidence scores
   - Organize results by file_category as per API schema
   - Calculate processing statistics and metrics
5. Format final output according to API specification
6. Store complete JSON result in S3 document bucket at results path
7. Store output metadata in `kpi_final_output` table
8. **Mark analyze status as "complete"**

**Result Processing Logic**:
- **Duplicate Resolution**: Higher confidence scores take precedence
- **Cross-File Validation**: Consistency checks across documents
- **Quality Metrics**: Coverage, confidence, processing time calculations
- **Schema Compliance**: Final validation against API specification

**Failure Scenarios**:
- **S3 storage errors** → Mark analyze as "failed"
- **Data processing errors** → Mark analyze as "failed"
- **Output validation failures** → Mark analyze as "failed"

**Status Updates**:
- Analyze Status: "processing" → "complete" or "failed"

---
