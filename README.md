# JobPilot AI

**An AI-powered, self-hosted job search and application assistant.**

JobPilot AI helps job seekers organize their professional profile,
discover relevant opportunities, understand job requirements, prepare
tailored application documents, and track applications through their
lifecycle. Browser-assisted form preparation is planned, with **explicit
human approval required before any application is submitted**.




## Table of Contents

-   [Architecture overview](#architecture-overview)
-   [Request lifecycle](#request-lifecycle)
-   [Job discovery and filter
    pipeline](#job-discovery-and-filter-pipeline)
-   [Job matching and ranking](#job-matching-and-ranking)
-   [AI provider architecture](#ai-provider-architecture)
-   [Application and approval
    lifecycle](#application-and-approval-lifecycle)
-   [Entity relationship diagram](#entity-relationship-diagram)
-   [Project structure](#project-structure)
-   [API documentation](#api-documentation)
-   [Quickstart](#quickstart)
-   [Configuration reference](#configuration-reference)
-   [Development workflow](#development-workflow)
-   [License](#license)

------------------------------------------------------------------------

        


------------------------------------------------------------------------

## Architecture overview

JobPilot AI starts as a **modular monolith**. The frontend, API,
database, cache, and future worker are separate runtime components,
while backend business capabilities remain organized in one FastAPI
codebase.

##System architecture

flowchart TB
    U["User"] --> UI["Next.js Frontend<br/>TypeScript"]
    UI <-->|"REST API · JSON / HTTP"| API["FastAPI Backend"]

    subgraph BACKEND["Application Layer — Python"]
        API --> AUTH["Authentication & User Management"]
        API --> PROFILE["Profile & Resume Module"]
        API --> JOBS["Job Discovery & Management"]
        API --> MATCH["AI Matching Engine"]
        API --> APPS["Application Tracking"]
        API --> APPROVAL["Human Approval Service"]

        PROFILE --> PARSER["Resume Parser<br/>PDF / DOCX"]
        MATCH --> ROUTER["AI Provider Router"]
        ROUTER --> GEMINI["Gemini — Primary"]
        ROUTER --> OR["OpenRouter — Fallback"]

        JOBS --> SOURCES["Job Sources / Integrations"]
        APPROVAL --> BROWSER["Playwright Automation<br/>Planned"]
    end

    subgraph INFRA["Data & Infrastructure Layer"]
        DB[("PostgreSQL + pgvector")]
        REDIS[("Redis")]
        FILES[("Local File Storage")]
        WORKER["Background Worker<br/>Planned"]
    end

    AUTH --> DB
    PROFILE --> DB
    PROFILE --> FILES
    JOBS --> DB
    MATCH --> DB
    APPS --> DB
    APPROVAL --> DB

    API --> REDIS
    REDIS --> WORKER
    WORKER --> JOBS
    WORKER --> MATCH
    BROWSER --> APPROVAL


### High-level architecture

``` mermaid
flowchart TB
    U[User] --> FE[Next.js + TypeScript]
    FE -->|REST / JSON| API[FastAPI Backend]

    subgraph Backend["Backend modules - target architecture"]
      API --> AUTH[Authentication & Authorization]
      API --> PROF[Professional Profile]
      API --> JOBS[Job Discovery & Filtering]
      API --> MATCH[Matching & Ranking]
      API --> DOCS[Resume & Cover Letter]
      API --> APP[Application Management]
      API --> AUTO[Automation Controller]
      PROF --> AI[AI Orchestration]
      JOBS --> AI
      MATCH --> AI
      DOCS --> AI
      AI --> ROUTER[AI Provider Router]
      ROUTER --> GEM[Gemini - Primary]
      ROUTER -. retryable failure .-> OR[OpenRouter - Fallback]
    end

    AUTH --> DB[(PostgreSQL + pgvector)]
    PROF --> DB
    JOBS --> DB
    MATCH --> DB
    DOCS --> DB
    APP --> DB
    AUTO --> DB

    API <--> REDIS[(Redis)]
    API --> STORAGE[Local Document Storage]
    AUTO --> PW[Playwright - planned]
    REDIS -. future task queue .-> WORKER[Background Worker - planned]
    WORKER --> DB
    WORKER --> AI
```

### Runtime and deployment view

``` mermaid
flowchart TB
    subgraph Laptop["Ubuntu laptop - local environment"]
      BROWSER[Web Browser]
      FRONT[Next.js :3000]
      BACK[FastAPI :8000]
      subgraph DOCKER["Docker Compose"]
        PG[(PostgreSQL :5432)]
        RD[(Redis :6379)]
      end
      FILES[Local document storage]
    end

    BROWSER --> FRONT
    FRONT --> BACK
    BACK --> PG
    BACK --> RD
    BACK --> FILES
    BACK --> INTERNET[Internet]
    INTERNET --> GEM[Gemini API]
    INTERNET --> OR[OpenRouter API]
    INTERNET --> SOURCES[Supported job websites / APIs]
```

The database and Redis ports are bound to `127.0.0.1` in the supplied
Compose configuration. The backend and frontend are started locally
during development.

### Architecture layers

  -----------------------------------------------------------------------
  Layer                               Responsibility
  ----------------------------------- -----------------------------------
  Frontend                            Screens, forms, dashboards,
                                      document preview, user approval

  API layer                           HTTP endpoints, request/response
                                      schemas, status codes

  Middleware/core                     Configuration, request context,
                                      authentication, authorization

  Service layer                       Business rules and orchestration

  Repository layer                    Database queries and persistence

  AI layer                            Prompt construction, provider
                                      routing, output validation

  Data layer                          PostgreSQL, pgvector, Redis, local
                                      files

  Automation layer                    Controlled browser workflows and
                                      approval enforcement
  -----------------------------------------------------------------------

------------------------------------------------------------------------


## Request lifecycle

A request follows the relevant stages below. Not every endpoint requires
AI or a database write.

``` mermaid
flowchart TD
    A[User / Frontend] --> B[FastAPI endpoint]
    B --> C[Request ID and logging context]
    C --> D[Authentication]
    D --> E{Authenticated and authorized?}
    E -->|No| X[401 / 403 response]
    E -->|Yes| F[Pydantic schema validation]
    F --> G{Valid input?}
    G -->|No| Y[422 response]
    G -->|Yes| H[Normalization]
    H --> I[Business rules and filters]
    I --> J[Service layer]
    J --> K{AI required?}
    K -->|No| L[Repository / database]
    K -->|Yes| M[AI orchestration]
    M --> N[Gemini primary]
    N --> O{Retryable provider failure?}
    O -->|Yes| P[OpenRouter fallback]
    O -->|No| Q[Output and evidence validation]
    P --> Q
    Q --> L
    L --> R[Response serialization]
    R --> S[Audit / usage logging]
    S --> T[Frontend response]
```

### Lifecycle stages

1.  **Receive:** frontend sends an HTTP request.
2.  **Trace:** assign or propagate a request ID.
3.  **Authenticate:** identify the current user.
4.  **Authorize:** verify ownership of the requested resource.
5.  **Validate:** check input types, required fields, and limits.
6.  **Normalize:** standardize dates, URLs, labels, and text.
7.  **Filter:** apply the relevant business rules.
8.  **Execute:** call the service and repository layers.
9.  **AI orchestration:** call a provider only when the feature requires
    it.
10. **Validate output:** check response structure and evidence.
11. **Persist:** save the result if the operation requires it.
12. **Respond:** return a consistent response envelope.
13. **Observe:** record errors, latency, provider usage, and important
    actions.


## Job discovery and filter pipeline

The job pipeline turns raw opportunities into normalized, useful, and
ranked job records.

``` mermaid
flowchart TD
    A[Company career pages / supported boards / APIs / user input] --> B[Ingestion]
    B --> C[Raw job record]
    C --> D[Extract fields]
    D --> E[Schema validation]
    E --> F{Valid?}
    F -->|No| X[Reject or manual review]
    F -->|Yes| G[Normalize fields]
    G --> H[Duplicate detection]
    H --> I{Duplicate?}
    I -->|Yes| J[Merge or refresh existing record]
    I -->|No| K[Freshness / expiration check]
    J --> K
    K --> L{Active?}
    L -->|No| M[Archive / mark closed]
    L -->|Yes| N[Location and remote eligibility]
    N --> O[Employment type preferences]
    O --> P[Experience compatibility]
    P --> Q[Extract required and preferred skills]
    Q --> R[Profile matching]
    R --> S[Semantic analysis where useful]
    S --> T[Weighted ranking]
    T --> U[Store match explanation]
    U --> V[Recommendations dashboard]
```

### Filter stages

  -----------------------------------------------------------------------
  Stage                               Checks
  ----------------------------------- -----------------------------------
  Source validation                   Supported source, valid URL or
                                      manual source

  Schema validation                   Job title, company, description,
                                      source identifier

  Normalization                       Standardized location, dates,
                                      employment type, company name

  Duplicate detection                 URL, company/title/location, and
                                      similarity signals

  Freshness                           Expired, closed, unknown, or active

  Location                            Remote, hybrid, on-site,
                                      relocation, remote-country
                                      eligibility

  Employment type                     Internship, full-time, part-time,
                                      contract, freelance, graduate

  Experience                          Entry-level and years-of-experience
                                      compatibility

  Skill extraction                    Required and preferred
                                      technical/soft skills

  Profile matching                    Exact, related, missing, required,
                                      and preferred skills

  Ranking                             Weighted compatibility score and
                                      explanation
  -----------------------------------------------------------------------

A mismatch should usually lower a recommendation score rather than
automatically disqualify a job. Hard rejection should be reserved for
explicit constraints, such as a closed position or a user-defined
exclusion.

Job-source collection must respect the relevant website terms, access
controls, and rate limits. Do not bypass CAPTCHAs, authentication
controls, or anti-bot restrictions.

------------------------------------------------------------------------

## Job matching and ranking

The initial proposed score is:

\[ M = 0.40S + 0.25E + 0.20P + 0.15C \]

Where:

-   `S` = skills compatibility (0--100)
-   `E` = experience compatibility (0--100)
-   `P` = project relevance (0--100)
-   `C` = certification and education relevance (0--100)

Example:

  Component                     Score     Weight   Contribution
  --------------------------- ------- ---------- --------------
  Skills                           90        40%          36.00
  Experience                       70        25%          17.50
  Projects                         85        20%          17.00
  Certification / education        75        15%          11.25
  **Total**                             **100%**      **81.75**

Suggested display bands (configurable):

  Score      Label
  ---------- -----------------
  85--100    Excellent match
  70--84     Good match
  50--69     Moderate match
  Below 50   Low match

 The system should show the underlying matched and
missing requirements.

### Why pgvector?

Exact keyword matching misses related concepts. For example, a job
asking for "continuous integration and delivery" may be relevant to a
project describing GitHub Actions deployment pipelines. Embeddings can
help identify semantic similarity. pgvector keeps vector data in
PostgreSQL rather than requiring a separate vector database.

------------------------------------------------------------------------

## AI provider architecture

The backend uses a provider abstraction:

-   **Primary:** Gemini API.
-   **Fallback:** OpenRouter with the configured `openrouter/free`
    router.
-   **Secrets:** API keys are loaded only by the backend from
    environment configuration.
-   **Timeout:** configurable, default 60 seconds.
-   **Output limit:** configurable, default 4096 tokens.

### Provider routing

``` mermaid
flowchart TD
    A[Feature service] --> B[AI Router]
    B --> C[Gemini primary]
    C --> D{Success?}
    D -->|Yes| E[Validate output]
    D -->|No| F{Retryable failure?}
    F -->|No| G[Return provider error]
    F -->|Yes| H[OpenRouter fallback]
    H --> I[Validate output]
    E --> J[Feature response]
    I --> J
```

Fallback is intended for transient failures like network errors,
timeouts, HTTP 408, HTTP 429, and HTTP 5xx. Invalid credentials and
malformed requests should not be silently hidden by fallback.

Provider free-tier availability, model routing, quotas, and data-use
terms can change. Check the providers' current documentation before
relying on a free tier for ongoing use.


]

```

Approval must be enforced by the backend. A frontend button or
confirmation dialog alone is not a security boundary.

An approval should be bound to the exact application, resume version,
cover letter version, and relevant form data. Editing any approved
content invalidates the approval. Approval should be time-limited and
single-use.

------------------------------------------------------------------------

## Entity relationship diagram

The following is the **target logical ER model**. It describes the
intended full product schema, not the current Phase 1 database
implementation. Implement it incrementally with migrations.

``` mermaid
erDiagram
    USER ||--|| PROFILE : owns
    USER ||--o{ USER_PREFERENCE : configures
    USER ||--o{ JOB : saves
    USER ||--o{ RESUME : creates
    USER ||--o{ COVER_LETTER : creates
    USER ||--o{ APPLICATION : manages
    USER ||--o{ NOTIFICATION : receives
    USER ||--o{ AI_USAGE_LOG : generates
    USER ||--o{ AUDIT_LOG : produces

    PROFILE ||--o{ EDUCATION : contains
    PROFILE ||--o{ EXPERIENCE : contains
    PROFILE ||--o{ PROJECT : includes
    PROFILE ||--o{ CERTIFICATION : earns
    PROFILE ||--o{ PROFILE_SKILL : has
    SKILL ||--o{ PROFILE_SKILL : assigned_to
    SKILL ||--o{ JOB_SKILL : required_by

    JOB_SOURCE ||--o{ JOB : provides
    JOB ||--o{ JOB_SKILL : requires
    JOB ||--o{ JOB_MATCH : evaluated
    PROFILE ||--o{ JOB_MATCH : matched
    USER ||--o{ JOB_MATCH : receives

    RESUME ||--|{ RESUME_VERSION : has
    COVER_LETTER ||--|{ COVER_LETTER_VERSION : has

    JOB ||--o{ APPLICATION : receives
    RESUME_VERSION ||--o{ APPLICATION : selected_for
    COVER_LETTER_VERSION ||--o{ APPLICATION : selected_for

    APPLICATION ||--o{ APPLICATION_APPROVAL : requires
    APPLICATION ||--o{ APPLICATION_STATUS_HISTORY : tracks
    APPLICATION ||--o{ AUTOMATION_TASK : triggers
    APPLICATION ||--o{ INTERVIEW : schedules

    USER ||--o{ APPLICATION_APPROVAL : authorizes
    USER ||--o{ AUTOMATION_TASK : initiates

    USER {
        uuid id PK
        string email UK
        string password_hash
        string full_name
        boolean is_active
        datetime created_at
        datetime updated_at
    }

    PROFILE {
        uuid id PK
        uuid user_id FK
        string headline
        text summary
        string location
        string phone
        string linkedin_url
        string github_url
        datetime updated_at
    }

    USER_PREFERENCE {
        uuid id PK
        uuid user_id FK
        jsonb preferred_roles
        jsonb preferred_locations
        jsonb employment_types
        integer minimum_match_score
    }

    EDUCATION {
        uuid id PK
        uuid profile_id FK
        string institution
        string degree
        string field_of_study
        date start_date
        date end_date
        string grade
    }

    EXPERIENCE {
        uuid id PK
        uuid profile_id FK
        string company
        string job_title
        text description
        date start_date
        date end_date
    }

    PROJECT {
        uuid id PK
        uuid profile_id FK
        string title
        text description
        jsonb technologies
        string project_url
        date start_date
        date end_date
    }

    CERTIFICATION {
        uuid id PK
        uuid profile_id FK
        string name
        string issuer
        string credential_id
        string credential_url
        date issue_date
        date expiry_date
    }

    SKILL {
        uuid id PK
        string name UK
        string category
    }

    PROFILE_SKILL {
        uuid id PK
        uuid profile_id FK
        uuid skill_id FK
        string proficiency
        text evidence
    }

    JOB_SOURCE {
        uuid id PK
        string source_name
        string source_type
        string base_url
        boolean is_active
    }

    JOB {
        uuid id PK
        uuid user_id FK
        uuid source_id FK
        string company_name
        string job_title
        text description
        string location
        string employment_type
        string source_url
        datetime posted_at
        datetime expires_at
        string status
        datetime created_at
    }

    JOB_SKILL {
        uuid id PK
        uuid job_id FK
        uuid skill_id FK
        boolean is_required
        float importance
    }

    JOB_MATCH {
        uuid id PK
        uuid user_id FK
        uuid profile_id FK
        uuid job_id FK
        float skill_score
        float experience_score
        float project_score
        float education_score
        float total_score
        jsonb explanation
        datetime created_at
    }

    RESUME {
        uuid id PK
        uuid user_id FK
        string title
        string format
        string storage_path
        datetime created_at
    }

    RESUME_VERSION {
        uuid id PK
        uuid resume_id FK
        integer version_number
        uuid job_id FK
        string storage_path
        jsonb source_record_ids
        string content_hash
        datetime created_at
    }

    COVER_LETTER {
        uuid id PK
        uuid user_id FK
        string title
        datetime created_at
    }

    COVER_LETTER_VERSION {
        uuid id PK
        uuid cover_letter_id FK
        integer version_number
        uuid job_id FK
        text content
        jsonb source_record_ids
        string content_hash
        datetime created_at
    }

    APPLICATION {
        uuid id PK
        uuid user_id FK
        uuid job_id FK
        uuid resume_version_id FK
        uuid cover_letter_version_id FK
        string status
        datetime applied_at
        datetime created_at
        datetime updated_at
    }

    APPLICATION_APPROVAL {
        uuid id PK
        uuid application_id FK
        uuid user_id FK
        string approval_status
        string approved_content_hash
        datetime approved_at
        datetime expires_at
        datetime consumed_at
    }

    APPLICATION_STATUS_HISTORY {
        uuid id PK
        uuid application_id FK
        string previous_status
        string new_status
        text notes
        datetime changed_at
    }

    AUTOMATION_TASK {
        uuid id PK
        uuid application_id FK
        uuid user_id FK
        string task_status
        string browser_session_id
        jsonb execution_details
        datetime started_at
        datetime completed_at
    }

    INTERVIEW {
        uuid id PK
        uuid application_id FK
        datetime scheduled_at
        string interview_type
        string meeting_url
        text notes
        string status
    }

    AI_USAGE_LOG {
        uuid id PK
        uuid user_id FK
        string provider
        string model
        string feature
        integer input_tokens
        integer output_tokens
        integer latency_ms
        boolean fallback_used
        string status
        datetime created_at
    }

    AUDIT_LOG {
        uuid id PK
        uuid user_id FK
        string action
        string entity_type
        uuid entity_id
        jsonb metadata
        datetime created_at
    }

    NOTIFICATION {
        uuid id PK
        uuid user_id FK
        string title
        text message
        string notification_type
        boolean is_read
        datetime created_at
    }
```

### Relationship summary

  -----------------------------------------------------------------------------
  Relationship                  Cardinality             Meaning
  ----------------------------- ----------------------- -----------------------
  User--Profile                 1:1                     One user owns one
                                                        primary profile

  Profile--Education            1:N                     A profile can contain
                                                        multiple education
                                                        records

  Profile--Experience           1:N                     A profile can contain
                                                        multiple experience
                                                        records

  Profile--Project              1:N                     A profile can contain
                                                        multiple projects

  Profile--Certification        1:N                     A profile can contain
                                                        multiple certifications

  Profile--Skill                M:N                     A profile can have many
                                                        skills; a skill can
                                                        belong to many profiles

  Job--Skill                    M:N                     A job can require many
                                                        skills

  Job--JobMatch                 1:N                     A job can be matched
                                                        against multiple users

  Profile--JobMatch             1:N                     A profile can have
                                                        multiple match results

  Resume--ResumeVersion         1:N                     A resume can have
                                                        multiple saved versions

  CoverLetter--Version          1:N                     A cover letter can have
                                                        multiple versions

  User--Application             1:N                     A user can manage
                                                        multiple applications

  Job--Application              1:N                     Multiple users can
                                                        apply to the same job

  Application--Approval         1:N                     Approval attempts and
                                                        records are retained

  Application--AutomationTask   1:N                     An application can have
                                                        multiple automation
                                                        tasks

  Application--Interview        1:N                     An application can have
                                                        multiple interviews
  -----------------------------------------------------------------------------

### Database integrity recommendations

-   `users.email` must be unique.
-   `profiles.user_id` must be unique.
-   `profile_skills(profile_id, skill_id)` must be unique.
-   `job_skills(job_id, skill_id)` must be unique.
-   `(resume_id, version_number)` must be unique.
-   `(cover_letter_id, version_number)` must be unique.
-   Match scores must be constrained to 0--100.
-   All user-owned resources must be checked against the authenticated
    user.
-   Application records must reference the exact document versions used.
-   Approval records must be tied to an application and approved content
    hash.
-   Use indexes on foreign keys and commonly filtered columns.

**Implementation note:** the diagram is a logical target model. Physical
implementation may add columns such as `created_at`, `updated_at`,
soft-delete fields, source metadata, and indexes. Vector embedding
tables/extensions can be added when semantic matching is implemented.

------------------------------------------------------------------------

## Project structure

### Current starter archive structure

The Phase 1 archive is intentionally small:

``` text
jobpilot-ai-phase1-updated/
├── .env.example
├── .gitignore
├── README.md
├── docker-compose.yml
│
├── backend/
│   ├── Dockerfile
│   ├── requirements.txt
│   └── app/
│       ├── __init__.py
│       ├── main.py
│       ├── api/
│       │   ├── __init__.py
│       │   ├── health.py
│       │   └── ai.py
│       ├── ai/
│       │   ├── __init__.py
│       │   ├── providers.py
│       │   └── router.py
│       ├── core/
│       │   ├── __init__.py
│       │   ├── config.py
│       │   └── database.py
│       └── schemas/
│           ├── __init__.py
│           └── ai.py
│
├── frontend/
│   ├── package.json
│   ├── next-env.d.ts
│   ├── tsconfig.json
│   └── app/
│       ├── layout.tsx
│       └── page.tsx
│
├── storage/
│   └── .gitkeep
│
└── docs/
    └── phase-1-checklist.md
```

### Target structure as the application grows

``` text
jobpilot-ai/
├── frontend/
│   ├── app/
│   │   ├── login/
│   │   ├── dashboard/
│   │   ├── profile/
│   │   ├── jobs/
│   │   ├── matches/
│   │   ├── resumes/
│   │   ├── cover-letters/
│   │   ├── applications/
│   │   └── settings/
│   ├── components/
│   ├── hooks/
│   ├── services/
│   ├── types/
│   └── package.json
│
├── backend/
│   ├── app/
│   │   ├── main.py
│   │   ├── api/v1/
│   │   ├── core/
│   │   ├── models/
│   │   ├── schemas/
│   │   ├── repositories/
│   │   ├── services/
│   │   ├── ai/
│   │   │   ├── providers.py
│   │   │   ├── router.py
│   │   │   ├── prompts/
│   │   │   └── validators.py
│   │   ├── matching/
│   │   ├── automation/
│   │   ├── documents/
│   │   ├── workers/
│   │   └── utils/
│   ├── migrations/
│   ├── tests/
│   ├── requirements.txt
│   └── Dockerfile
│
├── storage/
│   ├── resumes/
│   ├── cover_letters/
│   └── generated_documents/
│
├── docs/
│   ├── architecture.md
│   ├── database-design.md
│   ├── api-documentation.md
│   └── development-roadmap.md
│
├── scripts/
├── docker-compose.yml
├── .env.example
├── .gitignore
└── README.md
```

The target structure is a design proposal. Add directories as the
corresponding features are implemented rather than creating empty
modules for every future feature immediately.

------------------------------------------------------------------------

## API documentation

The API is versioned under `/api/v1`. FastAPI generates interactive
documentation automatically.

### Base URLs

  Service          Local URL
  ---------------- --------------------------------------
  Frontend         `http://localhost:3000`
  Backend          `http://127.0.0.1:8000`
  Swagger UI       `http://127.0.0.1:8000/docs`
  ReDoc            `http://127.0.0.1:8000/redoc`
  OpenAPI schema   `http://127.0.0.1:8000/openapi.json`

### Implemented endpoints in the Phase 1 foundation

  Method   Endpoint                Description
  -------- ----------------------- --------------------------------------
  GET      `/`                     Service name, version, and docs link
  GET      `/health`               Basic API health check
  GET      `/health/ready`         Checks database connectivity
  POST     `/api/v1/ai/generate`   AI provider smoke test

#### `GET /health`

Example response:

``` json
{
  "status": "ok",
  "service": "jobpilot-api"
}
```

#### `GET /health/ready`

Executes a simple database query to verify connectivity.

Example response:

``` json
{
  "status": "ready",
  "database": "connected"
}
```

#### `POST /api/v1/ai/generate`

Request:

``` json
{
  "prompt": "Explain AWS EC2 in two sentences.",
  "system": "You are a concise cloud engineering assistant."
}
```

Response:

``` json
{
  "text": "EC2 is an AWS service that provides resizable compute capacity...",
  "provider": "gemini",
  "model": "gemini-3.5-flash-lite",
  "fallback_used": false
}
```

Validation constraints:

-   `prompt`: required string, 1--12,000 characters.
-   `system`: optional string, maximum 4,000 characters.

Error behavior:

-   Provider errors classified as retryable are returned as HTTP 503 if
    no provider succeeds.
-   Non-retryable provider errors are returned as HTTP 400.
-   Invalid request bodies are handled by FastAPI/Pydantic validation.

> The AI generation endpoint is currently a development smoke-test
> endpoint. It is not yet authenticated or rate-limited. Keep the
> backend bound to localhost and do not expose it to an untrusted
> network.

### Planned API endpoints

These endpoints describe the intended API surface; they are **not all
implemented yet**.

  -------------------------------------------------------------------------------------
  Method                  Endpoint                              Purpose
  ----------------------- ------------------------------------- -----------------------
  POST                    `/api/v1/auth/register`               Register account

  POST                    `/api/v1/auth/login`                  Authenticate

  GET                     `/api/v1/profile`                     Retrieve profile

  PUT                     `/api/v1/profile`                     Update profile

  POST                    `/api/v1/profile/education`           Add education

  POST                    `/api/v1/profile/experience`          Add experience

  POST                    `/api/v1/profile/projects`            Add project

  POST                    `/api/v1/profile/skills`              Add skill

  POST                    `/api/v1/profile/certifications`      Add certification

  POST                    `/api/v1/jobs`                        Create job record

  GET                     `/api/v1/jobs`                        List/filter jobs

  POST                    `/api/v1/jobs/import`                 Import a supported job

  GET                     `/api/v1/jobs/{job_id}`               Get job details

  POST                    `/api/v1/jobs/{job_id}/match`         Calculate job match

  GET                     `/api/v1/matches`                     List job matches

  POST                    `/api/v1/resumes/generate`            Generate tailored
                                                                resume

  GET                     `/api/v1/resumes`                     List resumes

  POST                    `/api/v1/cover-letters/generate`      Generate cover letter

  POST                    `/api/v1/applications`                Create application
                                                                draft

  GET                     `/api/v1/applications`                List applications

  POST                    `/api/v1/applications/{id}/approve`   Approve exact
                                                                application package

  POST                    `/api/v1/automation/{id}/prepare`     Prepare supported
                                                                browser form

  POST                    `/api/v1/automation/{id}/submit`      Submit after backend
                                                                approval check

  GET                     `/api/v1/dashboard/stats`             Dashboard statistics
  -------------------------------------------------------------------------------------

### API conventions for future endpoints

Successful response:

``` json
{
  "status": "success",
  "message": "Request completed successfully",
  "data": {},
  "request_id": "request-id"
}
```

Error response:

``` json
{
  "status": "error",
  "message": "Validation failed",
  "error_code": "VALIDATION_ERROR",
  "details": [],
  "request_id": "request-id"
}
```

Use appropriate HTTP status codes, pagination for list endpoints,
consistent error formats, and ownership checks on every user-owned
resource.

------------------------------------------------------------------------

## Quickstart

### Prerequisites

-   Ubuntu or another Linux environment.
-   Python 3.11 or later recommended.
-   Node.js 20 or later recommended.
-   npm.
-   Docker Engine and Docker Compose plugin.
-   Git.
-   Gemini API key.
-   OpenRouter API key for fallback testing.

The starter uses local PostgreSQL and Redis containers. The application
calls external AI APIs, so internet access is required for AI
generation.

### 1. Get the project

Extract the updated Phase 1 archive and enter the project directory:

``` bash
unzip jobpilot-ai-phase1-updated.zip
cd jobpilot-ai-phase1-updated
```

If cloning from a Git repository in the future, use the repository URL
provided by the project maintainer instead.

### 2. Create environment configuration

``` bash
cp .env.example .env
nano .env
```

Set a strong `SECRET_KEY`, a non-default PostgreSQL password, and your
API keys.

At minimum, configure:

``` dotenv
POSTGRES_PASSWORD=replace-with-a-strong-password
DATABASE_URL=postgresql+asyncpg://jobpilot:replace-with-a-strong-password@db:5432/jobpilot

GEMINI_API_KEY=your_gemini_api_key
OPENROUTER_API_KEY=your_openrouter_api_key
```

Do not share API keys in chat, screenshots, source control, or frontend
code.

### 3. Start PostgreSQL and Redis

From the project root:

``` bash
docker compose up -d db redis
docker compose ps
```

Check PostgreSQL:

``` bash
docker compose exec db pg_isready -U jobpilot -d jobpilot
```

Check Redis:

``` bash
docker compose exec redis redis-cli ping
```

Expected Redis response:

``` text
PONG
```

### 4. Set up the backend

Open a new terminal:

``` bash
cd jobpilot-ai-phase1-updated/backend

python3 -m venv .venv
source .venv/bin/activate

pip install --upgrade pip
pip install -r requirements.txt
```

Start FastAPI:

``` bash
DATABASE_URL="postgresql+asyncpg://jobpilot:YOUR_PASSWORD@localhost:5432/jobpilot" \
uvicorn app.main:app --reload --host 127.0.0.1 --port 8000
```

Replace `YOUR_PASSWORD` with the value of `POSTGRES_PASSWORD` in the
root `.env`.

The backend loads the root `.env` automatically. The explicit
`DATABASE_URL` override is needed for a backend process running on the
host, because the Compose hostname `db` is only resolvable inside the
Docker network.

Verify:

``` bash
curl http://127.0.0.1:8000/health
curl http://127.0.0.1:8000/health/ready
```

Open API documentation:

``` text
http://127.0.0.1:8000/docs
```

### 5. Test AI generation

With the backend running and API keys configured:

``` bash
curl -X POST http://127.0.0.1:8000/api/v1/ai/generate \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "Explain what AWS EC2 is in two sentences.",
    "system": "You are a concise cloud engineering assistant."
  }'
```

A successful response includes the generated text, provider, model, and
whether fallback was used.

### 6. Start the frontend

Open another terminal:

``` bash
cd jobpilot-ai-phase1-updated/frontend

npm install
npm run dev
```

Open:

``` text
http://localhost:3000
```

The frontend is currently a scaffold; the full dashboard and product
modules are planned work.

### 7. Stop local services

``` bash
docker compose down
```

To stop services and remove their persisted volumes (this permanently
deletes local database and Redis data):

``` bash
docker compose down -v
```

Do not use `-v` unless you intentionally want to delete the local data.

------------------------------------------------------------------------

## Configuration reference

The supplied `.env.example` contains the following settings.

  ---------------------------------------------------------------------------------
  Variable                 Purpose                 Default / example
  ------------------------ ----------------------- --------------------------------
  `APP_NAME`               Application display     `JobPilot AI`
                           name                    

  `APP_ENV`                Runtime environment     `development`

  `SECRET_KEY`             Secret for future       Replace placeholder
                           auth/session features   

  `POSTGRES_PASSWORD`      PostgreSQL password     Replace placeholder

  `DATABASE_URL`           SQLAlchemy async        Compose hostname `db`
                           database URL            

  `REDIS_URL`              Redis connection URL    `redis://redis:6379/0`

  `AI_PRIMARY_PROVIDER`    Primary AI provider     `gemini`

  `AI_FALLBACK_PROVIDER`   Fallback provider       `openrouter`

  `AI_REQUEST_TIMEOUT`     Provider timeout in     `60`
                           seconds                 

  `AI_MAX_OUTPUT_TOKENS`   Maximum output tokens   `4096`

  `GEMINI_API_KEY`         Gemini API credential   Empty until configured

  `GEMINI_MODEL`           Gemini model identifier `gemini-3.5-flash-lite`

  `OPENROUTER_API_KEY`     OpenRouter API          Empty until configured
                           credential              

  `OPENROUTER_MODEL`       OpenRouter model/router `openrouter/free`

  `OPENROUTER_BASE_URL`    OpenRouter API base URL `https://openrouter.ai/api/v1`

  `OPENROUTER_SITE_URL`    Application referrer    `http://localhost:3000`
                           metadata                

  `OPENROUTER_APP_NAME`    Application title       `JobPilot AI`
                           metadata                
  ---------------------------------------------------------------------------------

Model availability and provider model identifiers can change. Confirm
the configured model is currently supported by the provider if requests
fail.

------------------------------------------------------------------------

## Development workflow

### Backend

``` bash
cd backend
source .venv/bin/activate
uvicorn app.main:app --reload --host 127.0.0.1 --port 8000
```

### Frontend

``` bash
cd frontend
npm run dev
```

### Database

``` bash
docker compose up -d db redis
docker compose logs -f db
docker compose logs -f redis
```

### Migrations

Alembic is included as a dependency, but migration configuration and
application schema are not yet implemented in the Phase 1 foundation.
When database models are added, initialize and configure migrations
before creating production schema:

``` bash
cd backend
alembic init migrations
```

Then configure `alembic.ini` and the migration environment to use the
application's SQLAlchemy metadata and database settings. Do not run
autogenerated migrations until the models and metadata are configured.

### Testing

Automated test coverage is planned. Add tests for:

-   Provider routing and fallback classification.
-   Request validation and error responses.
-   Database repositories and constraints.
-   User ownership and authorization.
-   Job filtering and duplicate detection.
-   Match score calculations.
-   Generated document evidence validation.
-   Approval expiration, revocation, and single-use behavior.
-   Submission endpoint approval enforcement.

------------------------------------------------------------------------

## Security and privacy

JobPilot AI stores sensitive career information, including contact
details, employment history, resumes, and application records.

### Required safeguards

-   Keep `.env` out of Git.
-   Use strong, unique secrets and database passwords.
-   Never expose provider API keys to the browser.
-   Keep database and Redis ports bound to localhost for local use.
-   Add authentication and per-user authorization before multi-user use.
-   Add rate limits and request-size limits before exposing API
    endpoints.
-   Store uploaded/generated documents in private directories.
-   Validate file types and impose file-size limits.
-   Use parameterized database operations through SQLAlchemy.
-   Treat job descriptions and website content as untrusted data.
-   Never execute arbitrary AI-generated shell commands.
-   Keep audit records for approvals and submissions.
-   Back up the database and document directory.
-   Review external AI providers' data retention and training policies
    before sending personal information.

### Human approval requirement

The browser automation module must not submit an application merely
because an AI model recommends doing so. The backend must verify an
explicit, current, single-use approval bound to the exact application
and document versions.

### Development endpoint warning

The current `/api/v1/ai/generate` endpoint is a development smoke test.
It does not yet implement user authentication, per-user quotas, or
production-grade rate limiting. Keep it local until these controls are
added.

### External AI data handling

Gemini and OpenRouter are external services. Requests may contain
personal information if the application sends it. Minimize and redact
data where practical, review current provider terms, and obtain the
user's consent before transmitting sensitive profile content.

------------------------------------------------------------------------

## Resource and deployment notes

The initial Compose setup limits resource use:

  Service      Image                        Memory limit   CPU limit
  ------------ -------------------------- -------------- -----------
  PostgreSQL   `pgvector/pgvector:pg16`          1536 MB         1.0
  Redis        `redis:7-alpine`                   256 MB         0.5

The AI inference is cloud-based; local GPU inference is not part of the
current architecture. Playwright and background workers should be
introduced incrementally to avoid unnecessary resource consumption.

The Compose file persists database and Redis data in named Docker
volumes. Resume and generated document storage is intended to remain in
the project's local storage directory.

------------------------------------------------------------------------

## Roadmap

  -----------------------------------------------------------------------
  Phase                   Scope                   Status
  ----------------------- ----------------------- -----------------------
  1                       Project foundation,     Foundation created;
                          local services, AI      live setup/API
                          provider adapters,      verification pending
                          health checks           

  2                       PostgreSQL schema,      Planned
                          Alembic migrations,     
                          master professional     
                          profile CRUD            

  3                       Authentication,         Planned
                          authorization, document 
                          upload and parsing      

  4                       Job ingestion,          Planned
                          normalization,          
                          filtering and           
                          deduplication           

  5                       Skill extraction, job   Planned
                          matching, ranking and   
                          pgvector                

  6                       Evidence-backed resume  Planned
                          and cover letter        
                          generation              

  7                       Application tracker,    Planned
                          status history and      
                          approval records        

  8                       Playwright form         Planned
                          preparation and         
                          human-approved          
                          submission              

  9                       Background workers,     Planned
                          notifications,          
                          observability, backups  
                          and hardening           
  -----------------------------------------------------------------------

### Immediate next milestone

Implement Phase 2:

1.  Create SQLAlchemy models for `User`, `Profile`, `Education`,
    `Experience`, `Skill`, `ProfileSkill`, `Project`, and
    `Certification`.
2.  Configure Alembic migrations.
3.  Implement profile CRUD services and API endpoints.
4.  Add database constraints and tests.
5.  Build the profile management UI.
6.  Ensure every profile record is owned by the authenticated user.

------------------------------------------------------------------------

## Troubleshooting

### PostgreSQL connection error

If the backend runs directly on Ubuntu, use `localhost` in the host-run
`DATABASE_URL`:

``` bash
DATABASE_URL="postgresql+asyncpg://jobpilot:YOUR_PASSWORD@localhost:5432/jobpilot"
```

If the backend runs inside Docker Compose, use the service hostname
`db`.

### PostgreSQL container is not healthy

``` bash
docker compose ps
docker compose logs db
docker compose exec db pg_isready -U jobpilot -d jobpilot
```

### Redis connection error

``` bash
docker compose logs redis
docker compose exec redis redis-cli ping
```

### AI provider error

Check:

-   API key is present in the root `.env`.
-   Key has not expired or been revoked.
-   Model identifier is supported.
-   Internet connectivity is available.
-   Provider quota/rate limits have not been reached.
-   The backend was restarted after configuration changes.

Never paste API keys into issue reports. Share only redacted error
messages.

### Frontend dependency or startup issue

``` bash
cd frontend
node --version
npm --version
npm install
npm run dev
```

------------------------------------------------------------------------

## Contributing

Contributions and suggestions are welcome.

Before implementing a feature:

1.  Check the roadmap and current implementation status.
2.  Keep business logic in services rather than API route handlers.
3.  Validate input and output with Pydantic schemas.
4.  Add database migrations for schema changes.
5.  Add tests for business rules and security boundaries.
6.  Keep secrets and personal data out of commits and logs.
7.  Preserve the human approval requirement for application submission.

Suggested branch naming:

``` text
feature/profile-management
feature/job-ingestion
feature/job-matching
fix/ai-fallback
docs/api-reference
```

------------------------------------------------------------------------

## License

No license has been declared yet. Until a license is added to the
repository, all rights are reserved by the copyright holder. Add a
`LICENSE` file before distributing or reusing the project under an
open-source license.

------------------------------------------------------------------------

**JobPilot AI --- Find opportunities. Understand your fit. Prepare with
confidence. Apply with control.**
