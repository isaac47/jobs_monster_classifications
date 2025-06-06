# Complete KPI Extraction Workflow Documentation

## Overview
This document describes the complete workflow for the Benchmark KPI Extraction API, which provides automated extraction of Key Performance Indicators (KPIs) from financial documents. The system processes quarterly reports, press releases, and result slides from banking and financial institutions.

## Architecture Components
- **Frontend**: Client application for user interaction
- **API Gateway**: REST API endpoints with authentication
- **Lambda Functions**: Serverless processing components
- **S3**: Document and result storage
- **DynamoDB**: Status and results database
- **Bedrock LLM**: AI-powered text analysis
- **Step Functions**: Workflow orchestration
- **SQS**: Asynchronous message queuing
- **CloudWatch/X-Ray**: Monitoring and observability

---

## Complete Workflow Table

### Phase 1: User Interface & API Layer

| Flow | Description | Services | Arrow Event | Service Name | Details | Comments |
|------|-------------|----------|-------------|--------------|---------|----------|
| 1 | User submits KPI extraction request | Client App | HTTP Request | client-app | User provides: analyze_id (UUID), bank_id, bank_is_french flag, kpi_list, taxonomy, kpi_detail_levels, and file metadata | Frontend generates analyze_id for tracking |
| 2 | API Gateway receives presigned URL request | API Gateway | HTTP POST | api-gateway | Endpoint: POST /upload/presigned-url with JWT token in Authorization header | Entry point for file upload process |
| 2.1 | JWT token validation | Lambda Authorizer | Token Validation | lambda-authorizer | Verifies token scope, expiration, and signature against auth provider | Security checkpoint |
| 2.1.1 | Token refresh handling | Lambda Authorizer | Token Refresh | lambda-authorizer | Provides token refresh guidance if token is near expiration (within 1 hour) | Proactive token management |
| 2.2 | Request parameter validation | API Gateway | Parameter Check | api-gateway | Validates required fields: analyze_id, bank_is_french, kpi_list, taxonomy, kpi_detail_levels, bank_id, files | Input sanitization |
| 2.3 | Generate presigned URLs | Lambda | Lambda Invoke | presigned-url-lambda | Creates time-limited (30 min) presigned URLs for secure direct S3 upload | One URL per file, maximum 25MB |
| 2.4 | Request logging | CloudWatch | Log Event | cloudwatch-logs | All requests logged with userID, timestamp, analyze_id, bank_id, IP address | Audit trail creation |
| 3 | Presigned URLs returned to client | API Gateway | HTTP Response | api-gateway | Returns analyze_id, upload_timestamp, and array of file objects with presigned URLs and file_ids | Response includes upload instructions |
| 4 | User uploads documents via presigned URLs | S3 | S3 PUT | s3-bucket | Direct upload to S3 bucket with structure: /input/{analyze_id}/{file_id}/{file_name} | Bypasses API Gateway for large files |
| 4.1 | Rich metadata attachment | S3 | S3 Metadata | s3-bucket | JSON metadata attached: file_category, analyze_id, bank_id, kpi_extraction_params, upload_timestamp | Metadata for processing context |
| 4.2 | Upload progress tracking | S3 | S3 Event | s3-bucket | Multipart upload events tracked for large files with progress indicators | User experience optimization |

### Phase 2: Document Ingestion & Initialization

| Flow | Description | Services | Arrow Event | Service Name | Details | Comments |
|------|-------------|----------|-------------|--------------|---------|----------|
| 5 | S3 upload completion trigger | S3 | S3 Event | s3-bucket | ObjectCreated event triggers when all files for analyze_id are uploaded | Event-driven processing initiation |
| 5.1 | Upload aggregation check | Lambda | Lambda Invoke | upload-aggregator-lambda | Verifies all expected files are uploaded before starting processing | Ensures complete document set |
| 5.2 | Process initiation message | SQS | SQS Message | sqs-processing-queue | Message contains: analyze_id, file_paths[], bank_id, processing_params, timestamp | Decouples upload from processing |
| 6 | Initialize processing status | DynamoDB | DynamoDB Write | dynamodb-status-table | Creates status record with analyze_id, status: "processing", current_step: "initialization" | Status tracking initialization |
| 6.1 | Document processing orchestration | Step Functions | State Machine Start | step-functions-orchestrator | Initiates KPI extraction pipeline with error handling and retry logic | Central workflow management |
| 6.2 | Processing start notification | SQS | SQS Message | sqs-notification-queue | Optional notification to external systems about processing start | Integration hook |

