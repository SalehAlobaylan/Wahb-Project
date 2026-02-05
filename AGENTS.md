# Wahb Project - Agent Configuration

**Last Updated:** February 3, 2026
**Project Type:** Mono-repo with 5 services + 1 frontend app

---

## Project Overview

**Wahb** is a mobile-first social platform combining:
- **For You Feed:** TikTok-style full-screen audio/video content (MP4)
- **News Feed:** Magazine-style editorial slides (1 featured + 3 related items)

**Core Services:**
1. **Wahb-Platform** - Next.js frontend web app (consumer UI)
2. **Content-Management-System** - Go backend (CMS + Feed Service + Interactions)
3. **Aggregation-Service** - Node.js worker fleet (content ingestion + media processing)
4. **Platform-Console** - Next.js admin dashboard (CRM + Platform operations)
5. **CRM-Service** - Go backend (CRM workflows)

---

## Context Files Location

**All modules contain identical context files in their `context/` folders:**

```
Wahb-Project/
├── context/                           # Root mono-repo context
├── Wahb-Platform/
│   └── context/                      # Frontend context
├── Content-Management-System/
│   └── context/
├── CRM-Service/
│   └── context/
├── Aggregation-Service/
│   └── context/
└── Platform-Console/
    └── context/
```

**Context Files:**
- `Wahb_Overall_Project_Context_Requirements.md` - Complete project context, architecture
- `Wahb_Platform_Context_Requirements .md` - Frontend implementation details
- `PRD.md` - Product Requirements Document
- `LLM_Context_Requirements.md` - AI agent guidelines and constraints
- `Aggregation_Service_Context_Requirements.md` - Worker fleet requirements
- `Platform_Console_Context_Requirements.md` - Admin dashboard requirements
- `CMS_Context_Requirements.md` - CMS/Feed Service specifications
- `CRM_Context_Requirements.md` - CRM Service requirements (most detailed with status table)

---

## Service Boundaries (CRITICAL)

| Service | CAN Do | CANNOT Do |
|---------|----------|-------------|
| **Wahb-Platform** | UI, feed display, interactions | Scraping, media processing |
| **CMS** | Serve feeds, content CRUD, pgvector search | Scraping, FFmpeg |
| **Aggregation** | Scrape, FFmpeg, transcribe, embed | Serve user traffic, assemble feeds |
| **Console** | Admin UI, trigger jobs, CRM workflows | Direct queue access |
| **CRM** | Customer/deal/activity management | Content ingestion, feed serving |

---

## Common Prompts by Module

### For Wahb-Platform (Frontend)

```
I need to modify the [For You/News] feed UI in Wahb-Platform.
Current implementation uses Next.js 15, React 19, shadcn/ui, Zustand, and TanStack Query.
Please reference:
- Wahb-Platform/context/Wahb_Platform_Context_Requirements .md
- Wahb-Platform/context/PRD.md
```

### For Content-Management-System (Backend)

```
I need to add/update [feed endpoints/content retrieval/interactions] in CMS.
Current stack: Go + Gin + GORM + PostgreSQL + pgvector.
Please reference:
- Content-Management-System/context/CMS_Context_Requirements.md
- Content-Management-System/context/Wahb_Overall_Project_Context_Requirements.md
```

### For Aggregation-Service (Workers)

```
I need to add/support [RSS/YouTube/Podcast] ingestion in Aggregation Service.
Current stack: Node.js + Fastify + BullMQ + Redis + FFmpeg + Whisper.
Please reference:
- Aggregation-Service/context/Aggregation_Service_Context_Requirements.md
- Aggregation-Service/context/LLM_Context_Requirements.md
```

### For Platform-Console (Admin UI)

```
I need to add/update [platform admin/CRM] features in Platform Console.
Current stack: Next.js, TypeScript, calls CMS + CRM APIs.
Please reference:
- Platform-Console/context/Platform_Console_Context_Requirements.md
- Platform-Console/context/CRM_Context_Requirements.md
```

### For CRM-Service (CRM Backend)

```
I need to add/update [customers/contacts/deals/activities] in CRM Service.
Current stack: Go + Gin + GORM + PostgreSQL + JWT auth (HS256).
Please reference:
- CRM-Service/context/CRM_Context_Requirements.md
- CRM-Service/context/Wahb_Overall_Project_Context_Requirements.md
```

---

## General Task Prompts

### Add New Feature
```
I want to add [feature description] to [service name].

Please:
1. Read the context file for that service
2. Understand the service's role and boundaries
3. Check if this feature fits within the service's responsibilities
4. Suggest implementation approach considering the overall system architecture
5. Reference any relevant API contracts or data models
```

### Debug Issue
```
I'm experiencing [issue description] in [service name].

Please:
1. Read the relevant context files for that service
2. Understand the expected behavior from requirements
3. Suggest debugging steps
4. Check for potential integration issues with other services
5. Review logs/errors if available
```

### Update Documentation
```
I need to update documentation for [feature/component] in [service].

Please:
1. Read the relevant context file
2. Identify what needs to be updated
3. Keep documentation consistent with other services
4. Ensure technical accuracy
5. Include examples where helpful
```

### Code Review
```
Please review this code for [service name] [paste code here].

Please:
1. Read the service's context requirements
2. Check for adherence to service boundaries
3. Verify API contract compliance
4. Check for security issues (SQL injection, XSS, auth bypass)
5. Suggest improvements
```

---

