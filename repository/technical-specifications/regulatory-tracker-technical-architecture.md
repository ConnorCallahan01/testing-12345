# Regulatory Tracker - Technical Architecture Documentation

## Architecture Overview

The Regulatory Tracker is built as a modern, cloud-ready application using a three-tier architecture pattern with asynchronous background processing for AI-powered regulatory analysis.

---

## System Architecture Layers

### 1. Client Layer

**Component:** Web Browser  
**Supported Browsers:** Chrome, Firefox, Safari, Edge  
**Protocol:** HTTP/HTTPS  
**Port:** 3000 (development), 80/443 (production)

The client layer is a standard web browser that renders the React-based frontend and communicates with the backend via RESTful APIs.

---

### 2. Frontend Layer - Next.js 14

#### Core Framework
- **Next.js 14**: React framework with App Router
- **React 19**: UI component library
- **TypeScript**: Type-safe JavaScript
- **Tailwind CSS**: Utility-first CSS framework

#### Architecture Components

**App Router (File-based Routing)**
```
src/app/
├── page.tsx                    # Dashboard (/)
├── scans/
│   ├── new/page.tsx           # New Scan Form (/scans/new)
│   └── [id]/page.tsx          # Scan Status (/scans/:id)
├── results/
│   ├── page.tsx               # Results List (/results)
│   └── [id]/page.tsx          # Result Detail (/results/:id)
├── workflow/
│   └── page.tsx               # Workflow Queue (/workflow)
└── knowledge/
    └── page.tsx               # Knowledge Store (/knowledge)
```

**React Components**
- **Dashboard Components**: StatsCards, RecentScans
- **Results Components**: ResultsList, ResultDetail, ImpactBadge
- **Workflow Components**: WorkflowQueue, StatusTransition
- **Knowledge Components**: KnowledgeList, SearchBar, ExportButton
- **Shared Components**: Navigation, Loading, ErrorBoundary

**API Client**
- Centralized API communication layer
- Axios/Fetch for HTTP requests
- Error handling and retry logic
- Request/response interceptors
- Type-safe API interfaces

**State Management**
- React Context for global state
- Server Components for data fetching
- Client Components for interactivity

---

### 3. Backend Layer - FastAPI

#### Core Framework
- **FastAPI**: Modern Python web framework
- **Python 3.11+**: Programming language
- **Pydantic**: Data validation and serialization
- **Uvicorn**: ASGI server

#### API Endpoints

**Health Check**
```
GET /api/health
Response: { "status": "healthy", "timestamp": "..." }
```

**Scans Management**
```
POST /api/scans
Body: { "domain": "AML", "jurisdiction": "US", "sub_domains": "..." }
Response: { "id": "uuid", "status": "pending" }

GET /api/scans
Response: [{ "id": "...", "status": "...", "created_at": "..." }]

GET /api/scans/{id}
Response: { "id": "...", "status": "running:analyzing", "progress": "..." }
```

**Results Management**
```
GET /api/results
Query: ?impact_level=high&workflow_status=draft
Response: [{ "id": "...", "title": "...", "impact_level": "..." }]

GET /api/results/{id}
Response: { "id": "...", "executive_summary": "...", "analysis": {...} }

PATCH /api/results/{id}
Body: { "workflow_status": "in_review" }
Response: { "id": "...", "workflow_status": "in_review" }
```

**Workflow Management**
```
GET /api/workflow/queue
Response: {
  "draft": [...],
  "in_review": [...],
  "verified": [...]
}

POST /api/workflow/{id}/transition
Body: { "to_status": "verified", "comment": "..." }
Response: { "id": "...", "workflow_status": "verified" }
```

**Knowledge Store**
```
GET /api/knowledge
Query: ?q=beneficial+ownership&impact_level=high
Response: [{ "id": "...", "title": "...", "verified_at": "..." }]

GET /api/knowledge/{id}/export
Response: Binary .docx file download
```

**Sources Registry**
```
GET /api/sources
Response: [{ "source_id": "...", "source_name": "FinCEN", "url": "..." }]
```

---

### 4. Core Services (Backend)

#### ScanOrchestrator

**Purpose:** Coordinates the multi-stage scan pipeline

**Responsibilities:**
- Manages state transitions across 7 stages
- Handles errors and retries
- Updates scan status in real-time
- Coordinates between services
- Logs all activities

**7-Stage Pipeline:**

1. **Initializing**
   - Verify scan record exists
   - Set up execution context
   - Update status: `running:initializing`

2. **Retrieving Sources**
   - Load regulatory source files
   - Parse JSON documents
   - Update status: `running:retrieving_sources`