### Phase 3: Document Validation & Preprocessing

| Flow | Description | Services | Arrow Event | Service Name | Details | Comments |
|------|-------------|----------|-------------|--------------|---------|----------|
| 7 | File validation step | Lambda | Lambda Invoke | file-validator-lambda | Validates: file format (PDF/XLSX), size (<25MB), file integrity, text layer presence | Quality gate before processing |
| 7.1 | Status update: file_validation | DynamoDB | DynamoDB Write | dynamodb-status-table | Updates current_step: "file_validation", progress metrics | Real-time status for frontend |
| 7.2 | File corruption detection | Lambda | File Check | file-validator-lambda | Attempts to open and read file headers, detects corruption or encryption | Prevents downstream failures |
| 7.3 | Document type classification | Lambda | Classification | document-classifier-lambda | Identifies document structure: tabular vs text, language detection | Processing strategy optimization |
| 8 | Virus and security scanning | Lambda | Security Scan | security-scanner-lambda | Scans uploaded files for malware, validates file signatures | Security compliance |
| 8.1 | Quarantine handling | S3 | S3 Move | s3-quarantine-bucket | Moves suspicious files to quarantine bucket with detailed logging | Security incident management |

### Phase 4: Text Extraction & Document Processing

| Flow | Description | Services | Arrow Event | Service Name | Details | Comments |
|------|-------------|----------|-------------|--------------|---------|----------|
| 9 | Text extraction from documents | Lambda | Lambda Invoke | text-extractor-lambda | PDF: OCR + text layer extraction; XLSX: cell content + formatting | Multi-format document processing |
| 9.1 | Status update: text_extraction | DynamoDB | DynamoDB Write | dynamodb-status-table | Updates current_step: "text_extraction", extracted_pages_count | Progress transparency |
| 9.2 | OCR processing for scanned PDFs | Textract | OCR API Call | textract-service | Advanced OCR for image-based PDFs with table detection | Handles legacy documents |
| 9.3 | Excel sheet processing | Lambda | Sheet Processing | excel-processor-lambda | Extracts data from multiple sheets, preserves cell relationships | Complex spreadsheet handling |
| 10 | Language detection and preprocessing | Lambda | Language Analysis | language-detector-lambda | Detects document language, applies bank_is_french context for processing | Language-aware extraction |
| 10.1 | Text normalization | Lambda | Text Processing | text-normalizer-lambda | Removes noise, standardizes formatting, handles special characters | Clean text for analysis |
| 11 | Document chunking for processing | Lambda | Lambda Invoke | chunking-lambda | Splits documents into semantic chunks while preserving financial context | Optimized for LLM token limits |
| 11.1 | Status update: chunking | DynamoDB | DynamoDB Write | dynamodb-status-table | Updates current_step: "chunking", total_chunks_created | Granular progress tracking |
| 11.2 | Context preservation | Lambda | Context Analysis | context-preserver-lambda | Maintains table relationships, page references, and financial statement structure | Accuracy optimization |
| 12 | Store processed chunks | S3 | S3 PUT | s3-processed-bucket | Saves chunked text to /processed/{analyze_id}/{file_id}/chunks/ with metadata | Intermediate result storage |

### Phase 5: KPI Identification & Value Extraction

