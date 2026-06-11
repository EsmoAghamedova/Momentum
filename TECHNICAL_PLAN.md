# Momentum - Technical Architecture Plan
## Version 1.0 | June 2026

---

## TABLE OF CONTENTS
1. [System Overview](#system-overview)
2. [Backend Architecture](#backend-architecture)
3. [Frontend Architecture](#frontend-architecture)
4. [Database Design](#database-design)
5. [API Contract](#api-contract)
6. [Authentication Flow](#authentication-flow)
7. [Data Flow Diagrams](#data-flow-diagrams)
8. [Deployment Architecture](#deployment-architecture)
9. [6-Week Sprint Plan](#6-week-sprint-plan)

---

## SYSTEM OVERVIEW

### High-Level Architecture
```
┌─────────────────────────────────────────────────────────────┐
│                    MOMENTUM SYSTEM                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────────┐         ┌───────────────────────┐ │
│  │   FRONTEND (React)   │         │  BACKEND (Flask API)  │ │
│  │  - Vercel Hosting    │◄───────►│  - Render Hosting     │ │
│  │  - Single Page App   │  REST   │  - SQLAlchemy ORM     │ │
│  │  - Vite Dev Server   │  HTTP   │  - JWT Auth           │ │
│  └──────────────────────┘         └───────────────────────┘ │
│           ▲                                 ▲                 │
│           │                                 │                 │
│           │                                 │                 │
│           └─────────────────┬───────────────┘                 │
│                             │                                 │
│                    ┌────────▼────────┐                       │
│                    │   SUPABASE DB   │                       │
│                    │  (PostgreSQL)   │                       │
│                    └─────────────────┘                       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Tech Stack Summary
- **Backend**: Python 3.12 + Flask
- **Frontend**: React 18 + Vite
- **Database**: Supabase (PostgreSQL)
- **Authentication**: JWT (JSON Web Tokens)
- **Hosting**: Render (Backend) + Vercel (Frontend)
- **Version Control**: GitHub (Monorepo)

---

## BACKEND ARCHITECTURE

### 1. PROJECT STRUCTURE

```
momentum/api/
│
├── app/
│   ├── __init__.py                 # App factory (creates Flask app)
│   ├── models/
│   │   ├── __init__.py
│   │   ├── user.py                 # User model
│   │   ├── task.py                 # Task model
│   │   └── category.py             # Category model
│   │
│   ├── routes/
│   │   ├── __init__.py
│   │   ├── auth.py                 # Login, Register, Token Refresh
│   │   ├── tasks.py                # Task CRUD, Status Update, Search, Filter
│   │   ├── categories.py           # Category CRUD
│   │   └── stats.py                # Dashboard stats, Activity data
│   │
│   ├── middleware/
│   │   ├── __init__.py
│   │   └── auth.py                 # JWT validation decorator
│   │
│   └── tests/
│       ├── __init__.py
│       ├── conftest.py             # Pytest fixtures
│       ├── test_auth.py            # Auth endpoint tests
│       ├── test_tasks.py           # Task endpoint tests
│       └── test_categories.py      # Category endpoint tests
│
├── config.py                        # Environment configs (dev, prod, test)
├── app.py                          # Entry point
├── requirements.txt                # Python dependencies
├── .env.example                    # Environment variables template
├── .gitignore
├── pytest.ini                      # Pytest configuration
└── README.md                       # Backend setup guide
```

### 2. Core Components

#### **App Factory Pattern**
- **Purpose**: Creates Flask application instance
- **Location**: `app/__init__.py`
- **Responsibilities**:
  - Initialize Flask app
  - Configure extensions (SQLAlchemy, JWT, CORS)
  - Register blueprints (routes)
  - Create database tables
  - Set up error handlers

#### **Configuration Management** (`config.py`)
- **Three environments**:
  1. **Development**: SQLite, Debug=True, CORS permissive
  2. **Production**: PostgreSQL (Supabase), Debug=False, CORS restricted
  3. **Testing**: In-memory SQLite, isolated tests
- **Manages**:
  - Database connection strings
  - JWT secret and expiration times
  - CORS allowed origins
  - Environment-specific flags

#### **Models (Data Layer)**
Located in `app/models/`, define database schema:

1. **User Model** (`user.py`)
   - Fields: id, name, email, password_hash, created_at
   - Methods: set_password(), check_password(), to_dict()
   - Relationships: owns multiple Tasks and Categories

2. **Task Model** (`task.py`)
   - Fields: id, title, deadline, status, category_id, user_id, created_at, updated_at
   - Status options: "Planned", "In Progress", "Done"
   - Methods: to_dict()
   - Relationships: belongs to User and Category

3. **Category Model** (`category.py`)
   - Fields: id, name, user_id, created_at
   - Constraint: user_id + name must be unique
   - Methods: to_dict()
   - Relationships: belongs to User, has multiple Tasks

#### **Routes (API Endpoints)** - Located in `app/routes/`

1. **auth.py**: Authentication routes
   - Register, Login, Token Refresh, Get Current User

2. **tasks.py**: Task management routes
   - CRUD operations, Status changes, Search, Filter, Paginate

3. **categories.py**: Category management routes
   - List, Create

4. **stats.py**: Dashboard statistics routes
   - Dashboard stats, Activity data

#### **Middleware** (`app/middleware/`)
- **JWT Validation Decorator**:
  - Checks token validity
  - Extracts user ID
  - Protects routes

### 3. Request/Response Pattern

```
REQUEST → Processing → RESPONSE

Request:
├── HTTP method + URL endpoint
├── Headers (JWT token in Authorization header)
└── Body (JSON data for POST/PUT/PATCH)

Processing:
├── Route matching
├── JWT validation
├── Database query
├── Business logic
└── Database commit

Response:
├── Status code (200, 201, 400, 401, 404, 500)
├── Response body (JSON)
└── Headers
```

---

## FRONTEND ARCHITECTURE

### 1. PROJECT STRUCTURE

```
momentum/web/
│
├── src/
│   ├── main.jsx                    # Entry point
│   ├── App.jsx                     # Root component
│   │
│   ├── pages/
│   │   ├── Dashboard.jsx           # Main dashboard with stats
│   │   ├── Kanban.jsx              # Kanban board view
│   │   ├── TaskList.jsx            # List view of tasks
│   │   ├── Login.jsx               # Login page
│   │   ├── Register.jsx            # Registration page
│   │   └── NotFound.jsx            # 404 page
│   │
│   ├── components/
│   │   ├── Layout/
│   │   │   ├── Navbar.jsx          # Navigation bar
│   │   │   ├── Sidebar.jsx         # Side menu
│   │   │   └── Footer.jsx          # Footer
│   │   │
│   │   ├── Task/
│   │   │   ├── TaskCard.jsx        # Individual task display
│   │   │   ├── TaskModal.jsx       # Create/Edit modal
│   │   │   └── TaskForm.jsx        # Form inputs
│   │   │
│   │   ├── Kanban/
│   │   │   ├── KanbanColumn.jsx    # Single column
│   │   │   ├── KanbanCard.jsx      # Card in kanban
│   │   │   └── DragDropManager.jsx # Drag-drop logic
│   │   │
│   │   ├── Dashboard/
│   │   │   ├── StatsCard.jsx       # Stat display
│   │   │   ├── ProgressChart.jsx   # Recharts visualization
│   │   │   └── QuickAdd.jsx        # Quick add button
│   │   │
│   │   ├── Auth/
│   │   │   ├── ProtectedRoute.jsx  # Wrapper for protected pages
│   │   │   └── AuthGuard.jsx       # Redirect if not authenticated
│   │   │
│   │   ├── Common/
│   │   │   ├── Loading.jsx         # Loading spinner
│   │   │   ├── Error.jsx           # Error message
│   │   │   └── EmptyState.jsx      # Empty state message
│   │   │
│   │   └── Filter/
│   │       ├── StatusFilter.jsx    # Filter by status
│   │       ├── CategoryFilter.jsx  # Filter by category
│   │       └── SearchBox.jsx       # Search tasks
│   │
│   ├── api/
│   │   ├── client.js               # Axios HTTP client
│   │   ├── auth.js                 # Authentication API calls
│   │   ├── tasks.js                # Task API calls
│   │   ├── categories.js           # Category API calls
│   │   └── stats.js                # Stats API calls
│   │
│   ├── store/
│   │   ├── authStore.js            # Auth state (Zustand or Context)
│   │   ├── taskStore.js            # Task state
│   │   └── categoryStore.js        # Category state
│   │
│   ├── hooks/
│   │   ├── useAuth.js              # Auth logic hook
│   │   ├── useTasks.js             # Task management hook
│   │   └── useCategories.js        # Category management hook
│   │
│   ├── utils/
│   │   ├── dateUtils.js            # Date formatting helpers
│   │   ├── tokenUtils.js           # JWT token handling
│   │   └── validators.js           # Form validation
│   │
│   ├── styles/
│   │   ├── globals.css             # Global styles
│   │   ├── tailwind.config.js      # Tailwind configuration
│   │   └── variables.css           # CSS custom properties
│   │
│   └── pages/
│       ├── _app.jsx                # Global app wrapper
│       └── _document.jsx           # HTML document setup
│
├── index.html                      # HTML entry point
├── vite.config.js                  # Vite build configuration
├── package.json                    # Dependencies
├── .env.example                    # Environment template
├── .gitignore
└── README.md                       # Frontend setup guide
```

### 2. Key Components

**Pages** (Full-screen views):
- Login/Register - Authentication pages
- Dashboard - Stats and quick overview
- Kanban - Board view with drag-drop
- TaskList - Table/list view with filters

**Components** (Reusable blocks):
- TaskCard - Display single task
- TaskModal - Create/edit form
- KanbanColumn - Kanban column container
- StatsCard - Display metric
- ProgressChart - Recharts visualization
- Navbar - Navigation
- ProtectedRoute - Auth guard

**API Layer** (`api/` folder):
- Handles all HTTP communication
- Axios client with JWT interceptors
- Separate files for auth, tasks, categories, stats

**State Management** (`store/` folder):
- authStore - User, tokens, login state
- taskStore - Tasks list, filters, search
- categoryStore - Categories list

**Hooks** (`hooks/` folder):
- useAuth() - Login, register, logout
- useTasks() - Fetch, create, update, delete, filter
- useCategories() - Fetch categories

---

## DATABASE DESIGN

### Entity Relationship Diagram (ERD)

```
┌──────────────┐
│    USERS     │
├──────────────┤
│ id (PK)      │ ◄─────────┐
│ name         │           │ (1:N)
│ email (UQ)   │           │
│ password_hash│           │
│ created_at   │           │
└──────────────┘           │
                           │
        ┌────────────────────┴──────────────────┐
        │                                       │
┌───────▼──────────┐                    ┌──────▼──────────┐
│   CATEGORIES     │                    │     TASKS       │
├──────────────────┤                    ├─────────────────┤
│ id (PK)          │                    │ id (PK)         │
│ name             │                    │ title           │
│ user_id (FK)     │◄──────┐            │ deadline        │
│ created_at       │       │ (1:N)      │ status          │
└──────────────────┘       │            │ user_id (FK)    │
                           │            │ category_id(FK) │
                           └────────────┤ created_at      │
                                        │ updated_at      │
                                        └─────────────────┘
```

### Table Schemas

#### **USERS Table**
| Column | Type | Constraints | Purpose |
|--------|------|-------------|---------|
| id | INT | PK, Auto-increment | Unique user ID |
| name | VARCHAR(120) | NOT NULL | User's full name |
| email | VARCHAR(120) | NOT NULL, UNIQUE, INDEX | Login identifier |
| password_hash | VARCHAR(255) | NOT NULL | Hashed password |
| created_at | TIMESTAMP | DEFAULT now() | Registration date |

#### **CATEGORIES Table**
| Column | Type | Constraints | Purpose |
|--------|------|-------------|---------|
| id | INT | PK, Auto-increment | Unique category ID |
| name | VARCHAR(50) | NOT NULL | Category name |
| user_id | INT | FK(users.id), NOT NULL | Owner |
| created_at | TIMESTAMP | DEFAULT now() | Creation date |

#### **TASKS Table**
| Column | Type | Constraints | Purpose |
|--------|------|-------------|---------|
| id | INT | PK, Auto-increment | Unique task ID |
| title | VARCHAR(200) | NOT NULL | Task name |
| deadline | DATE | NULLABLE | Due date |
| status | VARCHAR(20) | NOT NULL, DEFAULT 'Planned' | Planned/In Progress/Done |
| user_id | INT | FK(users.id), NOT NULL | Owner |
| category_id | INT | FK(categories.id), NULLABLE | Task category |
| created_at | TIMESTAMP | DEFAULT now() | Creation date |
| updated_at | TIMESTAMP | DEFAULT now(), auto-update | Last modified date |

---

## API CONTRACT

### 12 Total Endpoints

#### **Authentication (4 endpoints)**

**POST /auth/register**
- Create user account
- Body: `{name, email, password}`
- Response: User object

**POST /auth/login**
- Authenticate and return tokens
- Body: `{email, password}`
- Response: `{access_token, refresh_token, user}`

**POST /auth/refresh**
- Get new access token
- Headers: JWT refresh token
- Response: `{access_token}`

**GET /auth/me**
- Get current user info
- Headers: JWT access token
- Response: User object

#### **Tasks (5 endpoints)**

**GET /tasks**
- List tasks with filters/search/pagination
- Query: `?status=&category_id=&search=&page=`
- Response: `{tasks[], total, pages, current_page}`

**POST /tasks**
- Create task
- Body: `{title, deadline, category_id, status}`
- Response: Task object

**PUT /tasks/:id**
- Update task
- Body: Any field to update
- Response: Task object

**DELETE /tasks/:id**
- Delete task
- Response: `{message}`

**PATCH /tasks/:id/status**
- Update task status (Kanban drag-drop)
- Body: `{status}`
- Response: Task object

#### **Categories (2 endpoints)**

**GET /categories**
- List user's categories
- Response: Category object array

**POST /categories**
- Create category
- Body: `{name}`
- Response: Category object

#### **Stats (2 endpoints)**

**GET /stats**
- Dashboard statistics
- Response: `{total_done, in_progress, overdue, total}`

**GET /stats/activity**
- Daily activity (last 7 days)
- Response: `[{date, count}]`

---

## AUTHENTICATION FLOW

### JWT Token System

**Two Tokens:**
1. **Access Token** (15 minutes)
   - Stored in memory
   - Sent in every API request header: `Authorization: Bearer {token}`
   - Used to access protected routes

2. **Refresh Token** (7 days)
   - Stored in httpOnly cookie (secure, cannot be accessed by JS)
   - Used to get new access token when it expires
   - More secure because JS cannot read it

**Why Two Tokens?**
- Access token is short-lived (reduces damage if stolen)
- Refresh token lets user stay logged in without re-entering password
- httpOnly cookie prevents XSS attacks

### Token Refresh Flow
```
User makes API request
    ↓
Has valid access token? → YES → Send request
    ↓ NO (expired)
Has refresh token? → NO → Redirect to login
    ↓ YES
Send refresh token to /auth/refresh
    ↓
Get new access token
    ↓
Retry original request
```

---

## DATA FLOW DIAGRAMS

### Create Task Flow
```
User clicks "+ New Task"
    ↓
TaskModal opens
    ↓
User fills form (title, deadline, category, status)
    ↓
Click "Save"
    ↓
Validate (client-side)
    ↓
POST /tasks with form data
    ↓
Backend: Validate → Create → Save → Commit
    ↓
Response: 201 Created {task object}
    ↓
Frontend: Add to store → Close modal → Re-render
```

### Kanban Drag-Drop Flow
```
User drags task from "Planned" to "In Progress"
    ↓
dnd-kit detects drop
    ↓
Optimistic update: Update UI immediately
    ↓
PATCH /tasks/:id/status (async)
    ↓
If success: Keep UI updated
    ↓
If error: Revert UI + show error message
```

### Search & Filter Flow
```
User types "project" in search box
    ↓
Debounce 300-500ms (wait for user to stop typing)
    ↓
GET /tasks?search=project&status=In%20Progress&page=1
    ↓
Backend: Filter → Paginate → Return results
    ↓
Frontend: Clear old results → Display new → Update pagination
```

---

## DEPLOYMENT ARCHITECTURE

### Deployment Stack

```
GitHub (main branch push)
    ↓
    ├─→ GitHub Actions CI (backend)
    │   ├─ Run lint
    │   ├─ Run tests
    │   └─ Build
    │       ↓
    │   ✓ All pass → Render auto-deploys
    │
    └─→ Vercel (frontend)
        ├─ Install deps
        ├─ Build React app
        └─ Tests
            ↓
        ✓ All pass → Deploy to CDN
```

### Production URLs
- **Backend API**: `https://momentum-api.render.com`
- **Frontend App**: `https://momentum.vercel.app`
- **Database**: Supabase PostgreSQL

### Monitoring
- UptimeRobot pings backend every 10 minutes (prevents cold starts)
- Vercel auto-scales frontend
- Supabase provides monitoring dashboard

---

## 6-WEEK SPRINT PLAN

### Week 1 - Setup & Auth
**Backend Person:**
- [ ] Flask app factory setup
- [ ] Configuration management (dev/test/prod)
- [ ] User model + database
- [ ] JWT implementation (register, login, refresh endpoints)
- [ ] Basic error handling

**Frontend Person:**
- [ ] React + Vite setup
- [ ] Routing setup (React Router)
- [ ] Login page + form
- [ ] Register page + form
- [ ] Protected route wrapper
- [ ] Mock API (for testing locally)

**Friday Sync:**
- [ ] Define final API contract together
- [ ] Test login/register end-to-end
- [ ] Demo working authentication

---

### Week 2 - Task CRUD
**Backend Person:**
- [ ] Task model + database
- [ ] Category model + database
- [ ] GET /tasks (with pagination)
- [ ] POST /tasks (create)
- [ ] PUT /tasks/:id (edit)
- [ ] DELETE /tasks/:id (delete)
- [ ] Unit tests for all endpoints

**Frontend Person:**
- [ ] Task list page (table/grid view)
- [ ] TaskCard component (display task)
- [ ] TaskModal component (create/edit form)
- [ ] Edit/Delete buttons
- [ ] Handle API responses

**Deliverable:** Full task CRUD working end-to-end

---

### Week 3 - Kanban & Search
**Backend Person:**
- [ ] PATCH /tasks/:id/status endpoint
- [ ] GET /tasks search parameter
- [ ] GET /tasks filter by status
- [ ] GET /tasks filter by category
- [ ] Pagination logic
- [ ] Tests for search/filter/pagination

**Frontend Person:**
- [ ] Kanban board page (3 columns)
- [ ] KanbanColumn component
- [ ] KanbanCard component
- [ ] Drag-and-drop setup (dnd-kit)
- [ ] Status update on drop
- [ ] Filter UI (category, status)
- [ ] Search box (with debounce)

**Deliverable:** Kanban board with drag-drop, search, filters working

---

### Week 4 - Dashboard & Stats
**Backend Person:**
- [ ] GET /stats endpoint (done, in-progress, overdue counts)
- [ ] GET /stats/activity endpoint (last 7 days)
- [ ] Ensure calculations are correct
- [ ] Unit tests

**Frontend Person:**
- [ ] Dashboard page
- [ ] StatsCard components (4 cards)
- [ ] ProgressChart component (Recharts)
- [ ] Quick-add task button
- [ ] Integrate API calls

**Deliverable:** Full app demo with all features working

---

### Week 5 - Deployment
**Backend Person:**
- [ ] Create Supabase project + database
- [ ] Configure environment variables
- [ ] Deploy to Render
- [ ] Set up GitHub Actions CI/CD
- [ ] Configure Render auto-deploy
- [ ] Test production API

**Frontend Person:**
- [ ] Create Vercel project
- [ ] Set environment variables (API_BASE_URL)
- [ ] Configure auto-deploy from GitHub
- [ ] Test Vercel preview URLs
- [ ] Verify API calls work in production

**Deliverable:** Live URLs working (momentum-api.render.com, momentum.vercel.app)

---

### Week 6 - QA & Polish
**Backend Person:**
- [ ] Security audit (JWT validation, input validation)
- [ ] Generate API documentation (Swagger/Postman)
- [ ] Code review + cleanup
- [ ] Final testing

**Frontend Person:**
- [ ] Cross-browser testing
- [ ] Mobile responsive check
- [ ] Error handling & edge cases
- [ ] UI polish + animations
- [ ] Accessibility check

**Friday Sync:**
- [ ] Final demo
- [ ] Retrospective
- [ ] Plan future features

---

## KEY ARCHITECTURAL DECISIONS

1. **Monorepo** - Single GitHub repo for backend + frontend
2. **JWT + httpOnly Cookies** - Balance security and UX
3. **SQLAlchemy ORM** - Prevents SQL injection, type-safe
4. **Supabase** - Managed PostgreSQL, free tier sufficient
5. **Render + Vercel** - Auto-deploy from GitHub, free tier
6. **Tailwind CSS** - Utility-first, responsive by default
7. **dnd-kit** - Modern drag-drop library
8. **Zustand/Context** - Lightweight state management

---

## NEXT STEPS

1. **Backend Developer**: Start Flask setup
2. **Frontend Developer**: Start React + Vite setup
3. **Friday**: Sync on API contract
4. **Monday**: Begin parallel development

Good luck! 🚀
