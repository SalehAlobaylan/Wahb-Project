# Claude AI Context - Wahb Project

**Last Updated:** February 3, 2026

---

## Quick Overview

**Wahb** is a mono-repo with 5 services + 1 frontend app:
- **Wahb-Platform** (Next.js) - Consumer UI for For You + News feeds
- **Content-Management-System** (Go) - CMS + Feed Service + Interactions
- **Aggregation-Service** (Node.js) - Content ingestion + media processing workers
- **Platform-Console** (Next.js) - Admin dashboard (Platform ops + CRM)
- **CRM-Service** (Go) - Customer, deals, activities management

---

## Context Files (READ THESE FIRST)

**All modules have identical `context/` folders with 8 files:**

| File | Purpose |
|------|---------|
| `Wahb_Overall_Project_Context_Requirements.md` | Complete architecture, service boundaries, API contracts |
| `AI_Agent_Context_Requirements.md` | Service boundaries, data contracts, agent rules |
| `PRD.md` | Product requirements, feed specs, success metrics |
| `Wahb_Platform_Context_Requirements .md` | Frontend implementation (Next.js, feeds, UI) |
| `CMS_Context_Requirements.md` | Go CMS/Feed Service (endpoints, models, DB schema) |
| `Aggregation_Service_Context_Requirements.md` | Node.js workers (ingestion, FFmpeg, embeddings) |
| `Platform_Console_Context_Requirements.md` | Admin UI (CMS + CRM integration, auth flow) |
| `CRM_Context_Requirements.md` | CRM Service (customers, deals, activities, implementation status) |

**Location:** `[module-name]/context/` or root `/context/`

---

## Service Boundaries (CRITICAL - Don't Cross These)

```
┌─────────────────────────────────────────────────────────────┐
│  Aggregation Service                                   │
│  - Scrape external sources (RSS, YouTube, X, Reddit)     │
│  - Run FFmpeg (audio → MP4)                          │
│  - Generate transcripts (Whisper)                           │
│  - Generate embeddings (384-dim vectors)                  │
│  - Upload to object storage                                 │
│  - Write to CMS via APIs                                   │
│                                                          │
│  CANNOT: Serve user APIs, assemble feeds              │
└─────────────────────────────────────────────────────────────┘
                          ↓ writes via APIs
┌─────────────────────────────────────────────────────────────┐
│  CMS (Go)                                               │
│  - Read content_items                                     │
│  - Execute pgvector similarity                               │
│  - Assemble feeds (For You, News)                        │
│  - Serve public APIs (/api/v1/feed/*)                     │
│  - Content CRUD, interactions tracking                       │
│  - Issue JWTs for admin auth                               │
│                                                          │
│  CANNOT: Scrape, run FFmpeg, ingest sources        │
└─────────────────────────────────────────────────────────────┘
                ↑ serves JSON            ↑ serves JSON
┌──────────────────┐         ┌──────────────────┐
│  Wahb-Platform   │         │  Platform-Console │
│  (Next.js)        │         │  (Next.js)       │
│                   │         │                  │
│  - Feed UI        │         │  - Platform ops   │
│  - Interactions    │         │  - CRM UI        │
│  - Bottom nav     │         │  - Source mgmt   │
│                   │         │                  │
│  CANNOT: Scrape, │         │  - Calls CMS +   │
│  FFmpeg, DB write│         │    CRM APIs      │
└──────────────────┘         └──────────────────┘
      ↑ auth via CMS          ↑ auth via CMS
      (JWT Bearer)           (JWT Bearer)
                                ↓ verifies JWT
                        ┌──────────────────┐
                        │  CRM Service   │
                        │  (Go)          │
                        │                  │
                        │  - Customers     │
                        │  - Contacts      │
                        │  - Deals         │
                        │  - Activities    │
                        │  - Notes, Tags   │
                        │                  │
                        │  CANNOT: Content  │
                        │    ingestion, feed │
                        │    serving          │
                        └──────────────────┘
```

---

## Quick Prompts for Common Tasks

### Adding Feature to Service

```
I need to add [feature] to [service name].

Please:
1. Read [service]/context/[relevant_context_file].md
2. Understand service's responsibilities and boundaries
3. Check if feature fits (no crossing service boundaries)
4. Reference API contracts from Wahb_Overall_Project_Context_Requirements.md
5. Suggest implementation approach
```

### Debugging

```
I'm getting [error/issue] in [service name].

Please:
1. Read the service's context files
2. Check expected behavior from requirements
3. Suggest debugging steps
4. Check integration points with other services if relevant
```

### Cross-Service Changes

