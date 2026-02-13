# End-to-End Workflows

> From idea to production — a complete SaaS build through prompting, with automation and internal tooling examples.

← [DevOps](./devops.md) | [Back to Index](./README.md)

---

## The Build: A Task Management SaaS

This chapter walks through building **a complete task management SaaS** from zero to production using prompting as the primary development workflow. This is not pseudocode or theory — it's the actual sequence of prompts and decisions a technical builder would use.

### Product Specification

**Product:** TaskFlow — a collaborative task management platform for small teams.

**Core Features:**
- Workspaces with team members
- Projects within workspaces
- Tasks with status, priority, assignees, due dates
- Comments on tasks
- Activity feed (who did what)
- Real-time updates (WebSocket)
- Search across tasks

**Constraints:**
- Team: 1 full-stack engineer
- Budget: ≤ $200/month infrastructure
- Timeline: 4 weeks to MVP
- Target: 100 daily active users at launch

---

## Phase 1: Architecture (Day 1)

### Prompt 1.1 — System Design

```
You are a senior systems architect. I'm building a task management SaaS 
called TaskFlow. Design the complete system architecture.

PRODUCT:
- Workspaces → Projects → Tasks → Comments
- Team members with roles (owner, admin, member)
- Real-time updates when tasks are modified
- Full-text search across tasks
- Activity feed per workspace

CONSTRAINTS:
- Solo developer
- Budget: ≤ $200/month
- MVP in 4 weeks
- 100 DAU at launch, targeting 1K in 6 months

DECISIONS I NEED:
1. Monolith vs microservices (with justification)
2. Technology stack (backend, frontend, database, cache)
3. Database schema (ER diagram + DDL)
4. API endpoints (full list)
5. Real-time strategy (WebSocket approach)
6. Search strategy
7. Deployment target
8. What to build first (priority order)

For each decision, explain the tradeoff and what you'd change at 10x scale.
```

#### Expected Architecture Output

```mermaid
graph TB
    subgraph "Client"
        WEB[React SPA]
    end
    
    subgraph "Backend (Monolith)"
        API[FastAPI Application]
        WS[WebSocket Handler]
        WORKER[Background Worker]
    end
    
    subgraph "Data"
        PG[(PostgreSQL)]
        REDIS[(Redis)]
    end
    
    subgraph "Infrastructure"
        CDN[CDN / Static Assets]
        LB[Reverse Proxy / Caddy]
    end
    
    WEB --> LB
    LB --> API
    LB --> WS
    API --> PG
    API --> REDIS
    WS --> REDIS
    WORKER --> PG
    WORKER --> REDIS
    CDN --> WEB
```

**Architecture Decisions:**

| Decision | Choice | Reasoning |
|----------|--------|-----------|
| Architecture | Modular monolith | Solo dev, shared data, no independent scaling needs |
| Backend | Python + FastAPI | Async, fast to build, great type support |
| Frontend | React + Vite + TanStack Query | Fast dev cycle, excellent data fetching |
| Database | PostgreSQL | Relational data, full-text search, reliable |
| Cache | Redis | Sessions, real-time pub/sub, caching |
| Real-time | WebSocket via FastAPI + Redis pub/sub | Simple, no extra infrastructure |
| Search | PostgreSQL tsvector | Good enough for MVP, no extra service |
| Deployment | Single VPS (Hetzner/Railway) | < $50/month, sufficient for 1K users |

---

## Phase 2: Database Schema (Day 1-2)

### Prompt 2.1 — Schema Design

