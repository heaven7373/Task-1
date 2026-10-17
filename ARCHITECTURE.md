# SkillTrack: Architecture Document

A small full-stack web app for tracking learning goals, study sessions, and resources, with progress analytics.

---

## 1. Product Idea

**Problem:** Self-directed learners juggle many topics (e.g., Java, Python, ML) with no single place to see goals, time spent, and resources.

**Solution:** SkillTrack lets a user create learning goals, break them into topics, log study sessions, attach resources, and view progress and streaks.

**Target user:** Students and early-career developers.

**MVP scope**
- Sign up / log in
- CRUD for goals and topics
- Log study sessions (duration, notes)
- Save resources (links) per topic
- Dashboard: weekly hours, streak, goal completion

**Out of scope (v2):** social features, AI recommendations, mobile app, notifications.

---

## 2. Tech Stack

| Layer | Choice | Reason |
|---|---|---|
| Frontend | React 18 + Vite + TypeScript, React Router, TanStack Query, Tailwind CSS | Fast dev, simple server-state caching |
| Backend | Python FastAPI | Auto OpenAPI docs, typed, quick to build |
| ORM / Migrations | SQLAlchemy 2.0 + Alembic | Standard, migration support |
| Database | PostgreSQL 16 | Relational data with aggregations |
| Auth | JWT (access token) + bcrypt password hashing | Stateless, simple |
| Testing | Pytest (API), Vitest + RTL (UI) | |
| DevOps | Docker Compose, GitHub Actions CI | One-command setup |
| Deploy | Render/Railway (API + DB), Vercel (frontend) | Free tiers |

---

## 3. System Architecture

```
┌──────────────┐   HTTPS/JSON   ┌───────────────────┐    SQL    ┌─────────────┐
│ React SPA    │ ─────────────► │ FastAPI           │ ────────► │ PostgreSQL  │
│ (Vite, TS)   │ ◄───────────── │ routers → services│ ◄──────── │             │
└──────────────┘    JWT Bearer  │ → repositories    │           └─────────────┘
                                └───────────────────┘
```

**Backend layering:** `routers` (HTTP) → `services` (business logic) → `models` (SQLAlchemy). Pydantic `schemas` validate input/output.

---

## 4. Data Models

### ERD (relationships)
- User 1—N Goal
- Goal 1—N Topic
- Topic 1—N Session
- Topic 1—N Resource

### Tables

**users**
| Column | Type | Notes |
|---|---|---|
| id | UUID PK | |
| email | VARCHAR(255) | unique, indexed |
| name | VARCHAR(100) | |
| password_hash | VARCHAR(255) | bcrypt |
| created_at | TIMESTAMPTZ | default now() |

**goals**
| Column | Type | Notes |
|---|---|---|
| id | UUID PK | |
| user_id | UUID FK → users | ON DELETE CASCADE |
| title | VARCHAR(150) | |
| description | TEXT | nullable |
| target_date | DATE | nullable |
| status | ENUM(active, completed, archived) | default active |
| created_at / updated_at | TIMESTAMPTZ | |

**topics**
| Column | Type | Notes |
|---|---|---|
| id | UUID PK | |
| goal_id | UUID FK → goals | CASCADE |
| title | VARCHAR(150) | |
| status | ENUM(todo, in_progress, done) | default todo |
| position | INT | ordering within goal |
| created_at | TIMESTAMPTZ | |

**sessions**
| Column | Type | Notes |
|---|---|---|
| id | UUID PK | |
| user_id | UUID FK → users | denormalized for fast stats |
| topic_id | UUID FK → topics | CASCADE |
| minutes | INT | CHECK > 0 |
| notes | TEXT | nullable |
| studied_on | DATE | indexed |
| created_at | TIMESTAMPTZ | |

**resources**
| Column | Type | Notes |
|---|---|---|
| id | UUID PK | |
| topic_id | UUID FK → topics | CASCADE |
| title | VARCHAR(200) | |
| url | TEXT | |
| type | ENUM(video, article, book, course, other) | |
| created_at | TIMESTAMPTZ | |

**Indexes:** `users(email)`, `goals(user_id, status)`, `sessions(user_id, studied_on)`, `topics(goal_id, position)`.

---

## 5. API Design

Base path: `/api/v1`. All routes except `/auth/*` require `Authorization: Bearer <token>`. Users can access only their own data (ownership checked in services).

### Auth
| Method | Route | Description |
|---|---|---|
| POST | `/auth/register` | Create account → returns token |
| POST | `/auth/login` | Returns JWT |
| GET | `/auth/me` | Current user |

### Goals
| Method | Route | Description |
|---|---|---|
| GET | `/goals?status=` | List user goals with progress % |
| POST | `/goals` | Create goal |
| GET | `/goals/{id}` | Goal with topics |
| PATCH | `/goals/{id}` | Update |
| DELETE | `/goals/{id}` | Delete (cascades) |

### Topics
| Method | Route | Description |
|---|---|---|
| POST | `/goals/{goal_id}/topics` | Add topic |
| PATCH | `/topics/{id}` | Update title/status/position |
| DELETE | `/topics/{id}` | Delete |