| Flow | Description | Services | Arrow Event | Service Name | Details | Comments |
|------|-------------|----------|-------------|--------------|---------|----------|
| 13 | KPI identification in text chunks | Bedrock LLM | LLM API Call | bedrock-claude-model | Uses taxonomy and kpi_list to identify KPI mentions with confidence scores | AI-powered pattern recognition |
| 13.1 | Status update: kpi_identification | DynamoDB | DynamoDB Write | dynamodb-status-table | Updates current_step: "kpi_identification", kpis_identified_count | Progress monitoring |
| 13.2 | Taxonomy matching | Lambda | Pattern Matching | taxonomy-matcher-lambda | Matches identified text against provided KPI synonyms and variations | Synonym-aware identification |
| 13.3 | False positive filtering | Lambda | Validation | false-positive-filter-lambda | Removes non-financial KPI mentions using contextual analysis | Accuracy improvement |
| 14 | Value extraction from identified KPIs | Lambda | Lambda Invoke | value-extractor-lambda | Extracts numerical values, units (million/billion), currencies, percentages | Multi-format number parsing |
| 14.1 | Status update: value_extraction | DynamoDB | DynamoDB Write | dynamodb-status-table | Updates current_step: "value_extraction", values_extracted_count | Extraction progress |
| 14.2 | Currency and unit normalization | Lambda | Normalization | currency-normalizer-lambda | Standardizes currency formats, converts units to consistent base | Data standardization |
| 14.3 | Page reference tracking | Lambda | Reference Mapping | page-tracker-lambda | Maps extracted values to specific page numbers and document sections | Source traceability |
| 15 | Context analysis and validation | Bedrock LLM | LLM API Call | bedrock-claude-model | Analyzes KPI context using kpi_detail_levels for granular classification | Hierarchical KPI classification |
| 15.1 | Status update: context_analysis | DynamoDB | DynamoDB Write | dynamodb-status-table | Updates current_step: "context_analysis", contexts_analyzed_count | Analysis progress |
| 15.2 | Detail level classification | Lambda | Classification | detail-classifier-lambda | Assigns KPIs to specific detail levels (Group, Retail Banking, etc.) | Granular categorization |
| 15.3 | Confidence scoring | Lambda | Scoring | confidence-scorer-lambda | Calculates confidence scores for each extraction based on context quality | Quality assessment |

### Phase 6: Result Compilation & Quality Assurance

| Flow | Description | Services | Arrow Event | Service Name | Details | Comments |
|------|-------------|----------|-------------|--------------|---------|----------|
| 16 | Cross-validation of extracted values | Lambda | Lambda Invoke | cross-validator-lambda | Compares values across different document sections for consistency | Data integrity check |
| 16.1 | Duplicate detection and resolution | Lambda | Deduplication | duplicate-resolver-lambda | Identifies and resolves duplicate KPI extractions from multiple sources | Prevents data redundancy |
| 17 | Compile extraction results | Lambda | Lambda Invoke | result-compiler-lambda | Aggregates all KPI extractions by file_category with source page references | Final result assembly |
| 17.1 | Status update: result_compilation | DynamoDB | DynamoDB Write | dynamodb-status-table | Updates current_step: "result_compilation", results_compiled_count | Compilation progress |
| 17.2 | Result formatting | Lambda | Formatting | result-formatter-lambda | Formats results according to API schema with proper data types | Schema compliance |
| 17.3 | Quality metrics calculation | Lambda | Metrics | quality-calculator-lambda | Calculates overall extraction quality, coverage, and confidence metrics | Quality assessment |
| 18 | Store final results | DynamoDB | DynamoDB Write | dynamodb-results-table | Stores complete analysis_results with KPI values, currencies, units, source_pages | Persistent result storage |
| 18.1 | Mark processing complete | DynamoDB | DynamoDB Write | dynamodb-status-table | Updates status: "complete", processing_end_timestamp, total_processing_time | Final status update |
| 18.2 | Result backup | S3 | S3 PUT | s3-results-bucket | Backup copy of results in JSON format for archival and recovery | Data redundancy |
| 19 | Send completion notification | SQS | SQS Message | sqs-notification-queue | Notification with analyze_id, status, processing_time, result_summary | Integration notification |

### Phase 7: Status Monitoring & Results Retrieval