```
Generate the PostgreSQL schema for TaskFlow. Follow the architecture from 
Phase 1.

ENTITIES AND RELATIONSHIPS:
- User → belongs to many Workspaces (through membership)
- Workspace → has many Projects
- Project → has many Tasks
- Task → has many Comments
- Task → has one Assignee (optional, User)
- Task → has one Creator (User)
- Activity → polymorphic log of all actions

ACCESS PATTERNS (by frequency):
1. List tasks by project, filter by status/assignee, sort by created_at
2. Get single task with comments (eager load)
3. Activity feed for workspace (paginated, reverse chronological)
4. Search tasks by keyword across workspace
5. List projects by workspace

REQUIREMENTS:
- UUIDs for all primary keys
- Soft delete on tasks (deleted_at)
- Timestamps (created_at, updated_at) on everything
- CHECK constraints on enums
- Partial indexes for active records
- Full-text search vector on tasks (title + description)
- Multi-tenant isolation via workspace_id

OUTPUT:
1. Complete DDL (CREATE TABLE statements)
2. Mermaid ER diagram
3. Index justification table
4. Initial migration file
```

#### Schema Output (ER Diagram)

```mermaid
erDiagram
    USERS {
        uuid id PK
        varchar email UK
        varchar display_name
        varchar password_hash
        boolean email_verified
        timestamptz created_at
        timestamptz updated_at
    }
    
    WORKSPACES {
        uuid id PK
        varchar name
        varchar slug UK
        uuid owner_id FK
        timestamptz created_at
        timestamptz updated_at
    }
    
    WORKSPACE_MEMBERS {
        uuid id PK
        uuid workspace_id FK
        uuid user_id FK
        varchar role
        timestamptz joined_at
    }
    
    PROJECTS {
        uuid id PK
        uuid workspace_id FK
        varchar name
        varchar description
        varchar color
        timestamptz created_at
        timestamptz updated_at
    }

    TASKS {
        uuid id PK
        uuid project_id FK
        varchar title
        text description
        varchar status
        smallint priority
        uuid assignee_id FK
        uuid created_by FK
        timestamptz due_date
        tsvector search_vector
        timestamptz created_at
        timestamptz updated_at
        timestamptz deleted_at
    }

    COMMENTS {
        uuid id PK
        uuid task_id FK
        uuid author_id FK
        text body
        timestamptz created_at
        timestamptz updated_at
    }

    ACTIVITIES {
        uuid id PK
        uuid workspace_id FK
        uuid user_id FK
        varchar action
        varchar target_type
        uuid target_id
        jsonb metadata
        timestamptz created_at
    }

    USERS ||--o{ WORKSPACE_MEMBERS : "joins"
    WORKSPACES ||--o{ WORKSPACE_MEMBERS : "has"
    WORKSPACES ||--o{ PROJECTS : "contains"
    PROJECTS ||--o{ TASKS : "contains"
    TASKS ||--o{ COMMENTS : "has"
    USERS ||--o{ TASKS : "assigned"
    USERS ||--o{ TASKS : "created"
    USERS ||--o{ COMMENTS : "authored"
    WORKSPACES ||--o{ ACTIVITIES : "logged in"
```

---

## Phase 3: Backend Implementation (Day 2-8)

### Build Order

The build sequence is driven by dependency chains:

```
1. Auth system (everything depends on authentication)
   ├── User model + registration
   ├── Login + JWT tokens
   └── Auth middleware

2. Workspace + membership
   ├── Create workspace
   ├── Invite member
   └── Authorization middleware (workspace-scoped)

3. Projects CRUD
   └── Standard CRUD with workspace authorization

4. Tasks CRUD + business logic
   ├── Create, update, delete (soft)
   ├── Status transitions
   ├── Assignment
   └── Filtering + pagination

5. Comments
   └── CRUD under tasks

6. Activity feed
   └── Event logging + paginated read

7. Search
   └── Full-text search endpoint

8. Real-time
   └── WebSocket notifications
```

### Prompt 3.1 — Auth System