## System-Level Prompts

### Architecture Questions
```
I have a question about the overall Wahb platform architecture.

Please reference:
- Any module's context/Wahb_Overall_Project_Context_Requirements.md
- context/LLM_Context_Requirements.md

Focus on:
- Service boundaries
- Data flow between services
- API contracts
- Technology stack consistency
```

### Cross-Service Integration
```
I need to integrate [service A] with [service B].

Please reference:
- Both services' context files
- Overall project context

Consider:
- API contracts between services
- Authentication/authorization flow
- Error handling
- Data synchronization
```

### Performance Optimization
```
I need to optimize [feature/service] for performance.

Please:
1. Read the relevant context
2. Identify bottlenecks based on system design
3. Consider: pgvector queries, feed pagination, media processing, database indexes
4. Suggest caching strategies if applicable
5. Balance optimization with maintainability
```

---

## Important Notes for Agents

### 1. Always Read Context First
Before working on any module, read its `context/` folder to understand:
- Service responsibilities and boundaries
- Technology stack
- API contracts
- Data models
- Implementation status (especially for CRM)

### 2. Respect Service Boundaries
Never ask a service to do something outside its scope. For example:
- ❌ Ask CMS to scrape websites
- ❌ Ask Aggregation to serve user-facing APIs
- ❌ Ask Console to run FFmpeg
- ✅ Ask CMS to serve feeds
- ✅ Ask Aggregation to ingest content
- ✅ Ask Console to trigger ingestion via CMS APIs

### 3. Use Correct Terminology
- **Feed Service:** Part of CMS - assembles feeds, runs pgvector
- **Aggregation Service:** Worker fleet - ingests, processes, enriches content
- **Platform Console:** Admin UI - calls CMS + CRM APIs
- **CRM Service:** Backend for CRM - manages customers, deals, activities
- **CMS:** Backend - serves feeds + content + interactions

### 4. Check Implementation Status
**CRM_Context_Requirements.md** contains a detailed implementation status table:
- ✅ Complete: Feature is implemented
- ⚠️ Partial: Partially implemented (check notes)
- ❌ Not Started: Not yet implemented

### 5. Key Technical Constraints
- **For You Feed:** Must return MP4 URLs (audio must be converted upstream)
- **News Feed:** Must return slides with 1 featured + 3 related items
- **Vector Search:** Uses 384-dimension embeddings (all-MiniLM-L6-v2) in pgvector
- **Auth:** HS256 JWT with shared secret between CMS and CRM
- **Database:** PostgreSQL 15+ with pgvector extension

### 6. Feed API Contracts
**For You Feed** (`GET /api/v1/feed/foryou`):
- Cursor-based pagination
- Returns MP4 URLs
- Includes engagement counts and interaction flags

**News Feed** (`GET /api/v1/feed/news`):
- Cursor-based pagination
- Returns slides with 1 featured + 3 related items per slide
- Featured items are ARTICLE type
- Related items are TWEET or COMMENT type

### 7. CRM Auth Flow
- Console authenticates via CMS (`POST /admin/auth/login`)
- CMS issues JWT (HS256) with role claims
- Console stores token and attaches to all requests (`Authorization: Bearer <token>`)
- CRM verifies JWT (shared secret) and enforces RBAC
- On 401/403, redirect to login

---

## Quick Reference

### Module Mapping
| If working on... | Reference context in... |
|----------------|------------------------|
| Frontend UI (feeds) | `Wahb-Platform/context/Wahb_Platform_Context_Requirements .md` |
| Feed APIs / Content CRUD | `Content-Management-System/context/CMS_Context_Requirements.md` |
| Content ingestion / Media processing | `Aggregation-Service/context/Aggregation_Service_Context_Requirements.md` |
| Admin dashboard UI | `Platform-Console/context/Platform_Console_Context_Requirements.md` |
| CRM workflows | `Platform-Console/context/CRM_Context_Requirements.md` or `CRM-Service/context/CRM_Context_Requirements.md` |
| System architecture | Any `context/Wahb_Overall_Project_Context_Requirements.md` |
| AI agent guidelines | Any `context/LLM_Context_Requirements.md` |

### File Structure Reminder
```
context/
├── Wahb_Overall_Project_Context_Requirements.md    # Start here for architecture
├── LLM_Context_Requirements.md                   # AI agent rules
├── PRD.md                                        # Product requirements
├── Wahb_Platform_Context_Requirements .md         # Frontend only
├── CMS_Context_Requirements.md                     # Backend (CMS)
├── Aggregation_Service_Context_Requirements.md       # Workers
├── Platform_Console_Context_Requirements.md        # Admin UI
└── CRM_Context_Requirements.md                   # CRM (most detailed)
```

---

## Getting Started

When a user provides a task prompt:

1. **Identify the module** being worked on (e.g., "CMS", "frontend", "workers")
2. **Read the relevant context file(s)** from that module's `context/` folder
3. **Check service boundaries** to ensure the task fits
4. **Ask clarifying questions** if the task is ambiguous
5. **Provide implementation guidance** based on context
6. **Cross-reference** with other services if integration is needed

---

## Summary

This agent configuration ensures that:
- ✅ All AI agents have consistent context across the mono-repo
- ✅ Service boundaries are respected
- ✅ Technology stack and contracts are aligned
- ✅ Implementation status is tracked (especially CRM)
- ✅ Common prompts are available for quick reference

**Always reference context files before making architectural or implementation decisions!**