| Flow | Description | Services | Arrow Event | Service Name | Details | Comments |
|------|-------------|----------|-------------|--------------|---------|----------|
| 20 | Status check request | API Gateway | HTTP GET | api-gateway | Endpoint: GET /status/{analyze_id} with JWT authentication | Real-time status endpoint |
| 20.1 | Authenticate status request | Lambda Authorizer | Token Validation | lambda-authorizer | Verifies JWT token and user permissions for analyze_id | Access control |
| 20.2 | Retrieve current status | DynamoDB | DynamoDB Read | dynamodb-status-table | Returns status, current_step, error details, timestamps, progress metrics | Comprehensive status info |
| 20.3 | Status response formatting | Lambda | Response Format | status-formatter-lambda | Formats status response according to API schema | Consistent API response |
| 21 | Results retrieval request | API Gateway | HTTP GET | api-gateway | Endpoint: GET /answer/{analyze_id} with JWT authentication | Results endpoint |
| 21.1 | Authenticate results request | Lambda Authorizer | Token Validation | lambda-authorizer | Verifies JWT token and user permissions for analyze_id | Secure access control |
| 21.2 | Check processing completion | DynamoDB | DynamoDB Read | dynamodb-status-table | Verifies status is "complete" before returning results | Completion verification |
| 21.3 | Fetch complete results | DynamoDB | DynamoDB Read | dynamodb-results-table | Returns full analysis_results with all extracted KPIs organized by file | Complete dataset retrieval |
| 21.4 | Results response caching | ElastiCache | Cache Write | elasticache-cluster | Caches results for faster subsequent retrievals (24-hour TTL) | Performance optimization |

### Phase 8: Error Handling & Recovery

| Flow | Description | Services | Arrow Event | Service Name | Details | Comments |
|------|-------------|----------|-------------|--------------|---------|----------|
| 22 | Processing error detection | Lambda | Error Event | error-detector-lambda | Monitors all processing steps for failures and timeouts | Proactive error management |
| 22.1 | Error classification | Lambda | Classification | error-classifier-lambda | Categorizes errors: validation, processing, timeout, resource, external | Structured error handling |
| 22.2 | Retry logic execution | Step Functions | Retry State | step-functions-retry | Implements exponential backoff retry for transient failures | Resilience mechanism |
| 22.3 | Dead letter queue handling | SQS | DLQ Message | sqs-dead-letter-queue | Captures permanently failed messages for manual investigation | Failure analysis |
| 23 | Update error status | DynamoDB | DynamoDB Write | dynamodb-status-table | Updates status: "failed", error_code, error_message, error_details | Error state tracking |
| 23.1 | Error notification | SQS | SQS Message | sqs-error-notification | Sends error notifications to monitoring systems and stakeholders | Incident alerting |
| 23.2 | Partial result recovery | Lambda | Recovery Logic | partial-recovery-lambda | Attempts to recover and return partial results when possible | Data salvage attempt |
| 24 | Cleanup and resource management | Lambda | Cleanup | cleanup-lambda | Removes temporary files, releases resources after processing completion | Resource optimization |

### Phase 9: Observability & Security