```
Implement the authentication system for TaskFlow.

STACK: FastAPI, SQLAlchemy 2.0 (async), PostgreSQL, Pydantic v2

FLOWS:
1. POST /api/v1/auth/register → create user, hash password, return tokens
2. POST /api/v1/auth/login → verify credentials, return tokens
3. POST /api/v1/auth/refresh → rotate refresh token, return new pair
4. POST /api/v1/auth/logout → revoke refresh token

TOKEN STRATEGY:
- Access: JWT RS256, 15-min expiry, contains user_id + active workspace
- Refresh: random token, 7-day expiry, stored hashed in DB, rotated on use
- Refresh token family tracking for reuse detection

PASSWORD:
- bcrypt, 12 rounds
- Minimum 10 characters

AUTH MIDDLEWARE:
- FastAPI dependency that validates JWT and injects current user
- Workspace authorization dependency that checks membership + role

FILES TO GENERATE:
1. src/auth/schemas.py — Pydantic models
2. src/auth/service.py — Auth business logic
3. src/auth/routes.py — FastAPI endpoints
4. src/auth/dependencies.py — Auth middleware
5. src/auth/tokens.py — Token generation/validation
6. tests/test_auth.py — Test suite

FOLLOW THIS EXISTING PATTERN:
[Paste your service/repository pattern if established]

SECURITY:
- No password in response payloads
- Rate limit login: 5 attempts per minute per IP
- Constant-time comparison for passwords
- No error message differentiation between "wrong email" and "wrong password"
```

### Prompt 3.2 — Resource CRUD (Pattern Prompt)

Once auth exists, use a pattern prompt for each resource:

```
Generate the full CRUD for [RESOURCE] following the TaskFlow codebase patterns.

RESOURCE: Tasks
ENTITY: Task
PARENT: Project (tasks belong to a project)

FIELDS:
- title: str, required, 1-255 chars
- description: str, optional, max 5000 chars
- status: enum(todo, in_progress, done), default: todo
- priority: int 0-3, default: 0
- assignee_id: UUID optional, must be workspace member
- due_date: datetime optional

BUSINESS RULES:
- Only workspace members can create/view tasks
- Only task creator, assignee, or workspace admin can update
- Status transitions: todo↔in_progress↔done (any direction)
- Soft delete (set deleted_at, exclude from queries)
- Creating/updating a task logs an activity event

ENDPOINTS:
- GET /api/v1/projects/{project_id}/tasks — list with filters + pagination
- POST /api/v1/projects/{project_id}/tasks — create
- GET /api/v1/tasks/{task_id} — get with comments
- PATCH /api/v1/tasks/{task_id} — partial update
- DELETE /api/v1/tasks/{task_id} — soft delete

FOLLOW EXISTING PATTERNS FROM:
[Paste auth routes/service as reference for style]

GENERATE FILES:
1. src/tasks/schemas.py
2. src/tasks/service.py
3. src/tasks/routes.py
4. tests/test_tasks.py
```

---

## Phase 4: Frontend Implementation (Day 8-16)

### Prompt 4.1 — Project Setup

```
Set up a React project for TaskFlow with:

STACK:
- Vite + React 19 + TypeScript
- TanStack Query for server state
- Zustand for client state
- TanStack Router for routing
- shadcn/ui + Radix for components
- Tailwind CSS v4

PROJECT STRUCTURE:
src/
├── components/ui/        # shadcn components
├── components/features/  # domain components
├── hooks/                # custom hooks
├── lib/                  # utilities, api client
├── stores/               # Zustand stores
├── routes/               # page components
├── types/                # TypeScript types matching backend schemas
└── styles/               # global styles, tokens

SETUP:
1. Vite config with proxy to backend (localhost:8000)
2. TanStack Query provider with default config
3. Auth provider (token storage, auto-refresh)
4. Type-safe API client matching backend endpoints
5. Route definitions (login, dashboard, project, task detail)

Generate the initial project configuration and core providers.
```

### Prompt 4.2 — Feature Component

