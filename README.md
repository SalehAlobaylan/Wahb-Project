# Wahb Project

A mobile-first social platform combining immersive audio discovery with premium journalism.

**Project Type:** Mono-repo with 5 services + 1 frontend app

## Repository Structure

```
Wahb-Project/
├── context/                           # Unified context files for all modules
│   ├── Turfa_Overall_Project_Context_Requirements.md
│   ├── LLM_Context_Requirements.md
│   ├── PRD.md
│   ├── CMS_Context_Requirements.md
│   ├── CRM_Context_Requirements.md
│   ├── Aggregation_Service_Context_Requirements.md
│   ├── Platform_Console_Context_Requirements.md
│   └── Turfa_Platform_Context_Requirements .md
├── Aggregation-Service/               # Content ingestion worker fleet
├── Content-Management-System/           # Go backend (CMS + Feed Service)
├── CRM-Service/                         # Go backend (CRM workflows)
├── Platform-Console/                    # Next.js admin dashboard
└── Wahb-Platform/                       # Next.js frontend web app
```

## Core Services

1. **Wahb-Platform** - Next.js frontend (consumer UI for feeds)
2. **Content-Management-System** - Go backend (CMS + Feed Service + Interactions)
3. **Aggregation-Service** - Node.js workers (content ingestion + media processing)
4. **Platform-Console** - Next.js admin dashboard (CRM + Platform operations)
5. **CRM-Service** - Go backend (CRM workflows)

## Quick Links

- **Project Overview:** `context/Turfa_Overall_Project_Context_Requirements.md`
- **AI Agent Guide:** `agent.md` or `CLAUDE.md`
- **Service Boundaries:** `context/LLM_Context_Requirements.md`

## Getting Started

Each service has its own README and setup instructions in its respective folder.

## Context Files

All modules contain identical `context/` folders with unified documentation:
- Latest versions (Jan 25-31, 2026)
- 8 context files covering architecture, requirements, and API contracts
- Ensures consistent AI agent guidance across all services

---

*Formerly known as: Turfa / Lumen*
