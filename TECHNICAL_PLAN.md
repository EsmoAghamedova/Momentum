# Momentum — Backend-Only Technical Plan
## Version 3.0 | July 2026

---

## Table of Contents
1. [System Overview](#system-overview)
2. [Backend Architecture](#backend-architecture)
3. [Database Design](#database-design)
4. [API Contract](#api-contract)
5. [Authentication & Authorization](#authentication--authorization)
6. [Real-Time & Event Strategy](#real-time--event-strategy)
7. [Security, Reliability, and Observability](#security-reliability-and-observability)
8. [Testing Strategy](#testing-strategy)
9. [Deployment Architecture](#deployment-architecture)
10. [6-Week Backend Delivery Plan](#6-week-backend-delivery-plan)

---

## System Overview

### Product Direction
Momentum is now a **backend-only project**. This repository defines and delivers a production-ready API platform for task and team productivity workflows.

### High-Level Architecture

```text
┌───────────────────────────────────────────────────────────────┐
│                        MOMENTUM API CORE                     │
├───────────────────────────────────────────────────────────────┤
│  API Layer (Flask/FastAPI style)                             │
│   - Auth, Tasks, Categories, Teams, Notifications, Stats     │
│                                                               │
│  Domain Services                                              │
│   - Business rules, validation, orchestration                │
│                                                               │
│  Persistence Layer                                            │
│   - PostgreSQL + ORM + migrations                            │
│                                                               │
│  Async/Event Layer (optional)                                │
│   - Redis queue / pub-sub for notifications & realtime       │
│                                                               │
│  Platform                                                     │
│   - CI/CD, logs, metrics, tracing, secret management         │
└───────────────────────────────────────────────────────────────┘
```

### Tech Stack (Backend)
- **Language**: Python 3.12
- **Web Framework**: Flask (app factory pattern)
- **ORM**: SQLAlchemy
- **Database**: PostgreSQL (Supabase or managed Postgres)
- **Auth**: JWT (access + refresh token flow)
- **Migrations**: Alembic/Flask-Migrate
- **Testing**: Pytest
- **Async/Realtime (optional)**: Socket.IO or event queue integration
- **Deployment**: Render / containerized service

---

## Backend Architecture

### Proposed Repository Structure

```text
momentum/
├── app/
│   ├── __init__.py
│   ├── models/
│   │   ├── __init__.py
│   │   ├── user.py
│   │   ├── task.py
│   │   ├── category.py
│   │   ├── team.py
│   │   ├── team_member.py
│   │   ├── notification.py
│   │   └── password_reset.py
│   ├── routes/
│   │   ├── __init__.py
│   │   ├── auth.py
│   │   ├── tasks.py
│   │   ├── categories.py
│   │   ├── teams.py
│   │   ├── notifications.py
│   │   ├── stats.py
│   │   └── settings.py
│   ├── middleware/
│   │   ├── __init__.py
│   │   ├── auth.py
│   │   └── error_handler.py
│   ├── services/
│   │   ├── __init__.py
│   │   ├── auth_service.py
│   │   ├── task_service.py
│   │   ├── team_service.py
│   │   └── notification_service.py
│   └── tests/
│       ├── __init__.py
│       ├── conftest.py
│       ├── test_auth.py
│       ├── test_tasks.py
│       ├── test_categories.py
│       ├── test_teams.py
│       └── test_notifications.py
├── migrations/
├── config.py
├── app.py
├── requirements.txt
├── .env.example
├── pytest.ini
├── README.md
└── TECHNICAL_PLAN.md
```

### Design Principles
1. **Modular by domain** (auth/tasks/teams/etc.)
2. **Thin controllers, rich services**
3. **Strict input validation + normalized error format**
4. **Idempotent and predictable API behavior**
5. **Backward-compatible endpoint evolution**

---

## Database Design

### Core Entities
- Users
- Categories
- Tasks
- Teams
- TeamMembers
- Notifications
- PasswordResetTokens

### Entity Relationships
- User 1..N Tasks
- User 1..N Categories
- Team 1..N TeamMembers
- User N..N Teams (through TeamMembers)
- User 1..N Notifications
- Task optionally belongs to Team and Category

### Key Constraints
- `users.email` unique
- `categories (user_id, name)` unique
- `team_members (team_id, user_id)` unique
- Task status enum: `planned | in_progress | done`
- Foreign keys use cascade policy where appropriate

---

## API Contract

### Base
- Base path: `/api/v1`
- Content type: `application/json`
- Auth header: `Authorization: Bearer <access_token>`

### Auth Endpoints
- `POST /auth/register`
- `POST /auth/login`
- `POST /auth/refresh`
- `GET /auth/me`
- `POST /auth/password-reset`
- `POST /auth/reset-password/{token}`

### Task Endpoints
- `GET /tasks`
- `POST /tasks`
- `GET /tasks/{id}`
- `PUT /tasks/{id}`
- `PATCH /tasks/{id}/status`
- `DELETE /tasks/{id}`

### Category Endpoints
- `GET /categories`
- `POST /categories`
- `PUT /categories/{id}`
- `DELETE /categories/{id}`

### Team Endpoints
- `GET /teams`
- `POST /teams`
- `GET /teams/{id}`
- `PUT /teams/{id}`
- `POST /teams/{id}/members`
- `GET /teams/{id}/members`
- `DELETE /teams/{id}/members/{user_id}`

### Notification Endpoints
- `GET /notifications`
- `PATCH /notifications/{id}/read`
- `DELETE /notifications/{id}`

### Stats & Settings
- `GET /stats`
- `GET /stats/activity`
- `GET /settings`
- `PUT /settings/profile`
- `PUT /settings/onboarding`

### Standard Response Envelope

```json
{
  "success": true,
  "data": {},
  "error": null,
  "meta": {}
}
```

---

## Authentication & Authorization

### Strategy
- Access token: short-lived (15m)
- Refresh token: longer-lived (7d), rotated
- Password hashing: strong algorithm (bcrypt/argon2)
- Optional email verification and password reset token expiry

### Authorization Rules
- User can access only owned resources unless team role allows shared access
- Team admin/member permissions enforced at service layer
- All write endpoints require authentication

---

## Real-Time & Event Strategy

This backend remains fully functional without realtime. Optional realtime layer can be added for:
- task update broadcasts
- notification push
- team activity feed

Recommended approach:
- Emit domain events from services
- Handle with background worker or pub/sub channel
- Keep REST API as source of truth

---

## Security, Reliability, and Observability

### Security
- Strict CORS allowlist
- Input validation on all endpoints
- Rate limit auth endpoints
- Secure secret storage (no secrets in repo)
- SQL injection/XSS-safe patterns through ORM and validation

### Reliability
- DB transaction boundaries per request unit of work
- Retry strategy for transient infrastructure failures
- Health checks: `/health/live`, `/health/ready`

### Observability
- Structured logging (JSON)
- Metrics: request latency, error rate, throughput
- Trace IDs per request
- Audit logs for sensitive actions

---

## Testing Strategy

### Test Layers
1. Unit tests (services, validators)
2. Integration tests (routes + db)
3. Contract tests (request/response schema)
4. Smoke tests (post-deploy health)

### Quality Gates
- CI runs lint + tests on all pushes
- Minimum target coverage: 80%
- Block deploy on failing tests

---

## Deployment Architecture

### CI/CD Pipeline
1. Lint + static checks
2. Unit/integration tests
3. Build artifact/container
4. Deploy to staging
5. Smoke tests
6. Promote to production

### Environment Variables (example)

```env
APP_ENV=production
DATABASE_URL=postgresql://...
JWT_SECRET_KEY=...
JWT_ACCESS_TOKEN_EXPIRES=900
JWT_REFRESH_TOKEN_EXPIRES=604800
CORS_ORIGINS=https://your-client-domain.com
LOG_LEVEL=INFO
```

---

## 6-Week Backend Delivery Plan

### Week 1 — Foundation
- Project bootstrap, config, app factory
- DB connection + migrations setup
- Auth model and base middleware

### Week 2 — Auth
- Register/login/refresh/me
- Password reset flow
- Auth tests + error handling standardization

### Week 3 — Core Task Domain
- Categories CRUD
- Tasks CRUD + status transitions
- Filtering/search/pagination

### Week 4 — Collaboration
- Teams + memberships
- Permission rules
- Notification persistence

### Week 5 — Quality & Operations
- Stats/settings endpoints
- Logging, metrics, health checks
- CI quality gates and coverage target

### Week 6 — Hardening & Release
- Security review
- Performance tuning and indexing
- Staging validation + production release checklist

---

## Final Notes
This document intentionally removes all frontend architecture details and defines Momentum as a backend-first, client-agnostic API platform.