```
Build the Task Board view for TaskFlow.

DESIGN:
- Kanban-style board with columns: Todo, In Progress, Done
- Each column shows task cards
- Drag and drop between columns to change status
- Task card shows: title, priority badge, assignee avatar, due date
- Click card → slide-open detail panel (not a page navigation)

DATA:
- Fetch tasks from GET /api/v1/projects/{projectId}/tasks
- Update status via PATCH /api/v1/tasks/{taskId}
- Optimistic update on drag-drop

AESTHETIC DIRECTION:
- Dark theme, muted background
- Cards with subtle borders, no heavy shadows
- Smooth drag animation
- Column headers with task count badges
- Empty state with illustration when no tasks

COMPONENTS TO GENERATE:
1. TaskBoard — layout with columns
2. TaskColumn — individual status column
3. TaskCard — individual task card
4. TaskDetailPanel — slide-out panel with full task details
5. useTaskBoard — hook for board logic + mutations

USE:
- @dnd-kit for drag and drop
- shadcn/ui components for UI primitives
- TanStack Query for data fetching
- Zustand for UI state (selected task, panel open)
```

---

## Phase 5: Integration and Testing (Day 16-20)

### Prompt 5.1 — End-to-End Test

```
Write end-to-end tests for the core TaskFlow user journey:

JOURNEY:
1. User registers with email/password
2. User creates a workspace
3. User creates a project in the workspace
4. User creates 3 tasks in the project
5. User moves a task from "todo" to "in_progress" 
6. User adds a comment to a task
7. User searches for a task by keyword
8. User checks the activity feed

TEST FRAMEWORK: Playwright

REQUIREMENTS:
- Test runs against a real backend (docker-compose up)
- Database is reset before each test
- Use page object model for maintainability
- Assert both UI state AND API responses
- Screenshot on failure
- Test both happy path and key error states

GENERATE:
1. e2e/fixtures/setup.ts — test setup, auth helpers
2. e2e/pages/login.page.ts — page object
3. e2e/pages/dashboard.page.ts — page object
4. e2e/pages/project.page.ts — page object
5. e2e/tests/user-journey.spec.ts — full journey test
```

### Integration Verification Prompt

```
I've built the backend and frontend separately. Now I need to verify 
they integrate correctly.

BACKEND API SPEC:
[Paste API endpoint list with request/response schemas]

FRONTEND API CLIENT:
[Paste TypeScript API client types]

CHECK:
1. Do all TypeScript types match the Pydantic response schemas?
2. Are there endpoints the frontend calls that don't exist on backend?
3. Are there missing error handling paths?
4. Does the pagination cursor format match?
5. Does the auth token refresh flow handle all edge cases?

For each mismatch found:
- Which side needs to change (or both)
- The specific fix
```

---

## Phase 6: Deployment (Day 20-24)

### Prompt 6.1 — Production Deployment

```
Set up production deployment for TaskFlow on a single VPS.

INFRASTRUCTURE:
- VPS: Hetzner CX31 (4 vCPU, 8GB RAM, ~€10/month)
- OS: Ubuntu 24.04
- Domain: taskflow.app
- TLS: Let's Encrypt via Caddy

SERVICES:
- Frontend: Static build served by Caddy
- Backend: FastAPI in Docker (2 workers)
- Database: PostgreSQL 16 in Docker
- Cache: Redis 7 in Docker
- Worker: Background job processor in Docker

DEPLOYMENT:
- docker-compose for orchestration
- Caddy as reverse proxy (auto TLS, HTTP/2, GZIP)
- GitHub Actions: build → push image → SSH deploy
- Blue-green: pull new image, start, health check, swap, stop old

BACKUP:
- PostgreSQL: pg_dump to S3-compatible storage daily
- Retention: 30 days

MONITORING:
- Uptime: UptimeRobot (free tier)
- Logs: docker logs with logrotate
- Metrics: Caddy access logs + application structured logs
- Alerts: Slack webhook on error rate spike

GENERATE:
1. docker-compose.production.yml
2. Caddyfile
3. deploy.sh (deployment script)
4. backup.sh (database backup script)
5. .github/workflows/deploy.yml
```

### Production Docker Compose