3. **Detecting Changes**
   - Compute content hashes
   - Detect regulatory keywords
   - Filter changed sources
   - Update status: `running:detecting_changes`

4. **Analyzing**
   - For each changed source:
     - Build analysis prompt
     - Call Claude API
     - Parse JSON response
     - Create result record
   - Update status: `running:analyzing`

5. **Scoring**
   - Normalize confidence scores
   - Flag items for human review
   - Update result records
   - Update status: `running:scoring`

6. **Generating Artifacts**
   - Generate markdown artifacts
   - Create Word documents
   - Save to artifacts directory
   - Update status: `running:generating_artifacts`

7. **Finalizing**
   - Complete background tasks
   - Update status: `completed`
   - Make results available

**Error Handling:**
- Try-catch blocks at each stage
- Graceful degradation
- Status updates on failure: `failed`
- Error messages logged and stored

---

#### LLMService

**Purpose:** Integrates with Claude API for AI analysis

**Configuration:**
- API Key: `ANTHROPIC_API_KEY` environment variable
- Model: `claude-sonnet-4-20250514`
- Max Tokens: 4096
- Temperature: Default (1.0)

**Prompt Engineering:**

**System Prompt:**
```
You are an expert regulatory compliance analyst specializing in 
financial services regulations. Your role is to analyze regulatory 
documents and assess their impact on compliance programs.
```

**Task Prompt Template:**
```
Analyze the following regulatory document:

Domain: {domain}
Sub-domains: {sub_domains}
Jurisdiction: {jurisdiction}
Regulated Activities: {regulated_activities}

Source: {source_name}
URL: {url}
Publication Date: {publication_date}

Content:
{content}

Provide a structured analysis in JSON format:
{output_schema}
```

**Output Schema:**
```json
{
  "title": "string",
  "is_regulatory_change": "boolean",
  "is_relevant": "boolean",
  "relevance_rationale": "string",
  "executive_summary": "string",
  "change_description": "string",
  "impact_level": "low|medium|high|critical",
  "affected_areas": ["string"],
  "recommended_actions": ["string"],
  "effective_date": "YYYY-MM-DD",
  "confidence_score": "float (0.0-1.0)",
  "requires_human_review": "boolean",
  "citations": ["string"],
  "notes": "string"
}
```

**API Call Flow:**
1. Build prompt with source data and scan parameters
2. Make synchronous API call to Anthropic
3. Receive response with analysis
4. Parse JSON from response text
5. Validate against schema
6. Return structured analysis

**Error Handling:**
- Retry once on API errors
- Log all errors with context
- Return minimal result on failure
- Flag for human review on errors

---

#### ChangeDetectionService

**Purpose:** Identifies material changes in regulatory documents

**Detection Methods:**

**1. Content Hashing**
- Algorithm: SHA-256
- Normalization: Lowercase, collapse whitespace
- Comparison: Current hash vs. previous hash
- Result: Changed if hashes differ

**2. Keyword Detection**
- Pattern: Regex-based case-insensitive matching
- Keywords:
  - "amended", "amendment", "effective", "effective date"
  - "new rule", "final rule", "repealed", "revoked"
  - "supersedes", "revised", "revision", "modification"
  - "compliance deadline", "enforcement", "mandatory"
  - "requirement", "shall", "must comply", "takes effect"
- Result: Changed if keywords found

**3. Demo Mode**
- Always returns `True` for material changes
- Ensures all mock data is processed
- Useful for demonstrations and testing

**Configuration:**
```python
ChangeDetectionService(demo_mode=True)
```

**API:**
```python
async def has_material_change(
    source_data: Dict[str, Any],
    previous_hash: Optional[str] = None
) -> bool
```

---

#### ArtifactGenerator

**Purpose:** Creates professional documentation artifacts

**Output Formats:**

**1. Markdown Artifact**
- Stored in database
- Quick display in UI
- Structured format:
  ```markdown
  # {title}
  
  **Source:** {source_name}
  **Published:** {publication_date}
  **Effective:** {effective_date}
  
  ## Executive Summary
  {executive_summary}
  
  ## Change Description
  {change_description}
  
  ## Impact Assessment
  **Level:** {impact_level}
  **Affected Areas:** {affected_areas}
  
  ## Recommended Actions
  {recommended_actions}
  
  ## Confidence Score
  {confidence_score}
  ```

**2. Word Document (.docx)**
- Saved to artifacts directory
- Professional formatting
- Includes:
  - Cover page with metadata
  - Table of contents
  - Executive summary section
  - Detailed analysis sections
  - Headers and footers
  - Page numbers