| Flow | Description | Services | Arrow Event | Service Name | Details | Comments |
|------|-------------|----------|-------------|--------------|---------|----------|
| 25 | Comprehensive logging | CloudWatch | Log Stream | cloudwatch-logs | All processing steps logged with analyze_id correlation and performance metrics | Full audit trail |
| 25.1 | Structured log formatting | Lambda | Log Processing | log-formatter-lambda | Formats logs in JSON with standard fields for analysis | Log analytics optimization |
| 25.2 | Log retention management | CloudWatch | Retention Policy | cloudwatch-retention | Implements log retention policies based on compliance requirements | Storage cost management |
| 26 | Performance monitoring | X-Ray | Trace Event | x-ray-tracing | Distributed tracing across all Lambda functions, API calls, and database operations | Performance insights |
| 26.1 | Latency analysis | X-Ray | Performance Analysis | x-ray-analyzer | Identifies bottlenecks and performance optimization opportunities | Continuous improvement |
| 26.2 | Custom metrics collection | CloudWatch | Custom Metrics | cloudwatch-metrics | Tracks KPI extraction rates, accuracy metrics, processing times | Business metrics |
| 27 | Event monitoring and alerting | EventBridge | Event Rule | eventbridge-rules | Monitors processing events, triggers alerts for SLA violations | Operational monitoring |
| 27.1 | SLA monitoring | CloudWatch | Alarm | cloudwatch-alarms | Monitors processing time SLAs, error rates, and availability metrics | Service quality assurance |
| 27.2 | Automated scaling triggers | Auto Scaling | Scaling Event | auto-scaling-group | Adjusts Lambda concurrency and DynamoDB capacity based on load | Elastic resource management |
| 28 | Security scanning and compliance | KMS | Encryption | kms-customer-keys | All data encrypted at rest and in transit with customer-managed keys | Security compliance |
| 28.1 | Access pattern monitoring | CloudTrail | API Audit | cloudtrail-logs | Monitors all API access patterns for security anomalies | Security monitoring |
| 28.2 | Data residency compliance | S3 | Data Location | s3-region-restriction | Ensures data remains in specified regions for compliance requirements | Regulatory compliance |

### Phase 10: Data Lifecycle & Maintenance

| Flow | Description | Services | Arrow Event | Service Name | Details | Comments |
|------|-------------|----------|-------------|--------------|---------|----------|
| 29 | Data archival process | S3 | Lifecycle Policy | s3-lifecycle-management | Moves older documents and results to cheaper storage classes | Cost optimization |
| 29.1 | Automated data purging | Lambda | Scheduled Event | data-purger-lambda | Removes expired analyze_ids and associated data per retention policies | Compliance and storage management |
| 29.2 | Backup verification | Lambda | Backup Check | backup-verifier-lambda | Verifies integrity of backups and archive data | Data integrity assurance |
| 30 | System health monitoring | Lambda | Health Check | health-monitor-lambda | Performs end-to-end system health checks and dependency verification | System reliability |
| 30.1 | Capacity planning | CloudWatch | Metrics Analysis | capacity-planner-lambda | Analyzes usage patterns for capacity planning and cost optimization | Resource planning |
| 30.2 | Performance optimization | Lambda | Optimization | performance-optimizer-lambda | Identifies and implements performance improvements based on metrics | Continuous optimization |

---

## Processing States and Transitions

### Status Flow States:
1. **initialization** → File upload and basic validation
2. **file_validation** → File format and integrity checks
3. **text_extraction** → Document content extraction
4. **chunking** → Document segmentation for processing
5. **kpi_identification** → AI-powered KPI detection
6. **value_extraction** → Numerical value extraction
7. **context_analysis** → Contextual KPI classification
8. **result_compilation** → Final result assembly
9. **complete** → Processing successfully finished
10. **failed** → Processing encountered unrecoverable error

### Error Codes:
- **INVALID_REQUEST** - Malformed request parameters
- **AUTHENTICATION_FAILED** - JWT token issues
- **ACCESS_DENIED** - Insufficient permissions
- **FILE_PROCESSING_ERROR** - Document processing failures
- **KPI_EXTRACTION_ERROR** - AI model processing failures
- **RESOURCE_NOT_FOUND** - Missing analyze_id or results
- **TIMEOUT_ERROR** - Processing exceeded time limits
- **SYSTEM_ERROR** - Internal system failures

## Key Features:
- **Scalable Architecture**: Serverless components auto-scale based on demand
- **Real-time Status**: Comprehensive status tracking throughout processing
- **Error Resilience**: Multi-level error handling with retry mechanisms
- **Security First**: JWT authentication, encryption, and audit logging
- **Cost Optimized**: Pay-per-use model with automatic resource management
- **Multi-language Support**: French and English document processing
- **Quality Assurance**: Multiple validation steps and confidence scoring
- **Observability**: Complete monitoring, logging, and tracing capabilities

This workflow ensures secure, scalable, and reliable KPI extraction from financial documents with comprehensive error handling, real-time monitoring, and enterprise-grade security.