```yaml
# docker-compose.production.yml
services:
  caddy:
    image: caddy:2-alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
      - caddy_config:/config
      - ./frontend/dist:/srv/frontend
    depends_on:
      - api

  api:
    image: ghcr.io/yourorg/taskflow-api:${VERSION:-latest}
    restart: unless-stopped
    environment:
      DATABASE_URL: postgresql+asyncpg://taskflow:${DB_PASSWORD}@postgres:5432/taskflow
      REDIS_URL: redis://redis:6379/0
      JWT_PRIVATE_KEY_PATH: /run/secrets/jwt_private_key
      JWT_PUBLIC_KEY_PATH: /run/secrets/jwt_public_key
      ENVIRONMENT: production
      CORS_ORIGINS: https://taskflow.app
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    deploy:
      resources:
        limits:
          memory: 1G
          cpus: "2.0"
    healthcheck:
      test: ["CMD", "python", "-c", 
        "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"]
      interval: 15s
      timeout: 5s
      retries: 3
    secrets:
      - jwt_private_key
      - jwt_public_key

  worker:
    image: ghcr.io/yourorg/taskflow-api:${VERSION:-latest}
    restart: unless-stopped
    command: arq src.workers.WorkerSettings
    environment:
      DATABASE_URL: postgresql+asyncpg://taskflow:${DB_PASSWORD}@postgres:5432/taskflow
      REDIS_URL: redis://redis:6379/0
      ENVIRONMENT: production
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "1.0"

  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: taskflow
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: taskflow
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: pg_isready -U taskflow
      interval: 10s
      timeout: 5s
      retries: 5
    deploy:
      resources:
        limits:
          memory: 2G
          cpus: "2.0"
    command: >
      postgres
        -c shared_buffers=512MB
        -c effective_cache_size=1536MB
        -c maintenance_work_mem=128MB
        -c max_connections=100
        -c log_min_duration_statement=500

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data
    healthcheck:
      test: redis-cli ping
      interval: 10s
      timeout: 5s
      retries: 5
    deploy:
      resources:
        limits:
          memory: 300M

volumes:
  postgres_data:
  redis_data:
  caddy_data:
  caddy_config:

secrets:
  jwt_private_key:
    file: ./secrets/jwt_private.pem
  jwt_public_key:
    file: ./secrets/jwt_public.pem
```

---

## Phase 7: Post-Launch Iteration (Day 24+)

### Prompt 7.1 — Performance Optimization

```
TaskFlow has been running for 2 weeks. Here are the observed metrics:

ISSUES:
1. Task list endpoint: p99 = 800ms (target: 200ms)
   - Query: SELECT with 3 JOINs on tasks + assignee + project
   - Table size: 15K tasks
   - Current indexes: [list indexes]

2. Activity feed: p99 = 1.2s (target: 500ms)
   - Query: SELECT from activities WHERE workspace_id = ? ORDER BY created_at DESC
   - Table size: 200K activities
   - No index on (workspace_id, created_at)

3. WebSocket: connections drop after 60s
   - Nginx proxy timeout default

For each issue:
1. Root cause analysis
2. Fix (SQL, code, or config change)
3. Expected improvement
4. How to verify
```

### Prompt 7.2 — Feature Addition

```
Add a notification system to TaskFlow.

TRIGGERS:
- Assigned to a task → notification to assignee
- Comment on your task → notification to task creator
- Mentioned in comment (@username) → notification to mentioned user
- Task due date in 24 hours → notification to assignee

DELIVERY:
- In-app (bell icon with badge count)
- Email digest (daily, configurable)

ARCHITECTURE:
- Notifications table in PostgreSQL
- Redis pub/sub for real-time delivery to connected WebSocket clients
- Background worker for email digest

EXISTING CODE CONTEXT:
[Paste relevant existing patterns — activity logging, WebSocket handler]

GENERATE:
1. Database migration for notifications table
2. Notification service (create, mark read, list)
3. API endpoints (list, mark read, mark all read, preferences)
4. WebSocket broadcast integration
5. Email digest worker
6. Frontend: NotificationBell component, NotificationPanel

Follow existing codebase patterns.
```