- Filename: `{result_id}_{timestamp}.docx`

**Document Generation:**
```python
def generate_word_document(artifact_data: Dict[str, Any]) -> str:
    # Create Document object
    # Add cover page
    # Add table of contents
    # Add content sections
    # Apply styling
    # Save to artifacts directory
    # Return file path
```

---

### 5. Database Layer

#### SQLite Database

**File:** `data/regulatory_tracker.db`  
**Type:** File-based relational database  
**ORM:** None (direct SQL queries)

**Schema:**

**scans table**
```sql
CREATE TABLE scans (
    id TEXT PRIMARY KEY,
    domain TEXT NOT NULL,
    jurisdiction TEXT NOT NULL,
    sub_domains TEXT,
    status TEXT NOT NULL,
    error_message TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**results table**
```sql
CREATE TABLE results (
    id TEXT PRIMARY KEY,
    scan_id TEXT NOT NULL,
    source_id TEXT NOT NULL,
    source_name TEXT NOT NULL,
    title TEXT NOT NULL,
    executive_summary TEXT,
    change_description TEXT,
    is_relevant BOOLEAN DEFAULT TRUE,
    relevance_rationale TEXT,
    impact_level TEXT,
    affected_areas TEXT,
    recommended_actions TEXT,
    published_date TEXT,
    effective_date TEXT,
    confidence_score REAL,
    requires_human_review BOOLEAN DEFAULT FALSE,
    workflow_status TEXT DEFAULT 'draft',
    markdown_artifact TEXT,
    docx_path TEXT,
    llm_response TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (scan_id) REFERENCES scans(id)
);
```

**sources table**
```sql
CREATE TABLE sources (
    source_id TEXT PRIMARY KEY,
    source_name TEXT NOT NULL,
    document_type TEXT,
    url TEXT,
    jurisdiction TEXT,
    domain TEXT,
    last_checked TIMESTAMP,
    content_hash TEXT
);
```

**Database Operations:**
- `create_scan(scan_data)` - Insert new scan
- `get_scan(scan_id)` - Retrieve scan by ID
- `update_scan_status(scan_id, status, error)` - Update scan status
- `create_result(result_data)` - Insert new result
- `get_result(result_id)` - Retrieve result by ID
- `list_results(filters)` - Query results with filters
- `update_result(result_id, updates)` - Update result fields
- `list_verified_results()` - Query knowledge store

---

### 6. External Services

#### Anthropic Claude API

**Service:** Claude AI by Anthropic  
**Endpoint:** `https://api.anthropic.com/v1/messages`  
**Model:** `claude-sonnet-4-20250514`  
**Authentication:** API Key (Bearer token)

**Request Format:**
```json
{
  "model": "claude-sonnet-4-20250514",
  "max_tokens": 4096,
  "system": "You are an expert regulatory compliance analyst...",
  "messages": [
    {
      "role": "user",
      "content": "Analyze the following regulatory document..."
    }
  ]
}
```

**Response Format:**
```json
{
  "id": "msg_...",
  "type": "message",
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "{\"title\": \"...\", \"is_regulatory_change\": true, ...}"
    }
  ],
  "model": "claude-sonnet-4-20250514",
  "usage": {
    "input_tokens": 1234,
    "output_tokens": 567
  }
}
```

**Rate Limits:**
- Depends on API tier
- Production: Implement rate limiting and queuing
- Error handling: Exponential backoff on 429 errors

**Cost Considerations:**
- Input tokens: ~$3 per million tokens
- Output tokens: ~$15 per million tokens
- Average cost per analysis: $0.10 - $0.50
- Batch processing recommended for cost efficiency

---

### 7. Data Storage

#### Mock Sources Directory

**Location:** `data/mock-sources/`  
**Format:** JSON files  
**Purpose:** Simulates regulatory agency data sources

**Files:**
- `fincen-cdd-amendment.json` - FinCEN CDD Rule Amendment
- `fincen-sar-advisory.json` - FinCEN SAR Advisory
- `occ-tm-guidance.json` - OCC Transaction Monitoring Guidance
- `fed-aml-procedures.json` - Federal Reserve AML Procedures

**JSON Structure:**
```json
{
  "source_id": "fincen-cdd-2024-001",
  "source_name": "Financial Crimes Enforcement Network (FinCEN)",
  "document_type": "Final Rule Amendment",
  "title": "Amendment to Customer Due Diligence Requirements...",
  "url": "https://www.fincen.gov/...",
  "publication_date": "2024-11-15",
  "jurisdiction": "United States",
  "domain": "AML",
  "sub_domains": ["Client Onboarding", "KYC", "Beneficial Ownership"],
  "content": "DEPARTMENT OF THE TREASURY\n..."
}
```