```
I need to change [something] that affects multiple services.

Please:
1. Read Wahb_Overall_Project_Context_Requirements.md for architecture
2. Identify which services are affected
3. Check service boundaries for each
4. Suggest changes to each service respecting their boundaries
5. Update API contracts if needed
```

---

## Key Technical Constraints

### Feed Contracts
- **For You:** `GET /api/v1/feed/foryou` → MP4 URLs only
- **News:** `GET /api/v1/feed/news` → slides (1 featured + 3 related)
- **Pagination:** Cursor-based for both feeds

### Auth Flow
- **CMS** issues JWTs (HS256, shared secret)
- **Console** stores token, sends `Authorization: Bearer <token>`
- **CRM** verifies JWT, enforces RBAC (admin/manager/agent)

### Data Model
- **PostgreSQL 15+** with **pgvector** extension
- **Embeddings:** 384-dimension (all-MiniLM-L6-v2)
- **Content types:** ARTICLE, VIDEO, TWEET, COMMENT, PODCAST
- **Status workflow:** PENDING → PROCESSING → READY/FAILED/ARCHIVED

### Tech Stack
| Service | Tech |
|----------|-------|
| Wahb-Platform | Next.js 15, React 19, shadcn/ui, Zustand, TanStack Query |
| CMS | Go, Gin, GORM, PostgreSQL, pgvector |
| Aggregation | Node.js, Fastify, BullMQ, Redis, FFmpeg, Whisper |
| Platform-Console | Next.js, TypeScript (calls CMS + CRM APIs) |
| CRM | Go, Gin, GORM, PostgreSQL, JWT (HS256) |

---

## Quick Reference - Which Context to Read?

| Task | Read Context In... |
|------|-------------------|
| Frontend feeds/UI | `Wahb-Platform/context/Wahb_Platform_Context_Requirements .md` |
| CMS APIs, feeds | `Content-Management-System/context/CMS_Context_Requirements.md` |
| Ingestion, media | `Aggregation-Service/context/Aggregation_Service_Context_Requirements.md` |
| Admin dashboard | `Platform-Console/context/Platform_Console_Context_Requirements.md` |
| CRM features | `CRM-Service/context/CRM_Context_Requirements.md` |
| Architecture, boundaries | Any `context/Wahb_Overall_Project_Context_Requirements.md` |
| Agent rules | Any `context/AI_Agent_Context_Requirements.md` |

---

## Implementation Status Check

**CRM-Service** has a detailed status table in `CRM_Context_Requirements.md`:
- ✅ Complete - Feature is implemented
- ⚠️ Partial - Partially implemented (check notes)
- ❌ Not Started - Not yet implemented

Always check this table before working on CRM features.

---

## Common Pitfalls to Avoid

❌ **Asking CMS to scrape websites** → This is Aggregation's job
❌ **Asking Aggregation to serve user APIs** → This is CMS's job
❌ **Asking Console to directly access queues** → Use CMS admin APIs instead
❌ **Asking CRM to handle content ingestion** → CRM only does CRM workflows
❌ **Forgetting to check service boundaries** → Read AI_Agent_Context_Requirements.md first

✅ **Ask CMS to serve feeds** → Core responsibility
✅ **Ask Aggregation to process media** → Core responsibility
✅ **Ask Console to trigger jobs via CMS** → Correct flow
✅ **Ask CRM to manage customers/deals** → Core responsibility

---

## My Working Style

1. **Always read context files first** - This is non-negotiable
2. **Respect service boundaries** - Never cross them
3. **Ask clarifying questions** - If task is ambiguous
4. **Reference API contracts** - Especially for cross-service work
5. **Check implementation status** - Especially for CRM
6. **Suggest concrete next steps** - Don't just describe concepts

---

## Quick Commands Reference

### Check git status before suggesting changes
```bash
git status
```

### View context files
```bash
# Overall architecture
cat [module]/context/Wahb_Overall_Project_Context_Requirements.md

# Service-specific
cat [module]/context/[Service]_Context_Requirements.md
```

---

## Summary

**Key Principle:** Every module has a `context/` folder with up-to-date, unified documentation.

**Golden Rule:** Read the relevant context file(s) before making any architectural or implementation decisions.

**Service Boundaries:** The most critical thing to remember - don't ask services to do things outside their scope.

**Implementation Status:** Check CRM's status table to see what's already done.

**API Contracts:** Feeds use cursor pagination, For You returns MP4s, News returns slides with 1+3 pattern.

**Auth Flow:** CMS issues JWT, Console sends it, CRM verifies it (HS256, shared secret).