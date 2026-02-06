# Gemini Diagram Generation Prompts for Wahb Project

**Date:** February 6, 2026
**Purpose:** 4 detailed prompts for generating accurate system diagrams

---

## Prompt 1: Overall System Architecture Diagram

```
You are a technical diagram designer. Create a comprehensive system architecture diagram for the "Wahb" platform - a mobile-first social content platform with dual-mode discovery.

### System Overview
Wahb is a mono-repo with 5 services + 1 consumer frontend app:
1. Wahb-Platform (Next.js) - Consumer UI for For You + News feeds
2. Content-Management-System (Go) - CMS + Feed Service + Interactions
3. Aggregation-Service (Node.js) - Content ingestion + media processing workers
4. Platform-Console (Next.js) - Admin dashboard (Platform ops + CRM)
5. CRM-Service (Go) - Customer, deals, activities management
6. Shared Storage: PostgreSQL 15+ with pgvector, Redis (caching/queues), Supabase Storage/S3 (media artifacts), CDN

### Service Boundaries (CRITICAL - Visualize These Clearly)

```
┌─────────────────────────────────────────────────────────────┐
│  Aggregation Service (Node.js)                              │
│  - Scrape external sources (RSS, YouTube, X, Reddit)        │
│  - Run FFmpeg (audio → MP4)                                 │
│  - Generate transcripts (Whisper)                           │
│  - Generate embeddings (384-dim vectors)                    │
│  - Upload to object storage                                 │
│  - Write to CMS via APIs                                    │
│                                                             │
│  CANNOT: Serve user APIs, assemble feeds                    │
└─────────────────────────────────────────────────────────────┘
                          ↓ writes via APIs