---

## Automation Examples

### Prompt: Generate API Documentation

```
Generate OpenAPI 3.1 documentation for all TaskFlow endpoints.

SOURCE: [Paste all route files]

REQUIREMENTS:
- Every endpoint fully documented
- Request/response schemas include examples
- Authentication requirements per endpoint
- Error responses documented (401, 403, 404, 422)
- Tag by resource (Auth, Workspaces, Projects, Tasks, Comments)
- Include rate limit headers in responses

OUTPUT: openapi.yaml
```

### Prompt: Generate Database Seed Script

```
Generate a database seed script for TaskFlow development.

CREATE:
- 3 users (alice, bob, charlie) with known passwords
- 2 workspaces (alice owns both, bob is member of first)
- 3 projects (2 in workspace 1, 1 in workspace 2)
- 30 tasks across projects (varied status, priority, assignees)
- 50 comments across tasks
- 100 activity entries

REQUIREMENTS:
- Deterministic: same output every run (use fixed UUIDs)
- Idempotent: safe to run multiple times
- Include cleanup before insert
- Realistic data (not "test1", "test2" — use plausible names/titles)
- Use raw SQL or SQLAlchemy bulk insert

OUTPUT: scripts/seed.py
```

### Prompt: Generate Changelog

```
Given these git commits since the last release:

[Paste git log --oneline]

Generate a user-facing changelog following Keep a Changelog format.

CATEGORIES:
- Added (new features)
- Changed (changes in existing functionality)
- Fixed (bug fixes)
- Security (security fixes)

REQUIREMENTS:
- Write for end users, not developers
- Group related changes
- Link to relevant issues/PRs if commit messages include them
- Highlight breaking changes with ⚠️
```

---

## Internal Tool Creation

### Prompt: Admin Dashboard

```
Build an admin dashboard for TaskFlow that shows:

SECTIONS:
1. Overview: total users, workspaces, tasks created today/week/month
2. Users: searchable list, usage stats, ban/unban
3. Workspaces: member count, task count, storage usage
4. System: API response times (p50/p95/p99), error rate, active WebSockets
5. Logs: filterable structured log viewer

TECH: Simple React app with shadcn/ui, separate from main app
AUTH: Admin-only, checked via user.role == "admin"

API ENDPOINTS NEEDED:
- GET /api/v1/admin/stats
- GET /api/v1/admin/users?search=&page=
- GET /api/v1/admin/workspaces?search=&page=
- POST /api/v1/admin/users/{id}/ban
- GET /api/v1/admin/system/health
- GET /api/v1/admin/logs?level=&since=

SECURITY:
- Admin endpoints behind role check middleware
- Rate limited separately from public API
- Audit log every admin action
```

### Prompt: CLI Tool

```
Build a CLI tool for TaskFlow developers.

COMMANDS:
- taskflow db:migrate — run pending migrations
- taskflow db:seed — seed development data
- taskflow db:reset — drop all, recreate, migrate, seed
- taskflow user:create --email --password --admin — create user
- taskflow token:generate --user-id — generate API token for testing
- taskflow health — check all services (DB, Redis, API)
- taskflow logs --service --level --since — tail structured logs

TECH: Python + Click
PACKAGING: pip installable, or just a script in scripts/

REQUIREMENTS:
- Colored output
- Error messages with suggested fixes
- --help on every command
- Respects TASKFLOW_DATABASE_URL env var
```

---

## The Complete Timeline

