# Aggregation Service - Complete Guide

**Last Updated:** February 6, 2026

This guide explains how the Aggregation Service integrates with the Wahb platform, its workflow, and interactions with other services.

---

## Table of Contents

1. [Overview](#overview)
2. [Service Role & Boundaries](#service-role--boundaries)
3. [System Architecture & Interactions](#system-architecture--interactions)
4. [Pipeline Workflows](#pipeline-workflows)
5. [Queue Management](#queue-management)
6. [Service Integrations](#service-integrations)
7. [Data Flow & Artifacts](#data-flow--artifacts)
8. [API Contracts](#api-contracts)
9. [Configuration](#configuration)
10. [Operations & Monitoring](#operations--monitoring)
11. [Common Scenarios](#common-scenarios)
12. [Troubleshooting](#troubleshooting)

---

## Overview

The **Aggregation Service** is a Node.js-based worker fleet responsible for ingesting content from external sources, processing media, generating transcripts and embeddings, and writing results to the CMS database.

**Key Characteristics:**
- Asynchronous, queue-based processing
- No user-facing APIs (service-to-service only)
- Handles all media processing, scraping, and AI enrichment
- Produces MP4-ready content for the For You feed

---

## Service Role & Boundaries

### What Aggregation Service Does

✅ **Ingest content** from external sources (RSS, YouTube, Podcasts, Social, Manual)
✅ **Process media** with FFmpeg (download, transcode, MP4 conversion)
✅ **Generate transcripts** using Whisper
✅ **Create embeddings** (384-dimension vectors) for semantic search
✅ **Upload artifacts** to object storage (S3/Supabase Storage)
✅ **Write to CMS** via internal APIs (no direct DB access)

### What Aggregation Service Does NOT Do

❌ **Serve user-facing APIs** (only internal endpoints like /health)
❌ **Assemble feeds** (CMS Feed Service handles this)
❌ **Execute pgvector queries** (CMS performs similarity search)
❌ **Provide admin UI** (Platform Console is the admin interface)
❌ **Handle customer/deal workflows** (CRM Service owns this)

---

## System Architecture & Interactions

### High-Level Architecture

```
┌─────────────────┐      ┌──────────────────┐      ┌─────────────────┐
│  External       │      │  Aggregation     │      │  CMS / Feed     │
│  Sources        │─────▶│  Service         │─────▶│  Service        │
│  - RSS          │      │  (Node.js)       │      │  (Go)           │
│  - YouTube      │      │                  │      │                 │
│  - Podcasts     │      │  ┌──────────────┐│      │  ┌──────────────┐│
│  - Social       │      │  │  Queue       ││      │  │  PostgreSQL  ││
│  - Manual       │      │  │  Workers     ││      │  │  + pgvector  ││
└─────────────────┘      │  │  (BullMQ)    ││      │  └──────────────┘│
                         │  └──────────────┘│      └─────────────────┘
                         └──────────────────┘                │
                                 │                          │
                                 ▼                          ▼
                         ┌──────────────────┐      ┌─────────────────┐
                         │  Object Storage  │      │  Redis          │
                         │  (S3/Supabase)   │      │  (Cache/Queue)  │
                         └──────────────────┘      └─────────────────┘
                                 ▲                          ▲
                                 └──────────────────────────┘

┌─────────────────┐                    ┌─────────────────┐
│  Platform       │                    │  Wahb Platform  │
│  Console        │◀───────────────────▶│  (Next.js)      │
│  (Admin UI)     │   Feed APIs        │  (Consumer UI)  │
└─────────────────┘                    └─────────────────┘
```

### Interaction Flow

#### 1. Ingestion Trigger (Platform Console → Aggregation)

**Flow:**
```
Admin User → Platform Console → CMS Admin API → Queue Job → Aggregation Worker
```

**Steps:**
1. Admin creates or triggers a content source in Platform Console
2. Console calls CMS admin endpoint: `POST /admin/content-sources` or `POST /admin/jobs/trigger`
3. CMS validates and creates a job in Redis queue (BullMQ)
4. Aggregation worker picks up the job

#### 2. Content Processing (Aggregation → CMS → Storage)

**Flow:**
```
External Source → Aggregation Worker → Object Storage → CMS Database
```

**Steps:**
1. Worker fetches content from external source
2. Worker processes media (FFmpeg) and generates artifacts
3. Worker uploads artifacts to Object Storage (S3/Supabase)
4. Worker writes content metadata to CMS via internal API
5. CMS stores record in `content_items` table

#### 3. Feed Delivery (CMS → Wahb Platform)

**Flow:**
```
User Request → Wahb Platform → CMS Feed API → pgvector Query → Feed Response
```

**Steps:**
1. User opens Wahb Platform app
2. App requests feed: `GET /api/v1/feed/foryou` or `/api/v1/feed/news`
3. CMS queries `content_items` table with pgvector similarity
4. CMS assembles feed (MP4 URLs for For You, slides for News)
5. CMS returns feed to app

---

## Pipeline Workflows

### General Pipeline Stages

All content flows through these stages:

```
Fetch → Normalize → Media → Transcript → Embeddings → Status: READY
```

| Stage | Purpose | Output |
|-------|---------|--------|
| **Fetch** | Acquire raw data from source | Raw content, metadata |
| **Normalize** | Map to ContentItem schema | Structured data, idempotency key |
| **Media** | Download, transcode, upload | MP4 URL, thumbnail URL, duration |
| **Transcript** | Generate text from audio/video | Transcript record |
| **Embeddings** | Create vector for search | 384-dimension vector |

### 1. RSS / Web Article Pipeline

**Source Types:** RSS feeds, News sites

**Flow:**
```
1. Poll RSS feed for new items
2. Parse item (title, link, published_at)
3. If excerpt-only, scrape full article from allowlisted domain
4. Clean HTML to text/markdown
5. Normalize to ContentItem (type: ARTICLE)
6. Set status: READY
```

**Artifacts:**
- `content_items` record (type: ARTICLE)
- No media required
- Optional thumbnail

**Example Request:**
```bash
POST /admin/content-sources
{
  "name": "TechCrunch RSS",
  "type": "RSS",
  "feed_url": "https://techcrunch.com/feed/",
  "fetch_interval_minutes": 60,
  "metadata": {
    "scrape_full_content": true,
    "allowlisted_domain": "techcrunch.com"
  }
}
```

### 2. Video / Podcast Pipeline (YouTube)

**Source Types:** YouTube channels, playlists, Podcast RSS

**Flow:**
```
1. Poll channel/playlist for new media
2. Download with yt-dlp (metadata + media streams)
3. Transcode with FFmpeg:
   - Input: YouTube video/audio
   - Output: MP4 (H.264 video or AAC audio)
4. Generate transcript using Whisper
5. Upload artifacts to Object Storage:
   - MP4 file
   - Thumbnail image
6. Write to CMS:
   - ContentItem (type: VIDEO or PODCAST)
   - Transcript record
   - Generate embeddings from transcript + title
7. Set status: READY
```

**Artifacts:**
- `content_items` record (type: VIDEO or PODCAST)
- `media_url`: MP4 file URL (required for For You feed)
- `thumbnail_url`: Image URL
- `transcripts` record
- Embedding vector (384 dims)

**Example Request:**
```bash
POST /admin/content-sources
{
  "name": "TED Talks",
  "type": "YOUTUBE",
  "feed_url": "https://www.youtube.com/@TEDtalks",
  "fetch_interval_minutes": 1440,
  "api_config": {
    "channel_id": "UCsT0YrN5RMq-3WpJK7EeRNg"
  }
}
```

### 3. Social Pipeline (X/Twitter, Reddit)

**Source Types:** X/Twitter accounts, Reddit subreddits

**Flow:**
```
1. Fetch recent posts via API or approved scraping
2. Filter by engagement (likes, retweets, upvotes)
3. If reply/comment, capture parent context
4. Normalize to ContentItem (type: TWEET or COMMENT)
5. Generate embeddings from text
6. Set status: READY
```

**Artifacts:**
- `content_items` record (type: TWEET or COMMENT)
- No media required
- Embedding vector (384 dims)

**Example Request:**
```bash
POST /admin/content-sources
{
  "name": "Tech Twitter",
  "type": "TWITTER",
  "feed_url": "https://twitter.com/elonmusk",
  "fetch_interval_minutes": 30,
  "api_config": {
    "username": "elonmusk",
    "min_engagement_score": 100
  }
}
```

### 4. Manual Upload Pipeline

**Source Type:** Manual upload from Console

**Flow:**
```
1. Admin uploads file via Platform Console
2. Console streams file to Object Storage
3. Console calls CMS to create ContentItem with status: PENDING
4. CMS queues job for Aggregation Service
5. Aggregation worker:
   - Downloads file from Object Storage
   - Transcodes to MP4 if needed
   - Generates transcript
   - Creates embeddings
   - Updates ContentItem with artifacts
   - Sets status: READY
```

**Artifacts:**
- `content_items` record (type: VIDEO or PODCAST)
- `media_url`: MP4 file URL
- `thumbnail_url`: Auto-generated
- `transcripts` record
- Embedding vector (384 dims)

**Example Request:**
```bash
POST /admin/content/manual-upload
{
  "title": "My Podcast Episode",
  "type": "PODCAST",
  "file_url": "s3://wahb-uploads/uploads/audio.mp3",
  "metadata": {
    "author": "John Doe",
    "duration_sec": 1800
  }
}
```

---

## Queue Management

### BullMQ Architecture

The Aggregation Service uses BullMQ with Redis as the queue backend.

**Queue Names:**
- `fetch-queue` - Fetch raw data from sources
- `normalize-queue` - Normalize to ContentItem schema
- `media-queue` - Process media (FFmpeg, upload)
- `transcript-queue` - Generate transcripts
- `embeddings-queue` - Create embeddings

### Job Lifecycle

```
Create Job → Queued → Processing → Completed / Failed
                                  ↓
                            Retry (up to 3x)
```

### Job Data Structure

```javascript
{
  id: "uuid",
  idempotency_key: "canonical-url-or-hash",
  source_type: "RSS|YOUTUBE|PODCAST|TWITTER|REDDIT|MANUAL",
  source_config: { ... },
  content_item_id: "uuid",
  stage: "fetch|normalize|media|transcript|embeddings",
  retry_count: 0,
  max_retries: 3,
  created_at: timestamp,
  updated_at: timestamp
}
```

### Retry Strategy

- **Transient failures:** Network errors, rate limits → Auto-retry with exponential backoff
- **Permanent failures:** Invalid URL, API auth errors → Mark as FAILED, log error
- **Failed jobs:** Moved to dead-letter queue (DLQ) for manual review

---

## Service Integrations

### 1. Aggregation ↔ CMS Integration

**Aggregation calls CMS** to write data:

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/internal/content/upsert` | POST | Create/update ContentItem by idempotency key |
| `/internal/content/status` | PATCH | Update status (PENDING → PROCESSING → READY/FAILED) |
| `/internal/content/artifacts` | PATCH | Attach media_url, thumbnail_url, duration_sec |
| `/internal/transcripts/create` | POST | Create transcript record |

**Authentication:**
- Aggregation uses `CMS_SERVICE_TOKEN` (service-to-service)
- Token passed via `Authorization: Bearer <token>` header

**Example: Upsert ContentItem**
```bash
POST /internal/content/upsert
Authorization: Bearer <CMS_SERVICE_TOKEN>

{
  "idempotency_key": "https://youtube.com/watch?v=abc123",
  "type": "VIDEO",
  "title": "AI Revolution",
  "body_text": "Full transcript here...",
  "excerpt": "AI is changing everything...",
  "author": "TED Talks",
  "source_name": "YouTube",
  "source_feed_url": "https://youtube.com/@TEDtalks",
  "status": "READY",
  "media_url": "https://s3.../video.mp4",
  "thumbnail_url": "https://s3.../thumb.jpg",
  "duration_sec": 600,
  "topic_tags": ["AI", "Technology"],
  "embedding": [0.123, -0.456, ...], // 384 dims
  "metadata": {
    "youtube_video_id": "abc123",
    "original_url": "https://youtube.com/watch?v=abc123"
  }
}
```

### 2. Platform Console ↔ CMS ↔ Aggregation

**Console triggers Aggregation** via CMS:

1. Admin creates content source in Console
2. Console calls: `POST /admin/content-sources`
3. CMS stores source config in `content_sources` table
4. CMS triggers job: `POST /internal/jobs/trigger`
5. Aggregation picks up job and starts processing

**Example: Trigger Ingestion**
```bash
POST /admin/jobs/trigger
Authorization: Bearer <ADMIN_JWT>

{
  "source_id": "uuid-of-content-source",
  "job_type": "FETCH_NEW_ITEMS",
  "metadata": {
    "force_refresh": false
  }
}
```

### 3. CMS ↔ Wahb Platform

**Wahb Platform consumes feeds** from CMS:

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/v1/feed/foryou` | GET | For You feed (MP4 cards) |
| `/api/v1/feed/news` | GET | News feed (slides) |
| `/api/v1/content/:id` | GET | Single content item details |

**Example: For You Feed Request**
```bash
GET /api/v1/feed/foryou?limit=20&cursor=base64(timestamp:uuid)

Response:
{
  "items": [
    {
      "id": "uuid-1",
      "type": "VIDEO",
      "url": "https://s3.../video.mp4",
      "thumbnail_url": "https://s3.../thumb.jpg",
      "title": "AI Revolution",
      "duration": 600,
      "author": "TED Talks",
      "like_count": 1234,
      "view_count": 5678,
      "interactions": {
        "liked": false,
        "bookmarked": false
      }
    }
  ],
  "cursor": "base64(next_timestamp:next_uuid)",
  "has_more": true
}
```

---

## Data Flow & Artifacts

### Content Lifecycle

```
PENDING → PROCESSING → READY → (optionally) ARCHIVED
```

### Status Transitions

| Status | Trigger | Next Status |
|--------|---------|-------------|
| PENDING | Content created (manual or source trigger) | PROCESSING |
| PROCESSING | Job starts (any stage) | READY or FAILED |
| READY | All required artifacts complete | ARCHIVED (if outdated) |
| FAILED | Error in any stage (max retries exceeded) | — |

### Database Tables

#### content_items

```sql
CREATE TABLE content_items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  type VARCHAR(20) NOT NULL, -- ARTICLE, VIDEO, TWEET, COMMENT, PODCAST
  source VARCHAR(20) NOT NULL, -- RSS, PODCAST, YOUTUBE, UPLOAD, MANUAL
  status VARCHAR(20) NOT NULL, -- PENDING, PROCESSING, READY, FAILED, ARCHIVED

  -- Content fields
  title TEXT,
  body_text TEXT,
  excerpt TEXT,

  -- Media fields
  media_url TEXT,
  thumbnail_url TEXT,
  original_url TEXT,
  duration_sec INTEGER,

  -- Attribution
  author TEXT,
  source_name TEXT,
  source_feed_url TEXT,

  -- AI fields
  topic_tags TEXT[],
  embedding vector(384),
  metadata JSONB,

  -- Engagement
  like_count INTEGER DEFAULT 0,
  comment_count INTEGER DEFAULT 0,
  share_count INTEGER DEFAULT 0,
  view_count INTEGER DEFAULT 0,

  -- Timestamps
  published_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),

  -- Deduplication
  idempotency_key TEXT UNIQUE
);
```

#### transcripts

```sql
CREATE TABLE transcripts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  content_item_id UUID REFERENCES content_items(id),
  full_text TEXT NOT NULL,
  summary TEXT,
  word_timestamps JSONB,
  language VARCHAR(10),
  created_at TIMESTAMP DEFAULT NOW()
);
```

#### content_sources

```sql
CREATE TABLE content_sources (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(255) NOT NULL,
  type VARCHAR(20) NOT NULL, -- RSS, PODCAST, YOUTUBE, TWITTER, REDDIT
  feed_url TEXT,
  api_config JSONB,
  is_active BOOLEAN DEFAULT true,
  fetch_interval_minutes INTEGER,
  last_fetched_at TIMESTAMP,
  metadata JSONB,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
```

### Object Storage Artifacts

```
wahb-uploads/
├── media/
│   ├── videos/
│   │   ├── {content_item_id}.mp4
│   │   └── {content_item_id}_thumb.jpg
│   └── audio/
│       ├── {content_item_id}.mp4
│       └── {content_item_id}_thumb.jpg
├── transcripts/
│   └── {content_item_id}.txt
└── uploads/
    └── {user_id}/{filename}
```

---

## API Contracts

### CMS Internal APIs (Aggregation → CMS)

**Base URL:** `CMS_BASE_URL` (internal)

**Authentication:** `Authorization: Bearer <CMS_SERVICE_TOKEN>`

#### 1. Upsert ContentItem

```http
POST /internal/content/upsert
Content-Type: application/json

{
  "idempotency_key": "required-string-or-hash",
  "type": "ARTICLE|VIDEO|TWEET|COMMENT|PODCAST",
  "source": "RSS|PODCAST|YOUTUBE|UPLOAD|MANUAL",
  "status": "PENDING|PROCESSING|READY|FAILED",
  "title": "string",
  "body_text": "string",
  "excerpt": "string",
  "author": "string",
  "source_name": "string",
  "source_feed_url": "string",
  "media_url": "string",
  "thumbnail_url": "string",
  "original_url": "string",
  "duration_sec": integer,
  "topic_tags": ["string"],
  "embedding": [number, number, ...], // 384 dims
  "metadata": {},
  "published_at": "ISO8601"
}

Response:
{
  "id": "uuid",
  "created": boolean,
  "status": "PENDING|PROCESSING|READY|FAILED"
}
```

#### 2. Update Status

```http
PATCH /internal/content/{id}/status
Content-Type: application/json

{
  "status": "PENDING|PROCESSING|READY|FAILED"
}

Response:
{
  "id": "uuid",
  "status": "PENDING|PROCESSING|READY|FAILED"
}
```

#### 3. Attach Artifacts

```http
PATCH /internal/content/{id}/artifacts
Content-Type: application/json

{
  "media_url": "string",
  "thumbnail_url": "string",
  "duration_sec": integer
}

Response:
{
  "id": "uuid",
  "media_url": "string",
  "thumbnail_url": "string",
  "duration_sec": integer
}
```

#### 4. Create Transcript

```http
POST /internal/transcripts
Content-Type: application/json

{
  "content_item_id": "uuid",
  "full_text": "string",
  "summary": "string",
  "word_timestamps": {},
  "language": "en"
}

Response:
{
  "id": "uuid",
  "content_item_id": "uuid"
}
```

### CMS Admin APIs (Console → CMS)

**Base URL:** `CMS_BASE_URL` (public)

**Authentication:** `Authorization: Bearer <ADMIN_JWT>`

#### 1. Create Content Source

```http
POST /admin/content-sources
Content-Type: application/json

{
  "name": "string",
  "type": "RSS|PODCAST|YOUTUBE|TWITTER|REDDIT",
  "feed_url": "string",
  "api_config": {},
  "is_active": true,
  "fetch_interval_minutes": 60,
  "metadata": {}
}

Response:
{
  "id": "uuid",
  "name": "string",
  "status": "active"
}
```

#### 2. Trigger Ingestion Job

```http
POST /admin/jobs/trigger
Content-Type: application/json

{
  "source_id": "uuid",
  "job_type": "FETCH_NEW_ITEMS|REFRESH_ALL",
  "metadata": {}
}

Response:
{
  "job_id": "uuid",
  "status": "queued",
  "estimated_completion": "ISO8601"
}
```

#### 3. List Content Items

```http
GET /admin/content-items?type=VIDEO&status=READY&limit=50

Response:
{
  "items": [...],
  "total": 123,
  "page": 1,
  "limit": 50
}
```

---

## Configuration

### Environment Variables (Aggregation Service)

```bash
# Required
CMS_BASE_URL=http://cms:8080
CMS_SERVICE_TOKEN=secret-service-token
REDIS_URL=redis://redis:6379
STORAGE_BASE_URL=https://s3.amazonaws.com/wahb-uploads
STORAGE_BUCKET=wahb-uploads

# Optional (for AWS S3)
AWS_ACCESS_KEY_ID=xxx
AWS_SECRET_ACCESS_KEY=xxx
AWS_REGION=us-east-1

# Optional (for Supabase Storage)
SUPABASE_URL=https://xxx.supabase.co
SUPABASE_KEY=xxx

# Optional (Worker tuning)
WORKER_CONCURRENCY=5
QUEUE_NAMES=fetch-queue,normalize-queue,media-queue,transcript-queue,embeddings-queue

# Optional (Scraping)
SOURCE_ALLOWLIST_PATH=/config/allowlist.json
USER_AGENT=Wahb-Aggregator/1.0
```

### Scraping Allowlist

```json
{
  "allowed_domains": [
    "techcrunch.com",
    "theverge.com",
    "arstechnica.com"
  ],
  "user_agent": "Wahb-Aggregator/1.0 (+https://wahb.com/bot)",
  "request_timeout_ms": 10000,
  "max_retries": 3
}
```

---

## Operations & Monitoring

### Health Check

```http
GET /health

Response:
{
  "status": "ok",
  "queues": {
    "fetch-queue": { "waiting": 0, "active": 2, "completed": 1234, "failed": 5 },
    "normalize-queue": { "waiting": 5, "active": 3, "completed": 1200, "failed": 2 },
    "media-queue": { "waiting": 10, "active": 5, "completed": 1150, "failed": 3 },
    "transcript-queue": { "waiting": 8, "active": 4, "completed": 1100, "failed": 4 },
    "embeddings-queue": { "waiting": 6, "active": 2, "completed": 1080, "failed": 1 }
  },
  "database": { "status": "connected" },
  "redis": { "status": "connected" },
  "storage": { "status": "connected" }
}
```

### Metrics

- **Job throughput:** Jobs completed per minute
- **Queue depth:** Number of jobs waiting per queue
- **Failure rate:** Percentage of failed jobs
- **Average processing time:** Mean time per job by stage
- **Storage usage:** Total size of uploaded artifacts

### Logging

**Log Format:** JSON structured logging

**Key Fields:**
- `job_id`
- `content_item_id`
- `source_type`
- `stage`
- `status`
- `error` (if failed)
- `duration_ms`

**Example Log:**
```json
{
  "timestamp": "2026-02-06T10:30:00Z",
  "level": "info",
  "job_id": "job-uuid",
  "content_item_id": "content-uuid",
  "source_type": "YOUTUBE",
  "stage": "media",
  "status": "completed",
  "duration_ms": 5000,
  "message": "Successfully transcoded video to MP4"
}
```

---

## Common Scenarios

### Scenario 1: Adding a New RSS Feed

1. **Admin creates source in Console:**
   ```bash
   POST /admin/content-sources
   {
     "name": "News Site RSS",
     "type": "RSS",
     "flow_url": "https://example.com/feed/",
     "fetch_interval_minutes": 60,
     "metadata": { "scrape_full_content": true }
   }
   ```

2. **CMS queues fetch job**

3. **Aggregation worker:**
   - Fetches RSS feed
   - Parses items
   - Scrapes full articles (if allowlisted)
   - Upserts content items to CMS

4. **CMS stores articles** with status: READY

5. **Articles appear in News feed**

### Scenario 2: Processing a YouTube Video

1. **Admin creates YouTube source:**
   ```bash
   POST /admin/content-sources
   {
     "name": "Tech YouTube",
     "type": "YOUTUBE",
     "feed_url": "https://youtube.com/@channel",
     "fetch_interval_minutes": 1440
   }
   ```

2. **Aggregation worker:**
   - Polls YouTube API for new videos
   - Downloads video with yt-dlp
   - Transcodes to MP4 with FFmpeg
   - Generates transcript with Whisper
   - Uploads MP4, thumbnail, transcript to Object Storage
   - Creates embeddings from transcript + title
   - Upserts content item to CMS

3. **CMS stores video** with status: READY

4. **Video appears in For You feed**

### Scenario 3: Manual Upload via Console

1. **Admin uploads file in Console:**
   - Selects file (MP3, MP4, etc.)
   - Enters title, author, etc.
   - Clicks "Upload"

2. **Console streams file to Object Storage**

3. **Console creates PENDING content item:**
   ```bash
   POST /admin/content/manual-upload
   {
     "title": "My Podcast",
     "type": "PODCAST",
     "file_url": "s3://wahb-uploads/uploads/xxx.mp3"
   }
   ```

4. **CMS queues media processing job**

5. **Aggregation worker:**
   - Downloads file from Object Storage
   - Transcodes to MP4
   - Generates transcript
   - Creates embeddings
   - Updates content item with artifacts
   - Sets status: READY

6. **Content appears in For You feed**

### Scenario 4: Failed Job Recovery

1. **Job fails** (e.g., rate limit from YouTube API)

2. **Job moves to DLQ** after max retries

3. **Admin checks failed jobs in Console:**
   ```bash
   GET /admin/jobs/failed?limit=50
   ```

4. **Admin reviews error logs**

5. **Admin retries job:**
   ```bash
   POST /admin/jobs/{job_id}/retry
   ```

6. **Job re-enters queue** and processes

---

## Troubleshooting

### Common Issues & Solutions

#### Issue 1: Jobs stuck in PROCESSING status

**Symptoms:**
- Content items stuck at status: PROCESSING
- Queue showing active jobs not completing

**Possible Causes:**
- Worker crashed
- FFmpeg hung on large file
- Network timeout to storage

**Solutions:**
1. Check worker logs for errors
2. Restart worker pods
3. Check storage connectivity
4. Kill stuck jobs and retry

#### Issue 2: MP4 files not playable in app

**Symptoms:**
- For You feed returns MP4 URLs
- Videos won't play or show errors

**Possible Causes:**
- Incorrect codec (not H.264 / AAC)
- Container format issues
- File corrupted during upload

**Solutions:**
1. Verify FFmpeg output: `ffprobe file.mp4`
2. Check codec: Video must be H.264, Audio must be AAC
3. Re-transcode with correct codecs:
   ```bash
   ffmpeg -i input.mp4 -c:v libx264 -c:a aac output.mp4
   ```

#### Issue 3: Embeddings not generated

**Symptoms:**
- Content items have status: READY
- pgvector search returns no results
- `embedding` column is NULL

**Possible Causes:**
- Embeddings queue not running
- Embedding model not loaded
- Text too long for model

**Solutions:**
1. Check embeddings queue: `GET /health` → `queues.embeddings-queue`
2. Restart embeddings worker
3. Verify model is installed and accessible
4. Check embeddings worker logs

#### Issue 4: Transcript generation failing

**Symptoms:**
- Video/Podcast items stuck at status: PROCESSING
- Transcript queue showing failures
- Error: "Whisper model not found"

**Possible Causes:**
- Whisper model not downloaded
- Insufficient GPU/CPU resources
- Audio format not supported

**Solutions:**
1. Download Whisper model: `whisper download base`
2. Check system resources (RAM, GPU if available)
3. Verify audio format: `ffprobe file.mp4`
4. Use supported audio codec: AAC

#### Issue 5: Content not appearing in feeds

**Symptoms:**
- Content items show status: READY
- Not appearing in For You or News feed

**Possible Causes:**
- Wrong content type (e.g., ARTICLE in For You feed)
- Embedding not generated (for similarity search)
- Pagination cursor issues

**Solutions:**
1. Verify content type matches feed:
   - For You: VIDEO or PODCAST
   - News: ARTICLE (featured), TWEET or COMMENT (related)
2. Check embedding is not NULL
3. Test feed directly: `GET /api/v1/feed/foryou?limit=5`
4. Clear pagination cache if needed

---

## Quick Reference

### Service Boundaries

| Service | CAN | CANNOT |
|---------|-----|--------|
| **Aggregation** | Scrape, FFmpeg, Whisper, Embeddings | Serve user traffic, Assemble feeds |
| **CMS** | Serve feeds, pgvector search | Scrape, FFmpeg |
| **Console** | Admin UI, trigger jobs | Direct queue access |
| **Platform** | Display feeds, User interactions | Media processing |

### Key Endpoints

| Service | Endpoint | Purpose |
|---------|----------|---------|
| **CMS** | `POST /internal/content/upsert` | Create/update content |
| **CMS** | `GET /api/v1/feed/foryou` | For You feed |
| **CMS** | `GET /api/v1/feed/news` | News feed |
| **Console** | `POST /admin/content-sources` | Create source |
| **Console** | `POST /admin/jobs/trigger` | Trigger ingestion |

### Status Flow

```
PENDING → PROCESSING → READY → (optionally) ARCHIVED
                     ↓
                  FAILED (→ retry)
```

### Queue Names

- `fetch-queue`
- `normalize-queue`
- `media-queue`
- `transcript-queue`
- `embeddings-queue`

---

## Additional Resources

- **Overall Project Context:** `context/Wahb_Overall_Project_Context_Requirements.md`
- **Aggregation Service Context:** `Aggregation-Service/context/Aggregation_Service_Context_Requirements.md`
- **CMS Context:** `Content-Management-System/context/CMS_Context_Requirements.md`
- **Platform Console Context:** `Platform-Console/context/Platform_Console_Context_Requirements.md`

---

**Document Version:** 1.0
**Last Updated:** February 6, 2026
**Maintained By:** Wahb Engineering Team