┌─────────────────────────────────────────────────────────────┐
│  CMS (Go)                                                   │
│  - Read content_items                                       │
│  - Execute pgvector similarity                              │
│  - Assemble feeds (For You, News)                          │
│  - Serve public APIs (/api/v1/feed/*)                      │
│  - Content CRUD, interactions tracking                      │
│  - Issue JWTs for admin auth                                │
│                                                             │
│  CANNOT: Scrape, run FFmpeg, ingest external sources        │
└─────────────────────────────────────────────────────────────┘
                ↑ serves JSON            ↑ serves JSON
┌──────────────────┐         ┌──────────────────┐
│  Wahb-Platform   │         │  Platform-Console │
│  (Next.js)        │         │  (Next.js)       │
│                   │         │                  │
│  - Feed UI        │         │  - Platform ops  │
│  - Interactions   │         │  - CRM UI        │
│  - Bottom nav     │         │  - Source mgmt   │
│                   │         │                  │
│  CANNOT: Scrape,  │         │  - Calls CMS +   │
│  FFmpeg, DB write│         │    CRM APIs      │
└──────────────────┘         └──────────────────┘
      ↑ auth via CMS            ↑ auth via CMS
      (JWT Bearer)             (JWT Bearer)
                                ↓ verifies JWT
                        ┌──────────────────┐
                        │  CRM Service     │
                        │  (Go)            │
                        │                  │
                        │  - Customers     │
                        │  - Contacts      │
                        │  - Deals         │
                        │  - Activities    │
                        │  - Notes, Tags   │
                        │                  │
                        │  CANNOT: Content │
                        │    ingestion,    │
                        │    feed serving  │
                        └──────────────────┘
```

### Key External Sources (Show These Feeding Into Aggregation)
- RSS news feeds
- Podcast RSS / iTunes search
- YouTube Data API v3
- X/Twitter API
- Reddit API
- Manual uploads

### Key Data Flows (Label Arrows Clearly)
1. External Sources → Aggregation Service (fetch)
2. Aggregation Service → CMS (write content_items, transcripts, embeddings)
3. Aggregation Service → Object Storage (upload media)
4. CMS → PostgreSQL (read content_items, execute pgvector)
5. CMS → Wahb-Platform (serve feed JSON)
6. Platform-Console → CMS (admin: manage sources, trigger ingestion)
7. Platform-Console → CRM Service (CRM workflows)
8. CMS → Platform-Console (issue JWT for login)
9. CRM Service → Platform-Console (verify JWT, serve CRM data)

### Database Layer (Shared PostgreSQL)
- Tables: content_items, transcripts, user_interactions, content_sources
- CRM tables: customers, contacts, deals, activities, notes, tags
- Extensions: pgcrypto (UUID), vector (384-dim embeddings)

### Visual Style Requirements
- Use distinct colors for each service
- Show clear boundaries (boxes/shapes) for each service
- Label all arrows with action types (read/write/serve/verify)
- Include external sources on the left/top
- Place storage layer at the bottom
- Show JWT auth flow with distinct arrow style
- Add legend for service types (Frontend, Backend, Workers, Storage)

### Technical Stack Labels (Add as Small Tags)
- Wahb-Platform: Next.js 15, React 19, shadcn/ui, Zustand, TanStack Query
- CMS: Go, Gin, GORM, PostgreSQL, pgvector
- Aggregation: Node.js, Fastify, BullMQ, Redis, FFmpeg, Whisper, yt-dlp
- Platform-Console: Next.js, TypeScript
- CRM: Go, Gin, GORM, PostgreSQL, JWT (HS256)
- Storage: PostgreSQL 15+, pgvector, Redis, Supabase Storage/S3, CDN

Generate a comprehensive architecture diagram that shows all 6 components, their boundaries, data flows, and the strict separation of concerns between services.
```

---

## Prompt 2: Authentication & Authorization Flow Diagram

```
You are a technical diagram designer. Create a detailed authentication and authorization flow diagram for the Wahb platform.

### Auth Architecture Overview
Wahb uses a shared JWT-based authentication model with one centralized issuer:
- CMS (Go) is the JWT issuer - creates and signs tokens
- CRM Service (Go) is the verifier only - validates tokens and enforces RBAC
- Platform-Console (Next.js) is the client - stores and presents tokens
- JWT Algorithm: HS256 with shared JWT_SECRET between CMS and CRM

### Complete Auth Flow (Step-by-Step Sequence)

#### Step 1: Admin Login Flow
```
Platform-Console                    CMS (Go)
     │                                │
     │  POST /admin/login             │
     │  { email, password }           │
     │───────────────────────────────>│
     │                                │
     │                        Validate credentials
     │                                │
     │  HTTP 200 OK                   │
     │  {                             │
     │    token: "eyJ...",            │
     │    user: { id, role, name }    │
     │  }                             │
     │<───────────────────────────────│
     │                                │
     │  Store token in localStorage   │
     │  Redirect to /dashboard        │
```

#### Step 2: Using CMS Admin APIs (Platform Operations)
```
Platform-Console                    CMS (Go)
     │                                │
     │  GET /admin/content_sources    │
     │  Authorization: Bearer <token> │
     │───────────────────────────────>│
     │                                │
     │                        Verify JWT signature
     │                        Extract claims (sub, role)
     │                        Check role permissions
     │                                │
     │  HTTP 200 OK                   │
     │  { content_sources: [...] }    │
     │<───────────────────────────────│
```

#### Step 3: Using CRM APIs (CRM Workflows)
```
Platform-Console                    CRM Service (Go)
     │                                │
     │  GET /admin/customers          │
     │  Authorization: Bearer <token> │
     │───────────────────────────────>│
     │                                │
     │                        Verify JWT signature
     │                        Extract claims (sub, role)
     │                        Check RBAC: admin/manager/agent
     │                        Filter data by role permissions
     │                                │
     │  HTTP 200 OK                   │
     │  { customers: [...] }          │
     │<───────────────────────────────│
```

#### Step 4: Token Error Handling
```
Platform-Console          CMS/CRM Service
     │                        │
     │  Request with          │
     │  missing/expired       │
     │  token                 │
     │───────────────────────>│
     │                        │
     │                 Verify fails
     │                        │
     │  HTTP 401 Unauthorized │
     │  {                     │
     │    code: 401,          │
     │    message: "Invalid   │
     │      or missing token" │
     │  }                     │
     │<───────────────────────│
     │                        │
     │  Clear localStorage     │
     │  Redirect to /login    │
```

#### Step 5: Role-Based Access Control (RBAC)
```
Roles and Permissions:

Admin (full access):
├── CMS: All /admin/* endpoints
│   ├── content_sources (CRUD)
│   ├── content_items (view all, trigger ingestion)
│   └── Issue JWTs
└── CRM: All /admin/* endpoints
    ├── customers (CRUD, including all records)
    ├── deals (full pipeline management)
    ├── activities (all records)
    ├── reports (all reports)
    └── tags (manage all)

Manager (team management):
├── CMS: Read-only /admin/* endpoints
└── CRM: /admin/customers (view all), /admin/deals (all stages),
         /admin/activities (team tasks), /admin/reports (overview)

Agent/SalesRep (assigned records only):
├── CMS: No direct access (via Console only)
└── CRM: /admin/customers (assigned only), /admin/deals (assigned only),
         /admin/activities (my tasks), /admin/me/activities
```

### JWT Token Structure
```json
{
  "header": {
    "alg": "HS256",
    "typ": "JWT"
  },
  "payload": {
    "sub": "user-uuid-or-id",        // User ID
    "user_id": "user-uuid-or-id",    // Alternative field
    "role": "admin|manager|agent",   // RBAC role
    "name": "Full Name",             // Display name
    "email": "user@example.com",     // Email
    "iat": 1234567890,               // Issued at
    "exp": 1234570490                // Expiration (1 hour)
  }
}
```

### Shared Secret Configuration
```
CMS .env:
├── JWT_SECRET (shared with CRM)
├── JWT_ISSUER: "wahb-cms"
└── TOKEN_EXPIRY: "1h"

CRM Service .env:
├── JWT_SECRET (MUST match CMS)
├── JWT_ISSUER: "wahb-cms" (for validation)
└── HS256 verification only (no issuing)

Platform-Console .env:
├── NEXT_PUBLIC_CMS_BASE_URL
├── NEXT_PUBLIC_CRM_BASE_URL
└── No JWT_SECRET stored client-side
```

### CORS Requirements (Cross-Origin Security)
```
Platform-Console (Vercel) → CMS/CRM APIs:
├── Origin: *.vercel.app, localhost:3000
├── Allowed Headers: Authorization, Content-Type
├── Allowed Methods: GET, POST, PUT, PATCH, DELETE, OPTIONS
└── Credentials: not used (Bearer token in header)
```

### Visual Elements Required
1. Swimlane diagram showing: Platform-Console | CMS | CRM Service
2. Sequence flow for login (with numbered steps)
3. Token validation sequence for CMS API calls
4. Token validation sequence for CRM API calls
5. Error handling flow (401/403 responses)
6. RBAC decision tree (which role can access which endpoints)
7. JWT token structure as a visual card
8. Shared secret configuration as a configuration diagram
9. CORS setup as a security boundary diagram

### Color Coding
- Green: Successful auth flow
- Red: Error/Failed auth (401/403)
- Blue: JWT token lifecycle
- Yellow: Security boundaries (CORS, secret sharing)

Generate a sequence diagram + flowchart hybrid showing the complete auth lifecycle.
```

---

## Prompt 3: Aggregation Service Content Pipeline Flow

```
You are a technical diagram designer. Create a detailed content aggregation and processing pipeline diagram for the Aggregation Service in the Wahb platform.

### Aggregation Service Purpose
The Aggregation Service is an asynchronous Node.js worker fleet that:
- Ingests content from 6 external sources
- Processes media (downloads, transcodes to MP4)
- Generates transcripts (Whisper) and embeddings (384-dim vectors)
- Uploads artifacts to object storage
- Writes content back to CMS via internal APIs
- NEVER serves user-facing API traffic

### External Sources (Input Layer)
```
┌─────────────────────────────────────────────────────────────────┐
│                    EXTERNAL SOURCES                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Manual Uploads         2. RSS News Feeds                    │
│     └── Via CMS admin          └── Per-domain allowlist          │
│                                  └── Full article scraping       │
│                                                                  │
│  3. Podcast RSS            4. YouTube Data API v3               │
│     └── iTunes Search           └── Channel/playlist feeds      │
│     └── RSS feeds              └── Video metadata + streams     │
│                                                                  │
│  5. X/Twitter API           6. Reddit API                        │
│     └── Timeline/search         └── Subreddit/post fetching     │
│     └── Engagement filter       └── Comment/thread extraction   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Pipeline Stages (Processing Flow)

#### Stage 1: FETCH
```
┌─────────────────────────────────────────────────────────────┐
│  FETCH WORKERS (BullMQ Queue: fetch-queue)                  │
│                                                              │
│  RSS Sources:           Podcast/YouTube:                    │
│  ├── Poll RSS feed      ├── Poll channel/feed              │
│  ├── Parse items        ├── Get metadata                   │
│  └── Extract URLs       └── Extract media URLs             │
│                                                              │
│  Social APIs:           Manual Upload:                      │
│  ├── Fetch recent       ├── Receive from CMS                │
│  ├── Filter engagement  └── Validate input                  │
│  └── Extract thread                                         │
│                                                              │
│  Output: Raw source data + metadata                          │
│  Status: PENDING → PROCESSING                                │
└─────────────────────────────────────────────────────────────┘
```

#### Stage 2: NORMALIZE
```
┌─────────────────────────────────────────────────────────────┐
│  NORMALIZE WORKERS (BullMQ Queue: normalize-queue)          │
│                                                              │
│  Map to ContentItem Schema:                                 │
│  ├── Type: ARTICLE|VIDEO|TWEET|COMMENT|PODCAST             │
│  ├── Source: RSS|PODCAST|YOUTUBE|UPLOAD|MANUAL             │
│  ├── Fields: title, body_text, excerpt                      │
│  ├── Attribution: author, source_name, source_feed_url      │
│  ├── Timestamps: published_at, created_at                   │
│  └── Idempotency key: canonical URL OR hash(title+date)     │
│                                                              │
│  Deduplication Check:                                       │
│  └── Check if idempotency key exists in CMS                 │
│      └── Skip if exists (upsert)                             │
│      └── Proceed if new                                     │
│                                                              │
│  Output: Normalized ContentItem JSON                         │
│  Action: Upsert to CMS via /admin/content_items             │
└─────────────────────────────────────────────────────────────┘
```

#### Stage 3: MEDIA
```
┌─────────────────────────────────────────────────────────────┐
│  MEDIA WORKERS (BullMQ Queue: media-queue)                  │
│                                                              │
│  Download:              Transcode (FFmpeg):                 │
│  ├── yt-dlp (video)     ├── Audio → MP4 container          │
│  ├── curl (audio)       ├── Extract thumbnail              │
│  ├── wget (articles)    ├── Get duration_sec               │
│  └── Handle retries     └── Optimize for mobile            │
│                                                              │
│  Upload to Storage:     Update CMS:                         │
│  ├── Supabase Storage   ├── media_url                      │
│  │   or S3-compatible   ├── thumbnail_url                  │
│  ├── Original file      ├── original_url                   │
│  ├── Processed MP4      └── duration_sec                   │
│  └── Thumbnail image                                         │
│                                                              │
│  CDN: Publish URLs for frontend delivery                     │
│  Status: Update content_item to PROCESSING                  │
└─────────────────────────────────────────────────────────────┘
```

#### Stage 4: TRANSCRIPT
```
┌─────────────────────────────────────────────────────────────┐
│  TRANSCRIPT WORKERS (BullMQ Queue: transcript-queue)        │
│                                                              │
│  For audio/video content only:                              │
│  ├── Download media from storage                            │
│  ├── Run Whisper (transcription)                            │
│  ├── Generate: full_text, summary, word_timestamps          │
│  ├── Detect language                                        │
│  └── Store in transcripts table via CMS API                 │
│                                                              │
│  Output: Transcript record linked to content_item_id        │
│  Action: POST /admin/transcripts + link to ContentItem      │
└─────────────────────────────────────────────────────────────┘
```

#### Stage 5: EMBEDDINGS
```
┌─────────────────────────────────────────────────────────────┐
│  EMBEDDINGS WORKERS (BullMQ Queue: embeddings-queue)        │
│                                                              │
│  Generate 384-dim vectors:                                  │
│  ├── Model: all-MiniLM-L6-v2 (or equivalent)               │
│  ├── Input: title + excerpt + body_text                     │
│  ├── Output: vector(384) stored in content_items.embedding  │
│  └── Enable semantic similarity via pgvector                │
│                                                              │
│  Update CMS:                                                │
│  └── PATCH content_item with embedding vector               │
│                                                              │
│  Status: Set to READY (all required artifacts complete)     │
└─────────────────────────────────────────────────────────────┘
```

### Status Workflow
```
┌─────────┐     ┌─────────────┐     ┌──────────┐     ┌────────┐
│ PENDING │ ───>│ PROCESSING  │ ───>│  READY   │ ───>│ FAILED │
└─────────┘     └─────────────┘     └──────────┘     └────────┘
                      │                                      │
                      │                                      │
                      v                                      v
                 (any stage                           (retry
                  fails)                              with
                                                  backoff)
```

### CMS Integration (Service-to-Service)
```
Aggregation Service                    CMS (Go)
     │                                    │
     │  POST /admin/content_items          │
     │  { ContentItem (upsert) }           │
     │  Authorization: Bearer <service_token>
     │───────────────────────────────────>│
     │                                    │
     │                        GORM Upsert by
     │                        idempotency_key
     │                                    │
     │  201 Created                       │
     │  { id, status, ... }               │
     │<───────────────────────────────────│
     │                                    │
     │  PATCH /admin/content_items/:id     │
     │  { media_url, status, ... }        │
     │───────────────────────────────────>│
     │                                    │
     │                        Update fields
     │                        Transition status
     │                                    │
     │  200 OK                            │
     │<───────────────────────────────────│
```

### Idempotency & Deduplication
```
Primary Key Strategy:
├── If source has canonical URL → use as idempotency_key
├── If no URL (e.g., manual) → hash(title + published_at)
└── All CMS writes use upsert by idempotency_key

Prevents:
├── Duplicate content from re-runs
├── Multiple workers processing same item
└── Re-processing already ingested sources
```

### Retry Strategy (BullMQ)
```
┌─────────────────────────────────────────────────────────────┐
│  BullMQ Job Configuration                                    │
│                                                              │
│  Queue Settings:                                             │
│  ├── connection: Redis                                      │
│  ├── defaultJobOptions:                                     │
│  │   ├── attempts: 3                                        │
│  │   ├── backoff: 'exponential'                             │
│  │   ├── delay: 1000 (initial)                              │
│  │   └── removeOnComplete: 100 (keep last 100)             │
│  └── workers: concurrency=5 (tunable)                       │
│                                                              │
│  Transient Failures (Retry):                                │
│  ├── Rate limits (429)                                      │
│  ├── Network timeouts                                       │
│  ├── Temporary storage failures                             │
│  └── Scraping retries                                       │
│                                                              │
│  Permanent Failures (Mark FAILED):                          │
│  ├── Invalid source URLs                                    │
│  ├── Corrupted media files                                  │
│  ├── Failed transcoding (retries exhausted)                 │
│  └── CMS validation errors                                  │
└─────────────────────────────────────────────────────────────┘
```

### Technology Stack (Tag Each Stage)
```
Fetch Stage:      node-fetch, axios, rss-parser, yt-dlp
Normalize Stage:  lodash, date-fns, validator
Media Stage:      FFmpeg, yt-dlp, @aws-sdk/client-s3
Transcript Stage: Whisper (openai/whisper), ffmpeg
Embeddings Stage: @xenova/transformers (all-MiniLM-L6-v2)
Queue:            BullMQ, Redis (ioredis)
Storage:          Supabase Storage or S3-compatible
CMS Client:       axios/fetch with service token
```

### Artifacts Produced Per Content Type
```
ARTICLE:
├── Original HTML scraped (allowlisted domains only)
├── Cleaned body_text + excerpt
├── Extracted metadata (author, published_at)
└── Embedding vector (384-dim)

VIDEO/PODCAST:
├── Downloaded original media file
├── Transcoded MP4 (required for For You feed)
├── Generated thumbnail image
├── Duration in seconds
├── Transcript (full text + summary + timestamps)
└── Embedding vector

TWEET/COMMENT:
├── Captured text + author
├── Thread context (if reply)
├── Engagement metrics
└── Embedding vector
```

### Visual Style Requirements
1. Horizontal left-to-right pipeline flow
2. Color-coded stages (Fetch=Blue, Normalize=Green, Media=Orange, Transcript=Purple, Embeddings=Red)
3. Show external sources feeding into Stage 1
4. Show CMS API calls after each stage
5. Show storage uploads for media artifacts
6. Include status transitions as a parallel track
7. Add retry logic as feedback loops
8. Label all tools/APIs used in each stage
9. Show idempotency check as a decision gate
10. Include "DO NOT CROSS" boundaries (no user-facing APIs)

Generate a comprehensive pipeline diagram showing all 5 stages, their tooling, data flows, CMS integration, and error handling.
```

---

## Prompt 4: Technology Stack & External Integrations Diagram

```
You are a technical diagram designer. Create a comprehensive technology stack and external integrations diagram for the Wahb platform.

### System Components (6 Main Components)

#### 1. Wahb-Platform (Consumer Frontend)
```
┌─────────────────────────────────────────────────────────────┐
│  WAHB-PLATFORM (Next.js 15) - Consumer UI                   │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Framework:              UI Components:                     │
│  ├── Next.js 15 (App Router) ├── shadcn/ui + Radix UI      │
│  ├── React 19             ├── Framer Motion (animations)   │
│  └── TypeScript           └── Tailwind CSS                 │
│                                                              │
│  State Management:      Data Fetching:                      │
│  ├── Zustand (client)    ├── TanStack Query (React Query)  │
│  └── Session persistence └── API client (axios/fetch)      │
│                                                              │
│  Features:                                                   │
│  ├── For You feed (TikTok-style snap scroll)               │
│  ├── News feed (Magazine slides: 1+3)                      │
│  ├── Like, bookmark, share interactions                     │
│  ├── View tracking (Intersection Observer)                  │
│  ├── Bottom navigation                                      │
│  └── Feed switcher (For You ↔ News)                        │
│                                                              │
│  Deployment: Vercel (Production)                            │
└─────────────────────────────────────────────────────────────┘
```

#### 2. Content-Management-System (Go Backend)
```
┌─────────────────────────────────────────────────────────────┐
│  CMS / FEED SERVICE (Go 1.21+)                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Framework:              Database:                          │
│  ├── Gin (HTTP router)   ├── PostgreSQL 15+                │
│  ├── GORM (ORM)          ├── pgvector extension             │
│  └── lib/pg              ├── pgcrypto (UUID)               │
│                          └── Connection via DATABASE_URL   │
│                                                              │
│  APIs Provided:         Key Responsibilities:               │
│  ├── /api/v1/feed/foryou ├── Feed assembly (For You/News) │
│  ├── /api/v1/feed/news   ├── pgvector similarity queries   │
│  ├── /api/v1/content/:id ├── JWT issuance (HS256)          │
│  ├── /api/v1/interactions └── Content CRUD                 │
│  ├── /api/v1/interactions/bookmarks                        │
│  ├── /admin/* (platform ops)                                │
│  └── /admin/login (JWT issuer)                              │
│                                                              │
│  Admin Auth:            Deployment:                         │
│  ├── HS256 JWT signing  ├── Docker container                │
│  ├── JWT_SECRET (shared) └── DATABASE_URL only config      │
│  └── CORS for Platform-Console                              │
└─────────────────────────────────────────────────────────────┘
```

#### 3. Aggregation-Service (Node.js Workers)
```
┌─────────────────────────────────────────────────────────────┐
│  AGGREGATION SERVICE (Node.js LTS)                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Framework:              Queue System:                      │
│  ├── Fastify (minimal)   ├── BullMQ (job queues)            │
│  └── Worker processes    ├── Redis (queue backend)         │
│                          └── 5 stage queues                │
│                                                              │
│  Media Processing:      AI/ML:                              │
│  ├── FFmpeg (transcode) ├── Whisper (transcription)        │
│  ├── yt-dlp (download)  └── all-MiniLM-L6-v2 (embeddings)  │
│  └── ImageMagick (thumbnails)                               │
│                                                              │
│  External Source APIs:  Storage:                            │
│  ├── YouTube Data API v3 ├── Supabase Storage OR           │
│  ├── X/Twitter API       │   S3-compatible object storage  │
│  ├── Reddit API          ├── CDN (media delivery)           │
│  ├── Podcast RSS         └── STORAGE_BASE_URL config       │
│  ├── RSS feeds                                              │
│  └── iTunes Search API                                       │
│                                                              │
│  CMS Integration:      Deployment:                          │
│  ├── Service token auth ├── Docker container                │
│  └── Writes via /admin/* └── REDIS_URL required            │
└─────────────────────────────────────────────────────────────┘
```

#### 4. Platform-Console (Admin Dashboard)
```
┌─────────────────────────────────────────────────────────────┐
│  PLATFORM CONSOLE (Next.js 15) - Admin UI                   │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Framework:              UI Components:                     │
│  ├── Next.js 15 (App Router) ├── shadcn/ui + Radix UI      │
│  ├── React 19             └── TanStack Table (data grids)  │
│  └── TypeScript                                              │
│                                                              │
│  State & Auth:         Features:                            │
│  ├── TanStack Query     ├── Platform Operations:            │
│  ├── JWT in localStorage  │  ├── Content sources CRUD       │
│  └── API clients (CMS+CRM)│  ├── Trigger ingestion         │
│                          │  └── Content monitoring         │
│                          └── CRM Workflows:                 │
│                             ├── Customers, contacts, deals  │
│                             ├── Activities, tasks, tags     │
│                             └── Reports & analytics        │
│                                                              │
│  Deployment: Vercel (Production)                            │
│  Environment: NEXT_PUBLIC_CMS_BASE_URL, NEXT_PUBLIC_CRM_BASE_URL
└─────────────────────────────────────────────────────────────┘
```

#### 5. CRM-Service (Go CRM Backend)
```
┌─────────────────────────────────────────────────────────────┐
│  CRM SERVICE (Go 1.21+)                                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Framework:              Database:                          │
│  ├── Gin (HTTP router)   ├── PostgreSQL 15+                │
│  ├── GORM (ORM)          └── Separate schema (shared DB)   │
│  └── Zap (logging)                                          │
│                                                              │
│  RBAC System:           APIs:                               │
│  ├── JWT verification    ├── /admin/customers               │
│  ├── HS256 (shared)      ├── /admin/contacts                │
│  ├── Roles: admin/manager/agent ├── /admin/deals          │
│  └── CORS configured     ├── /admin/activities              │
│                          ├── /admin/tags                    │
│                          ├── /admin/reports/overview        │
│                          └── /admin/me (current user)       │
│                                                              │
│  Observability:         Deployment:                         │
│  ├── Structured logs (Zap) ├── Docker + docker-compose     │
│  ├── Prometheus metrics  └── JWT_SECRET must match CMS      │
│  └── Request ID tracking                                     │
└─────────────────────────────────────────────────────────────┘
```

#### 6. Shared Infrastructure
```
┌─────────────────────────────────────────────────────────────┐
│  SHARED INFRASTRUCTURE                                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Data Layer:             Caching:                           │
│  ├── PostgreSQL 15+      ├── Redis (queue backend)          │
│  │   ├── pgvector (384-dim)│ └── Optional: API caching     │
│  │   ├── pgcrypto (UUID)                                 │
│  │   └── Connection pooling                                │
│  └── Supabase Hosting (recommended)                        │
│                                                              │
│  Storage Layer:          CDN:                               │
│  ├── Supabase Storage    ├── Deliver media artifacts       │
│  │   or S3-compatible    ├── Global edge caching            │
│  │   ├── MP4 files       └── Cached on first request       │
│  │   ├── Thumbnails                                         │
│  │   └── Original media                                     │
│                                                              │
│  External Services:     Development:                        │
│  ├── YouTube Data API v3 ├── Docker Compose (local stack)  │
│  ├── X/Twitter API       ├── npm/yarn workspaces (if used) │
│  ├── Reddit API          └── Git monorepo structure        │
│  ├── iTunes/Search API                                      │
│  └── RSS feeds (public)                                      │
└─────────────────────────────────────────────────────────────┘
```

### Database Schema (PostgreSQL + pgvector)
```
┌─────────────────────────────────────────────────────────────┐
│  POSTGRESQL DATABASE (wahb_platform)                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Platform Tables (CMS-managed):                              │
│  ├── content_items (polymorphic: ARTICLE|VIDEO|TWEET|COMMENT|PODCAST)
│  │   └── embedding: vector(384)                            │
│  ├── transcripts (full_text, summary, word_timestamps)     │
│  ├── user_interactions (like/bookmark/share/view/complete) │
│  ├── content_sources (RSS, YouTube, podcast configs)       │
│  ├── pages (CMS pages)                                      │
│  ├── posts (CMS posts)                                      │
│  ├── media (CMS media)                                      │
│  └── post_media (junction table)                            │
│                                                              │
│  CRM Tables (CRM-managed):                                  │
│  ├── customers (soft delete, assigned_to)                  │
│  ├── contacts (primary designation, nested under customers)│
│  ├── deals (pipeline stages, amounts, probabilities)       │
│  ├── activities (tasks, meetings, calls, emails)           │
│  ├── notes (attached to customers/deals)                   │
│  ├── tags (many-to-many with customers)                    │
│  └── audit_logs (immutable change tracking)                │
│                                                              │
│  Extensions:                                                 │
│  ├── pgcrypto (generate UUID)                              │
│  └── vector (pgvector for similarity search)               │
└─────────────────────────────────────────────────────────────┘
```

### External APIs & Sources Integration Map
```
┌─────────────────────────────────────────────────────────────┐
│  EXTERNAL INTEGRATIONS (Aggregation Service ONLY)           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Content Sources (Ingest):                                  │
│  ├── YouTube Data API v3                                    │
│  │   └── Endpoints: search, videos, channels, playlists     │
│  ├── X/Twitter API v2                                       │
│  │   └── Endpoints: timelines, search, users               │
│  ├── Reddit API                                             │
│  │   └── Endpoints: subreddit, posts, comments             │
│  ├── iTunes Search / Podcast RSS                            │
│  │   └── Feeds: podcast index, episode lists               │
│  ├── RSS Feeds (public)                                     │
│  │   └── Parser: rss-parser (npm)                          │
│  └── Manual Uploads (via Console → CMS)                    │
│                                                              │
│  Processing Tools:                                           │
│  ├── FFmpeg (media transcode to MP4)                        │
│  ├── yt-dlp (YouTube/media downloader)                     │
│  ├── Whisper AI (audio transcription)                       │
│  └── all-MiniLM-L6-v2 (384-dim embeddings)                 │
│                                                              │
│  Storage APIs:                                              │
│  ├── Supabase Storage API                                   │
│  └── AWS S3 API (if using S3-compatible)                   │
└─────────────────────────────────────────────────────────────┘
```

### API Contract Matrix
```
┌─────────────────────────────────────────────────────────────┐
│  API ENDPOINT MATRIX                                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Wahb-Platform → CMS (Public APIs):                         │
│  ├── GET /api/v1/feed/foryou (cursor pag, MP4 URLs)        │
│  ├── GET /api/v1/feed/news (slides: 1 featured + 3 related)│
│  ├── GET /api/v1/content/:id                               │
│  ├── POST /api/v1/interactions                              │
│  ├── GET /api/v1/interactions/bookmarks                     │
│  └── DELETE /api/v1/interactions/:id                        │
│                                                              │
│  Platform-Console → CMS (Admin APIs):                       │
│  ├── POST /admin/login (issue JWT)                          │
│  ├── GET/POST/PUT/DELETE /admin/content_sources            │
│  ├── GET /admin/content_items (with filters)                │
│  └── POST /admin/content_sources/:id/trigger (ingestion)    │
│                                                              │
│  Platform-Console → CRM (Admin APIs):                       │
│  ├── GET /admin/me (current user from JWT)                  │
│  ├── GET/POST/PATCH /admin/customers                        │
│  ├── GET/POST/PUT/DELETE /admin/contacts                    │
│  ├── GET/POST/PATCH /admin/deals                            │
│  ├── GET/POST/PATCH /admin/activities                       │
│  ├── GET /admin/me/activities (my tasks)                    │
│  ├── GET/POST/PUT/DELETE /admin/tags                        │
│  └── GET /admin/reports/overview                            │
│                                                              │
│  Aggregation → CMS (Service-to-Service):                    │
│  ├── POST /admin/content_items (upsert by idempotency_key) │
│  ├── PATCH /admin/content_items/:id (status, artifacts)    │
│  └── POST /admin/transcripts (create + link)               │
└─────────────────────────────────────────────────────────────┘
```

### Environment Variables (Per Service)
```
┌─────────────────────────────────────────────────────────────┐
│  ENVIRONMENT VARIABLES CONFIGURATION                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Wahb-Platform:                                             │
│  ├── NEXT_PUBLIC_API_URL                                    │
│  └── NEXT_PUBLIC_USE_MOCK_DATA                              │
│                                                              │
│  CMS:                                                       │
│  ├── DATABASE_URL (PostgreSQL connection)                  │
│  ├── ENV (development|dev|production)                       │
│  ├── JWT_SECRET (shared with CRM)                          │
│  └── JWT_ISSUER: "wahb-cms"                                 │
│                                                              │
│  Aggregation Service:                                       │
│  ├── CMS_BASE_URL (internal API base)                      │
│  ├── CMS_SERVICE_TOKEN (service auth)                      │
│  ├── REDIS_URL (queue backend)                              │
│  ├── STORAGE_BASE_URL / STORAGE_BUCKET                     │
│  ├── YOUTUBE_API_KEY                                        │
│  ├── TWITTER_API_KEY / TWITTER_API_SECRET                  │
│  ├── REDDIT_CLIENT_ID / REDDIT_SECRET                      │
│  └── SOURCE_ALLOWLIST_PATH (scraping domains)              │
│                                                              │
│  Platform-Console:                                          │
│  ├── NEXT_PUBLIC_CMS_BASE_URL                               │
│  ├── NEXT_PUBLIC_CRM_BASE_URL                               │
│  └── NEXT_PUBLIC_APP_ENV                                    │
│                                                              │
│  CRM Service:                                               │
│  ├── DATABASE_URL (PostgreSQL, same as CMS)                │
│  ├── JWT_SECRET (MUST match CMS)                            │
│  ├── JWT_ISSUER: "wahb-cms" (validation only)              │
│  └── CORS_ALLOWED_ORIGINS (Vercel + localhost)             │
└─────────────────────────────────────────────────────────────┘
```

### Deployment Topology
```
┌─────────────────────────────────────────────────────────────┐
│  DEPLOYMENT & INFRASTRUCTURE                                 │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Production:                                                 │
│  ├── Wahb-Platform → Vercel (Edge network)                  │
│  ├── Platform-Console → Vercel (Edge network)               │
│  ├── CMS → Docker container (Cloud VM/K8s)                  │
│  ├── Aggregation → Docker containers (Cloud VM/K8s)         │
│  ├── CRM → Docker container (Cloud VM/K8s)                  │
│  ├── PostgreSQL → Supabase or managed Postgres             │
│  ├── Redis → Managed Redis (ElastiCache/Upstash)           │
│  ├── Storage → Supabase Storage or S3                       │
│  └── CDN → Supabase CDN or CloudFront                       │
│                                                              │
│  Development:                                                │
│  ├── All services → Docker Compose (local)                 │
│  ├── PostgreSQL → Docker container (local)                 │
│  ├── Redis → Docker container (local)                      │
│  └── Storage → MinIO (local S3-compatible)                 │
└─────────────────────────────────────────────────────────────┘
```

### Visual Style Requirements
1. Organize into 6 distinct sections (one per component)
2. Use color-coding for service types:
   - Blue: Frontend (Next.js apps)
   - Green: Backend services (Go)
   - Orange: Workers (Node.js)
   - Purple: Storage/Infrastructure
   - Yellow: External APIs
3. Show all technologies as tags/badges
4. Include API endpoints as connection arrows
5. Add environment variables as small config boxes
6. Show deployment platforms (Vercel, Docker, Supabase)
7. Include a legend for technology categories
8. Group external integrations separately
9. Show shared infrastructure (PostgreSQL, Redis, Storage) at bottom
10. Add data flow indicators between components

Generate a comprehensive technology stack diagram showing all 6 components, their tech stacks, external integrations, API contracts, deployment topology, and infrastructure layer.
```