### Sessions
| Method | Route | Description |
|---|---|---|
| GET | `/sessions?from=&to=&topic_id=` | List sessions |
| POST | `/topics/{topic_id}/sessions` | Log session |
| PATCH | `/sessions/{id}` | Edit |
| DELETE | `/sessions/{id}` | Delete |

### Resources
| Method | Route | Description |
|---|---|---|
| GET | `/topics/{topic_id}/resources` | List |
| POST | `/topics/{topic_id}/resources` | Add |
| DELETE | `/resources/{id}` | Remove |

### Stats
| Method | Route | Description |
|---|---|---|
| GET | `/stats/summary` | Total hours, current streak, active goals |
| GET | `/stats/weekly` | Minutes per day, last 7 days |
| GET | `/stats/by-goal` | Minutes per goal |

### Conventions
- Success: `200/201/204`. Errors: `{ "detail": "message", "code": "VALIDATION_ERROR" }` with `400/401/403/404/422`.
- Pagination: `?limit=20&offset=0` on list endpoints.
- Health check: `GET /health`.

---

## 6. Frontend Design

### Screens and routes
| Route | Screen | Key components |
|---|---|---|
| `/login`, `/register` | Auth | AuthForm |
| `/` | Dashboard | StatCards, WeeklyChart, StreakBadge, RecentSessions |
| `/goals` | Goals list | GoalCard (progress bar), NewGoalModal, status filter |
| `/goals/:id` | Goal detail | TopicList (drag reorder), TopicStatusToggle, AddTopicInput |
| `/topics/:id` | Topic detail | SessionLog, LogSessionForm, ResourceList, AddResourceForm |
| `/sessions` | History | DateRangeFilter, SessionTable |
| `/settings` | Profile | ProfileForm, ChangePassword |

### Frontend structure
- `api/`: typed fetch client with token injection + 401 redirect
- `hooks/`: TanStack Query hooks (`useGoals`, `useSessions`, `useStats`)
- `context/AuthContext`: token + user, stored in memory with localStorage persistence
- `components/ui/`: Button, Modal, Input, ProgressBar
- `ProtectedRoute` wrapper for authenticated pages

### State
- Server state: TanStack Query (cache, invalidation after mutations)
- UI state: local component state

---

## 7. Security and Non-Functional

- Passwords hashed with bcrypt; JWT expiry 60 min
- CORS restricted to frontend origin
- Input validation via Pydantic; parameterized queries via ORM
- Rate-limit `/auth/*` (slowapi)
- Secrets in environment variables (`.env`, never committed)
- Target: p95 API latency < 300 ms at small scale

---

## 8. Repository Structure (Starter)

```
skilltrack/
├── README.md
├── docker-compose.yml
├── .env.example
├── .github/workflows/ci.yml
├── docs/ARCHITECTURE.md
├── backend/
│   ├── requirements.txt
│   ├── alembic/
│   └── app/
│       ├── main.py
│       ├── core/        (config.py, security.py, db.py)
│       ├── models/      (user.py, goal.py, topic.py, session.py, resource.py)
│       ├── schemas/
│       ├── routers/     (auth.py, goals.py, topics.py, sessions.py, resources.py, stats.py)
│       ├── services/
│       └── tests/
└── frontend/
    ├── package.json
    ├── vite.config.ts
    └── src/
        ├── main.tsx
        ├── App.tsx
        ├── api/
        ├── hooks/
        ├── context/
        ├── components/
        └── pages/
```

---

## 9. Starter Commands

```bash
# Create repo skeleton
mkdir skilltrack && cd skilltrack && git init
npm create vite@latest frontend -- --template react-ts
mkdir -p backend/app/{core,models,schemas,routers,services,tests}
python -m venv backend/.venv && source backend/.venv/bin/activate
pip install fastapi "uvicorn[standard]" sqlalchemy alembic psycopg2-binary \
  pydantic-settings "python-jose[cryptography]" passlib[bcrypt] pytest httpx
pip freeze > backend/requirements.txt

# Run
docker compose up -d db
uvicorn app.main:app --reload --app-dir backend

# Publish
git add . && git commit -m "Initial architecture + scaffold"
git remote add origin https://github.com/<username>/skilltrack.git
git push -u origin main
```

---

## 10. Milestones

1. **Week 1:** DB models, migrations, auth endpoints
2. **Week 2:** Goals/topics/sessions/resources CRUD + tests
3. **Week 3:** Frontend auth, goals, topic screens
4. **Week 4:** Stats endpoints, dashboard, Docker, CI, deploy

---

## 11. Risks and Decisions

| Decision | Alternative | Why |
|---|---|---|
| FastAPI | Spring Boot | Faster to prototype; Spring Boot is a valid swap |
| JWT | Session cookies | Simpler for SPA + API split |
| Denormalized `user_id` on sessions | Join through topic/goal | Cheaper stats queries |
| PostgreSQL | SQLite | Needed for date aggregations and deployment parity |
