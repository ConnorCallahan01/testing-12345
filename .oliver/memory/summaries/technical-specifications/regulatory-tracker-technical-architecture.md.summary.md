# Regulatory Tracker - Technical Architecture Documentation

**Source**: repository/technical-specifications/regulatory-tracker-technical-architecture.md
**Type**: Technical Specification
**Upload Date**: 2026-02-09
**Source SHA**: Not provided

## Key Points

- Three-tier architecture with Next.js 14 frontend, FastAPI backend, and SQLite database, utilizing asynchronous background processing for AI-powered regulatory analysis
- ScanOrchestrator manages a 7-stage pipeline (Initializing → Retrieving Sources → Detecting Changes → Analyzing → Scoring → Generating Artifacts → Finalizing) with error handling and real-time status updates
- LLMService integrates Claude Sonnet 4 API for structured regulatory document analysis with prompt engineering and JSON schema validation
- ChangeDetectionService identifies material regulatory changes through SHA-256 content hashing, regex-based keyword detection, and demo mode for testing
- ArtifactGenerator creates professional markdown summaries and Word documents (.docx) with metadata, sections, and styling for compliance documentation
- RESTful API endpoints manage scans, results, workflow transitions, and knowledge store exports with query filtering and status tracking
- Architecture supports horizontal scaling via load balancers, database migration to PostgreSQL, distributed task queues (Celery/Redis), and comprehensive monitoring with Prometheus, Grafana, and Sentry

## Entities and Topics

- **Next.js 14 & React 19**: Frontend framework with TypeScript and Tailwind CSS for UI component development using App Router pattern
- **FastAPI & Python 3.11+**: Backend framework with Pydantic validation and Uvicorn ASGI server for RESTful API endpoints
- **Claude Sonnet 4 API**: Anthropic's AI model for regulatory document analysis with configurable prompts and JSON output validation
- **SQLite Database**: File-based relational database storing scans, results, sources, and workflow status with schema supporting foreign key relationships
- **ScanOrchestrator**: Core service coordinating multi-stage pipeline with state transitions, error handling, and real-time status management
- **Docker Compose**: Multi-container orchestration with separate frontend (Node.js) and backend (Python) services on bridge network
- **Asynchronous Processing**: Python asyncio for background task execution with task lifecycle management and error callbacks
- **Mock Sources Directory**: JSON files simulating regulatory agency data (FinCEN, OCC, Federal Reserve) for development and demonstration
- **Artifacts Directory**: Storage location for generated Word documents with naming convention `{result_id}_{timestamp}.docx`
- **Security & Scalability**: CORS configuration, JWT authentication, rate limiting, database indexing, query caching, and performance optimization strategies

## Relevance to Project

This technical architecture document serves as the foundational specification for implementing the Regulatory Tracker system, providing detailed guidance on component interactions, data flows, API contracts, and deployment infrastructure. It enables developers to understand the complete system design, implement each layer according to specifications, and plan for scalability and production deployment.