**Production Replacement:**
- Replace with API integrations to regulatory agencies
- Implement web scraping for agencies without APIs
- Monitor RSS feeds and email alerts
- Store in database with version history

---

#### Artifacts Directory

**Location:** `data/artifacts/`  
**Format:** Microsoft Word (.docx) files  
**Purpose:** Stores generated compliance documents

**Naming Convention:**
```
{result_id}_{timestamp}.docx
Example: 550e8400-e29b-41d4-a716-446655440000_20241115_143022.docx
```

**File Management:**
- Created during Stage 6 (Generating Artifacts)
- Persisted for download via Knowledge Store
- Cleanup policy: Retain for 90 days (configurable)
- Backup: Sync to cloud storage (S3, Azure Blob, etc.)

---

### 8. Background Processing

#### Asyncio Task Queue

**Framework:** Python asyncio  
**Concurrency:** Asynchronous I/O  
**Task Management:** `asyncio.create_task()`

**Task Creation:**
```python
def start_scan_background(scan_id: str, scan_params: Dict) -> asyncio.Task:
    orchestrator = ScanOrchestrator()
    task = asyncio.create_task(
        orchestrator.run_scan(scan_id, scan_params),
        name=f"scan-{scan_id}"
    )
    task.add_done_callback(handle_task_exception)
    return task
```

**Task Lifecycle:**
1. API endpoint receives scan request
2. Creates scan record in database
3. Launches background task
4. Returns scan ID immediately
5. Task runs asynchronously
6. Updates status throughout execution
7. Completes or fails with error

**Error Handling:**
- Try-catch at each stage
- Callback for task exceptions
- Status updated to `failed` on error
- Error message stored in database

**Monitoring:**
- Task name includes scan ID
- Logs all stage transitions
- Tracks execution time
- Reports errors with stack traces

---

## Data Flow

### Scan Execution Flow

```
1. User submits scan configuration
   ↓
2. Frontend sends POST /api/scans
   ↓
3. Backend creates scan record (status: pending)
   ↓
4. Background task launched
   ↓
5. Orchestrator Stage 1: Initializing
   ↓
6. Orchestrator Stage 2: Retrieving Sources
   ↓ (loads mock JSON files)
7. Orchestrator Stage 3: Detecting Changes
   ↓ (hash + keyword detection)
8. Orchestrator Stage 4: Analyzing
   ↓ (calls Claude API for each source)
9. Claude analyzes content and returns JSON
   ↓
10. Create result records in database
    ↓
11. Orchestrator Stage 5: Scoring
    ↓ (normalize scores, flag reviews)
12. Orchestrator Stage 6: Generating Artifacts
    ↓ (create markdown + .docx files)
13. Orchestrator Stage 7: Finalizing
    ↓
14. Update scan status to completed
    ↓
15. Frontend polls /api/scans/{id} and displays results
```

### Result Review Flow

```
1. User navigates to /results
   ↓
2. Frontend sends GET /api/results
   ↓
3. Backend queries database
   ↓
4. Returns list of results
   ↓
5. User clicks on a result
   ↓
6. Frontend sends GET /api/results/{id}
   ↓
7. Backend retrieves result with full details
   ↓
8. Frontend displays executive summary, analysis, etc.
```

### Workflow Transition Flow

```
1. User clicks "Move to Review" button
   ↓
2. Frontend sends POST /api/workflow/{id}/transition
   ↓
3. Backend validates transition (draft → in_review)
   ↓
4. Updates workflow_status in database
   ↓
5. Returns updated result
   ↓
6. Frontend refreshes workflow queue
```

### Export Flow

```
1. User clicks "Export" button
   ↓
2. Frontend sends GET /api/knowledge/{id}/export
   ↓
3. Backend retrieves docx_path from database
   ↓
4. Reads .docx file from artifacts directory
   ↓
5. Streams file to browser with Content-Disposition header
   ↓
6. Browser downloads file
```

---

## Technology Stack Summary

| Layer | Technology | Version | Purpose |
|-------|-----------|---------|---------|
| **Frontend** | Next.js | 14 | React framework with App Router |
| | React | 19 | UI component library |
| | TypeScript | 5+ | Type-safe JavaScript |
| | Tailwind CSS | 4+ | Utility-first CSS |
| **Backend** | FastAPI | Latest | Python web framework |
| | Python | 3.11+ | Programming language |
| | Pydantic | Latest | Data validation |
| | Uvicorn | Latest | ASGI server |
| **Database** | SQLite | 3+ | File-based relational DB |
| **AI** | Claude API | Sonnet 4 | Regulatory analysis |
| **Containerization** | Docker | Latest | Application containers |
| | Docker Compose | v2+ | Multi-container orchestration |
| **Document Generation** | python-docx | Latest | Word document creation |
| **Async Processing** | asyncio | Built-in | Background task execution |