```mermaid
gantt
    title TaskFlow MVP — 4 Week Build
    dateFormat  YYYY-MM-DD
    
    section Architecture
    System design           :a1, 2026-02-13, 1d
    Database schema         :a2, after a1, 1d
    
    section Backend
    Auth system             :b1, after a2, 2d
    Workspace + members     :b2, after b1, 1d
    Projects CRUD           :b3, after b2, 1d
    Tasks CRUD              :b4, after b3, 2d
    Comments + Activity     :b5, after b4, 1d
    Search + WebSocket      :b6, after b5, 1d
    
    section Frontend
    Project setup           :f1, after b2, 1d
    Auth pages              :f2, after f1, 1d
    Dashboard + navigation  :f3, after f2, 2d
    Task board              :f4, after f3, 3d
    Task detail + comments  :f5, after f4, 2d
    Search + real-time      :f6, after f5, 1d
    
    section Integration
    API integration         :i1, after f4, 2d
    E2E tests               :i2, after f6, 2d
    Bug fixes               :i3, after i2, 2d
    
    section Deployment
    Docker + CI/CD          :d1, after b6, 2d
    Production deploy       :d2, after i3, 1d
    Monitoring setup        :d3, after d2, 1d
    
    section Launch
    Beta testing            :l1, after d3, 2d
    Launch                  :milestone, after l1, 0d
```

---

## Prompting Workflow Summary

| Phase | # Prompts | Key Prompt Types |
|-------|-----------|-----------------|
| Architecture | 3-5 | Design, schema, API contract |
| Auth | 2-3 | Implementation, security review |
| Each resource | 1-2 | Pattern prompt (CRUD), edge cases |
| Frontend setup | 2-3 | Project config, design system, providers |
| Each view | 1-2 | Component + hook, polish |
| Integration | 2-3 | Contract validation, E2E tests |
| DevOps | 3-5 | Docker, CI, deploy, monitoring |
| **Total MVP** | **~25-35** | Over 4 weeks |

### The Key Insight

The prompts get **faster and more precise** as you build. Early prompts establish patterns. Later prompts reference those patterns:

```
"Generate the CRUD for Comments following the same pattern as Tasks.
Same auth middleware, same error handling, same test structure.
Parent entity: Task instead of Project."
```

By Week 3, a full resource (schema → API → frontend → tests) takes 2-3 prompts instead of 8-10.

---

## Common Failure Modes

| Failure | Symptom | Root Cause |
|---------|---------|------------|
| **Scope creep via prompting** | Asked for task management, ended up with project management + time tracking + Gantt charts | Each prompt adds "one more thing." Discipline: stick to MVP spec |
| **Inconsistent patterns** | Auth uses one style, tasks use another, comments use a third | Different prompts, different outputs. Establish pattern in Week 1, reference it thereafter |
| **No integration testing** | Backend and frontend built separately, don't actually work together | Frontend types diverged from backend schemas. Run integration check prompt weekly |
| **Security afterthought** | Auth works but authorization is missing | Security constraints not in every prompt. Add to system prompt |
| **No deployment until Week 4** | Everything works locally, nothing deploys | Docker should be in Week 1. Deployment should be continuous |
| **AI dependency** | Can't debug own system without AI | Review every generated line. Understand it before committing |

---

## Production Checklist

- [ ] Architecture documented and matches implementation
- [ ] Database schema has migrations for every change
- [ ] Auth system hardened (rate limiting, token rotation, CSRF)
- [ ] All CRUD endpoints have authorization checks
- [ ] Input validation on every endpoint
- [ ] Error handling is consistent across the application
- [ ] Frontend matches backend types (verified, not assumed)
- [ ] E2E tests cover the critical user journey
- [ ] Docker images are production-ready (multi-stage, non-root, health checks)
- [ ] CI pipeline runs on every PR (lint, test, security scan)
- [ ] Deployment is automated with rollback capability
- [ ] Monitoring covers the four golden signals
- [ ] Backup strategy tested (restore verified)
- [ ] All secrets are in environment variables or secret manager
- [ ] CORS, CSP, HSTS headers configured
- [ ] README documents how to run locally and deploy
- [ ] You understand every line of generated code

---

← [DevOps](./devops.md) | [Back to Index](./README.md)