---

## Deployment Architecture

### Docker Compose Setup

**Services:**

1. **Frontend Container**
   - Image: Node.js 20 Alpine
   - Port: 3000
   - Environment: `NEXT_PUBLIC_API_URL=http://backend:8000`
   - Volume: `./frontend:/app`
   - Command: `npm run dev`

2. **Backend Container**
   - Image: Python 3.11 Slim
   - Port: 8000
   - Environment: `ANTHROPIC_API_KEY`, `DATABASE_URL`
   - Volumes:
     - `./backend:/app`
     - `./data:/app/data`
   - Command: `uvicorn app.main:app --host 0.0.0.0 --port 8000`

**Networking:**
- Bridge network for inter-container communication
- Frontend → Backend: `http://backend:8000`
- Host → Frontend: `http://localhost:3000`
- Host → Backend: `http://localhost:8000`

**Volumes:**
- `./data/mock-sources` → `/app/data/mock-sources` (read-only)
- `./data/artifacts` → `/app/data/artifacts` (read-write)
- `./data/regulatory_tracker.db` → `/app/data/regulatory_tracker.db` (read-write)

---

## Security Considerations

### API Security
- **CORS**: Configured for frontend origin
- **Rate Limiting**: Implement for production
- **Input Validation**: Pydantic schemas enforce type safety
- **SQL Injection**: Parameterized queries prevent injection

### Authentication (Production)
- JWT tokens for user authentication
- Role-based access control (RBAC)
- API key management for Claude API
- Secure credential storage (environment variables, secrets manager)

### Data Security
- **Encryption at Rest**: Encrypt SQLite database
- **Encryption in Transit**: HTTPS/TLS for all connections
- **API Key Protection**: Never expose in client-side code
- **Audit Logging**: Track all data access and modifications

---

## Scalability Considerations

### Horizontal Scaling
- **Frontend**: Deploy multiple Next.js instances behind load balancer
- **Backend**: Deploy multiple FastAPI instances with shared database
- **Database**: Migrate from SQLite to PostgreSQL for concurrent writes
- **Background Tasks**: Use Celery with Redis/RabbitMQ for distributed task queue

### Vertical Scaling
- **Increase CPU**: More cores for parallel processing
- **Increase Memory**: Larger context windows for AI analysis
- **Increase Storage**: More space for artifacts and database

### Caching
- **Redis**: Cache API responses, scan results, knowledge store queries
- **CDN**: Cache static assets (Next.js build output)
- **Database Query Cache**: Cache frequent queries

### Performance Optimization
- **Database Indexing**: Index on scan_id, workflow_status, impact_level
- **Connection Pooling**: Reuse database connections
- **Batch Processing**: Group multiple sources in single Claude API call
- **Lazy Loading**: Load results on-demand in UI

---

## Monitoring and Observability

### Logging
- **Application Logs**: Python logging module with structured JSON
- **Access Logs**: Uvicorn access logs
- **Error Logs**: Stack traces with context
- **Audit Logs**: Track all data modifications

### Metrics
- **API Metrics**: Request count, latency, error rate
- **Scan Metrics**: Scans per hour, average duration, success rate
- **AI Metrics**: Claude API calls, token usage, cost
- **Database Metrics**: Query performance, connection pool usage

### Alerting
- **Error Rate**: Alert on >5% error rate
- **API Latency**: Alert on p95 > 2 seconds
- **Scan Failures**: Alert on failed scans
- **Claude API Errors**: Alert on API errors or rate limits

### Tools
- **Prometheus**: Metrics collection
- **Grafana**: Metrics visualization
- **Sentry**: Error tracking
- **ELK Stack**: Log aggregation and search

---

## Conclusion

The Regulatory Tracker architecture demonstrates a modern, scalable approach to building AI-powered compliance applications. Key architectural decisions include:

1. **Separation of Concerns**: Clear boundaries between frontend, backend, and services
2. **Asynchronous Processing**: Background tasks for long-running AI analysis
3. **Modular Services**: Independent, testable components
4. **API-First Design**: RESTful APIs enable future integrations
5. **Containerization**: Docker ensures consistent deployment
6. **Scalability**: Architecture supports horizontal and vertical scaling
7. **Observability**: Comprehensive logging and monitoring

This architecture provides a solid foundation for production deployment and future enhancements.
