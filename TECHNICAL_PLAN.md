# Momentum - Technical Architecture Plan
## Version 2.0 | June 2026

---

## TABLE OF CONTENTS
1. [System Overview](#system-overview)
2. [Backend Architecture](#backend-architecture)
3. [Frontend Architecture](#frontend-architecture)
4. [Database Design](#database-design)
5. [API Contract](#api-contract)
6. [Authentication Flow](#authentication-flow)
7. [Real-time Synchronization](#real-time-synchronization)
8. [Data Flow Diagrams](#data-flow-diagrams)
9. [Deployment Architecture](#deployment-architecture)
10. [6-Week Sprint Plan](#6-week-sprint-plan)

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
│  │  - Real-time Sync    │  WS     │  - WebSocket Support  │ │
│  └──────────────────────┘         └───────────────────────┘ │
│           ▲                                 ▲                 │
│           │                                 │                 │
│           │                                 │                 │
│           └─────────────────┬───────────────┘                 │
│                             │                                 │
│                    ┌────────▼────────┐                       │
│                    │   SUPABASE DB   │                       │
│                    │  (PostgreSQL)   │                       │
│                    │  Real-time Sub  │                       │
│                    └─────────────────┘                       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Tech Stack Summary
- **Backend**: Python 3.12 + Flask + Flask-SocketIO
- **Frontend**: React 18 + Vite + Socket.IO Client
- **Database**: Supabase (PostgreSQL) + Real-time Subscriptions
- **Authentication**: JWT (JSON Web Tokens) + Password Reset via Email
- **Real-time**: WebSocket (Socket.IO) + Supabase Real-time
- **Email**: SendGrid for password reset emails
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
│   │   ├── category.py             # Category model
│   │   ├── team.py                 # Team model (NEW)
│   │   ├── team_member.py          # Team membership model (NEW)
│   │   ├── notification.py         # Notification model (NEW)
│   │   └── password_reset.py       # Password reset tokens (NEW)
│   │
│   ├── routes/
│   │   ├── __init__.py
│   │   ├── auth.py                 # Login, Register, Token Refresh, Password Reset (UPDATED)
│   │   ├── tasks.py                # Task CRUD, Status Update, Search, Filter
│   │   ├── categories.py           # Category CRUD
│   │   ├── stats.py                # Dashboard stats, Activity data
│   │   ├── teams.py                # Team management (NEW)
│   │   ├── notifications.py        # Notification endpoints (NEW)
│   │   └── settings.py             # User settings (NEW)
│   │
│   ├── middleware/
│   │   ├── __init__.py
│   │   ├── auth.py                 # JWT validation decorator
│   │   └── error_handler.py        # Global error handling (NEW)
│   │
│   ├── services/
│   │   ├── __init__.py
│   │   ├── email_service.py        # SendGrid integration (NEW)
│   │   ├── notification_service.py # Real-time notifications (NEW)
│   │   └── team_service.py         # Team operations (NEW)
│   │
│   ├── websocket/
│   │   ├── __init__.py
│   │   ├── handlers.py             # WebSocket event handlers (NEW)
│   │   └── events.py               # Event definitions (NEW)
│   │
│   └── tests/
│       ├── __init__.py
│       ├── conftest.py             # Pytest fixtures
│       ├── test_auth.py            # Auth endpoint tests
│       ├── test_tasks.py           # Task endpoint tests
│       ├── test_categories.py      # Category endpoint tests
│       ├── test_teams.py           # Team tests (NEW)
│       └── test_notifications.py   # Notification tests (NEW)
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
  - Configure extensions (SQLAlchemy, JWT, CORS, SocketIO)
  - Register blueprints (routes)
  - Create database tables
  - Set up error handlers
  - Initialize WebSocket namespace

#### **Configuration Management** (`config.py`)
- **Three environments**:
  1. **Development**: SQLite, Debug=True, CORS permissive
  2. **Production**: PostgreSQL (Supabase), Debug=False, CORS restricted
  3. **Testing**: In-memory SQLite, isolated tests
- **Manages**:
  - Database connection strings
  - JWT secret and expiration times
  - CORS allowed origins
  - SendGrid API key
  - WebSocket configuration
  - Environment-specific flags

#### **Models (Data Layer)**
Located in `app/models/`, define database schema:

1. **User Model** (`user.py`)
   - Fields: id, name, email, password_hash, created_at, updated_at, is_active, onboarded
   - Methods: set_password(), check_password(), to_dict()
   - Relationships: owns Tasks, Categories, Teams, Notifications

2. **Task Model** (`task.py`)
   - Fields: id, title, description, deadline, status, category_id, user_id, team_id, created_at, updated_at
   - Status options: "Planned", "In Progress", "Done"
   - Methods: to_dict()
   - Relationships: belongs to User, Category, Team (optional)

3. **Category Model** (`category.py`)
   - Fields: id, name, user_id, created_at
   - Constraint: user_id + name must be unique
   - Methods: to_dict()
   - Relationships: belongs to User, has multiple Tasks

4. **Team Model** (`team.py`) - NEW
   - Fields: id, name, description, owner_id, created_at, updated_at
   - Methods: to_dict()
   - Relationships: belongs to User (owner), has TeamMembers

5. **TeamMember Model** (`team_member.py`) - NEW
   - Fields: id, team_id, user_id, role (admin/member), joined_at
   - Methods: to_dict()
   - Relationships: belongs to Team and User

6. **Notification Model** (`notification.py`) - NEW
   - Fields: id, user_id, type, title, message, read, created_at, related_task_id
   - Types: task_assigned, task_updated, team_invite, mention
   - Methods: to_dict()

7. **PasswordReset Model** (`password_reset.py`) - NEW
   - Fields: id, user_id, token (unique), expires_at, used_at
   - Methods: to_dict()

#### **Routes (API Endpoints)** - Located in `app/routes/`

1. **auth.py**: Authentication routes (UPDATED)
   - Register, Login, Token Refresh, Get Current User, Password Reset, Verify Reset Token

2. **tasks.py**: Task management routes
   - CRUD operations, Status changes, Search, Filter, Paginate

3. **categories.py**: Category management routes
   - List, Create, Edit, Delete

4. **stats.py**: Dashboard statistics routes
   - Dashboard stats, Activity data

5. **teams.py**: Team management routes (NEW)
   - Create team, List teams, Add member, Remove member, Update team

6. **notifications.py**: Notification routes (NEW)
   - Get notifications, Mark as read, Delete notification

7. **settings.py**: User settings routes (NEW)
   - Get settings, Update profile, Update preferences, Onboarding status

#### **Middleware** (`app/middleware/`)
- **JWT Validation Decorator**: Checks token validity, extracts user ID
- **Error Handler**: Global error handling with standardized responses

#### **Services** (`app/services/`) - NEW
- **EmailService**: SendGrid integration for password reset emails
- **NotificationService**: Real-time notification management
- **TeamService**: Team operations and member management

#### **WebSocket** (`app/websocket/`) - NEW
- **Handlers**: Task updates, team notifications, real-time collaboration
- **Events**: task_update, task_status_change, notification_received, user_online/offline

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
├── WebSocket broadcast (if applicable)
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
│   │   ├── LandingPage.jsx         # Marketing page (NEW)
│   │   ├── Onboarding.jsx          # Onboarding wizard (NEW)
│   │   ├── Dashboard.jsx           # Main dashboard with stats
│   │   ├── Kanban.jsx              # Kanban board view
│   │   ├── TaskList.jsx            # List view of tasks
│   │   ├── TeamManagement.jsx      # Team management page (NEW)
│   │   ├── Settings.jsx            # User settings (NEW)
│   │   ├── Login.jsx               # Login page
│   │   ├── Register.jsx            # Registration page
│   │   ├── PasswordReset.jsx       # Password reset page (NEW)
│   │   ├── ResetForm.jsx           # Password reset form (NEW)
│   │   └── NotFound.jsx            # 404 page
│   │
│   ├── components/
│   │   ├── Layout/
│   │   │   ├── Navbar.jsx          # Navigation bar (UPDATED with notifications)
│   │   │   ├── Sidebar.jsx         # Side menu (UPDATED with teams)
│   │   │   ├── Footer.jsx          # Footer
│   │   │   └── NotificationCenter.jsx # Notification panel (NEW)
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
│   │   ├── Team/
│   │   │   ├── TeamCard.jsx        # Team display (NEW)
│   │   │   ├── TeamInviteModal.jsx # Invite modal (NEW)
│   │   │   ├── MemberList.jsx      # Team members list (NEW)
│   │   │   └── RoleSelector.jsx    # Role assignment (NEW)
│   │   │
│   │   ├── Auth/
│   │   │   ├── ProtectedRoute.jsx  # Wrapper for protected pages
│   │   │   ├── AuthGuard.jsx       # Redirect if not authenticated
│   │   │   └── OnboardingGuard.jsx # Redirect if not onboarded (NEW)
│   │   │
│   │   ├── Common/
│   │   │   ├── Loading.jsx         # Loading spinner
│   │   │   ├── Error.jsx           # Error message
│   │   │   ├── EmptyState.jsx      # Empty state message
│   │   │   └── Toast.jsx           # Toast notifications (NEW)
│   │   │
│   │   ├── Onboarding/
│   │   │   ├── OnboardingWizard.jsx # Multi-step wizard (NEW)
│   │   │   ├── Step1Profile.jsx    # Name and profile (NEW)
│   │   │   ├── Step2Categories.jsx # Create initial categories (NEW)
│   │   │   ├── Step3Sample.jsx     # Create sample tasks (NEW)
│   │   │   └── Step4Complete.jsx   # Completion screen (NEW)
│   │   │
│   │   └── Filter/
│   │       ├── StatusFilter.jsx    # Filter by status
│   │       ├── CategoryFilter.jsx  # Filter by category
│   │       ├── TeamFilter.jsx      # Filter by team (NEW)
│   │       └── SearchBox.jsx       # Search tasks
│   │
│   ├── api/
│   │   ├── client.js               # Axios HTTP client
│   │   ├── auth.js                 # Authentication API calls (UPDATED)
│   │   ├── tasks.js                # Task API calls
│   │   ├── categories.js           # Category API calls
│   │   ├── stats.js                # Stats API calls
│   │   ├── teams.js                # Team API calls (NEW)
│   │   ├── notifications.js        # Notification API calls (NEW)
│   │   ├── settings.js             # Settings API calls (NEW)
│   │   └── websocket.js            # WebSocket connection (NEW)
│   │
│   ├── store/
│   │   ├── authStore.js            # Auth state (Zustand)
│   │   ├── taskStore.js            # Task state
│   │   ├── categoryStore.js        # Category state
│   │   ├── teamStore.js            # Team state (NEW)
│   │   ├── notificationStore.js    # Notification state (NEW)
│   │   └── settingsStore.js        # Settings state (NEW)
│   │
│   ├── hooks/
│   │   ├── useAuth.js              # Auth logic hook (UPDATED)
│   │   ├── useTasks.js             # Task management hook
│   │   ├── useCategories.js        # Category management hook
│   │   ├── useTeams.js             # Team management hook (NEW)
│   │   ├── useNotifications.js     # Notification management hook (NEW)
│   │   ├── useWebSocket.js         # WebSocket connection hook (NEW)
│   │   └── useOnboarding.js        # Onboarding logic hook (NEW)
│   │
│   ├── utils/
│   │   ├── dateUtils.js            # Date formatting helpers
│   │   ├── tokenUtils.js           # JWT token handling
│   │   ├── validators.js           # Form validation
│   │   ├── emailValidator.js       # Email validation (NEW)
│   │   └── passwordUtils.js        # Password strength checker (NEW)
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
- Landing - Public marketing page
- Onboarding - Multi-step wizard after registration
- Login/Register - Authentication pages
- Password Reset - Password recovery
- Dashboard - Stats and quick overview
- Kanban - Board view with drag-drop
- TaskList - Table/list view with filters
- TeamManagement - Create/invite/manage teams
- Settings - User preferences and profile

**Components** (Reusable blocks):
- TaskCard - Display single task
- TaskModal - Create/edit form
- KanbanColumn - Kanban column container
- StatsCard - Display metric
- ProgressChart - Recharts visualization
- Navbar - Navigation (with notifications)
- NotificationCenter - Real-time notifications
- OnboardingWizard - Multi-step setup
- TeamCard - Team display
- ProtectedRoute - Auth guard
- Toast - Notification messages

**API Layer** (`api/` folder):
- Handles all HTTP communication
- Axios client with JWT interceptors
- WebSocket connection management
- Separate files for auth, tasks, teams, notifications, settings

**State Management** (`store/` folder):
- authStore - User, tokens, login state
- taskStore - Tasks list, filters, search
- categoryStore - Categories list
- teamStore - Teams, members, roles
- notificationStore - Notifications, unread count
- settingsStore - User preferences

**Hooks** (`hooks/` folder):
- useAuth() - Login, register, logout, password reset
- useTasks() - Fetch, create, update, delete, filter
- useCategories() - Fetch categories
- useTeams() - Team operations
- useNotifications() - Notification management
- useWebSocket() - Real-time connection
- useOnboarding() - Wizard progress

---

## DATABASE DESIGN

### Entity Relationship Diagram (ERD)

```
┌──────────────┐
│    USERS     │
├──────────────┤
│ id (PK)      │ ◄─────────────────────┐
│ name         │                       │ (1:N)
│ email (UQ)   │                       │
│ password_hash│                       │
│ is_active    │                       │
│ onboarded    │                       │
│ created_at   │                       │
└──────────────┘                       │
       ▲                               │
       │                               │
   ┌───┴─────────────────────────────────┴─────────────────┐
   │                                                        │
┌──▼──────────────┐    ┌──────────────┐    ┌──────────────┐
│  CATEGORIES     │    │   TASKS      │    │  TEAMS       │
├─────────────────┤    ├──────────────┤    ├──────────────┤
│ id (PK)         │    │ id (PK)      │    │ id (PK)      │
│ name            │◄───│ category_id  │    │ name         │
│ user_id (FK)    │    │ title        │    │ owner_id(FK) │
│ created_at      │    │ deadline     │    │ created_at   │
└────────────────┘    │ status       │    └──────────────┘
                      │ user_id (FK) │            ▲
                      │ team_id(FK)  │            │ (1:N)
                      │ created_at   │            │
                      │ updated_at   │    ┌───────┴──────────┐
                      └──────────────┘    │                  │
                                    ┌─────▼──────────────┐
                                    │  TEAM_MEMBERS      │
                                    ├────────────────────┤
                                    │ id (PK)            │
                                    │ team_id (FK)       │
                                    │ user_id (FK)       │
                                    │ role (admin/member)│
                                    │ joined_at          │
                                    └────────────────────┘

┌─────────────────────────┐
│   NOTIFICATIONS         │
├─────────────────────────┤
│ id (PK)                 │
│ user_id (FK) ───────┐   │
│ type                │   │
│ title               │   │
│ message             │   │
│ read                │   │
│ related_task_id(FK) │   │
│ created_at          │   │
└─────────────────────────┘

┌─────────────────────────┐
│  PASSWORD_RESET         │
├─────────────────────────┤
│ id (PK)                 │
│ user_id (FK) ───────┐   │
│ token (UNIQUE)      │   │
│ expires_at          │   │
│ used_at             │   │
│ created_at          │   │
└─────────────────────────┘
```

### Table Schemas

#### **USERS Table** (UPDATED)
| Column | Type | Constraints | Purpose |
|--------|------|-------------|---------|
| id | INT | PK, Auto-increment | Unique user ID |
| name | VARCHAR(120) | NOT NULL | User's full name |
| email | VARCHAR(120) | NOT NULL, UNIQUE, INDEX | Login identifier |
| password_hash | VARCHAR(255) | NOT NULL | Hashed password |
| is_active | BOOLEAN | DEFAULT true | Account status |
| onboarded | BOOLEAN | DEFAULT false | Onboarding status |
| created_at | TIMESTAMP | DEFAULT now() | Registration date |
| updated_at | TIMESTAMP | DEFAULT now(), auto-update | Last modified |

#### **CATEGORIES Table**
| Column | Type | Constraints | Purpose |
|--------|------|-------------|---------|
| id | INT | PK, Auto-increment | Unique category ID |
| name | VARCHAR(50) | NOT NULL | Category name |
| user_id | INT | FK(users.id), NOT NULL | Owner |
| created_at | TIMESTAMP | DEFAULT now() | Creation date |

#### **TASKS Table** (UPDATED)
| Column | Type | Constraints | Purpose |
|--------|------|-------------|---------|
| id | INT | PK, Auto-increment | Unique task ID |
| title | VARCHAR(200) | NOT NULL | Task name |
| description | TEXT | NULLABLE | Task details |
| deadline | DATE | NULLABLE | Due date |
| status | VARCHAR(20) | NOT NULL, DEFAULT 'Planned' | Planned/In Progress/Done |
| user_id | INT | FK(users.id), NOT NULL | Owner |
| category_id | INT | FK(categories.id), NULLABLE | Task category |
| team_id | INT | FK(teams.id), NULLABLE | Team context |
| created_at | TIMESTAMP | DEFAULT now() | Creation date |
| updated_at | TIMESTAMP | DEFAULT now(), auto-update | Last modified |

#### **TEAMS Table** (NEW)
| Column | Type | Constraints | Purpose |
|--------|------|-------------|---------|
| id | INT | PK, Auto-increment | Unique team ID |
| name | VARCHAR(100) | NOT NULL | Team name |
| description | TEXT | NULLABLE | Team description |
| owner_id | INT | FK(users.id), NOT NULL | Team owner |
| created_at | TIMESTAMP | DEFAULT now() | Creation date |
| updated_at | TIMESTAMP | DEFAULT now(), auto-update | Last modified |

#### **TEAM_MEMBERS Table** (NEW)
| Column | Type | Constraints | Purpose |
|--------|------|-------------|---------|
| id | INT | PK, Auto-increment | Unique membership ID |
| team_id | INT | FK(teams.id), NOT NULL | Team reference |
| user_id | INT | FK(users.id), NOT NULL | User reference |
| role | VARCHAR(20) | DEFAULT 'member' | admin or member |
| joined_at | TIMESTAMP | DEFAULT now() | Join date |

#### **NOTIFICATIONS Table** (NEW)
| Column | Type | Constraints | Purpose |
|--------|------|-------------|---------|
| id | INT | PK, Auto-increment | Unique notification ID |
| user_id | INT | FK(users.id), NOT NULL | Recipient |
| type | VARCHAR(50) | NOT NULL | task_assigned, task_updated, team_invite, mention |
| title | VARCHAR(200) | NOT NULL | Notification title |
| message | TEXT | NOT NULL | Notification message |
| read | BOOLEAN | DEFAULT false | Read status |
| related_task_id | INT | FK(tasks.id), NULLABLE | Associated task |
| created_at | TIMESTAMP | DEFAULT now() | Creation date |

#### **PASSWORD_RESET Table** (NEW)
| Column | Type | Constraints | Purpose |
|--------|------|-------------|---------|
| id | INT | PK, Auto-increment | Unique reset ID |
| user_id | INT | FK(users.id), NOT NULL | User requesting reset |
| token | VARCHAR(255) | UNIQUE, NOT NULL | Reset token |
| expires_at | TIMESTAMP | NOT NULL | Token expiration (24h) |
| used_at | TIMESTAMP | NULLABLE | When token was used |
| created_at | TIMESTAMP | DEFAULT now() | Creation date |

---

## API CONTRACT

### 20+ Total Endpoints

#### **Authentication (6 endpoints)** - UPDATED

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

**POST /auth/password-reset** (NEW)
- Request password reset email
- Body: `{email}`
- Response: `{message: "Reset email sent"}`

**POST /auth/reset-password/:token** (NEW)
- Reset password with token
- Body: `{password}`
- Response: `{message: "Password reset successful"}`

#### **Tasks (5 endpoints)**

**GET /tasks**
- List tasks with filters/search/pagination
- Query: `?status=&category_id=&team_id=&search=&page=`
- Response: `{tasks[], total, pages, current_page}`

**POST /tasks**
- Create task
- Body: `{title, description, deadline, category_id, team_id, status}`
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

#### **Categories (4 endpoints)** - UPDATED

**GET /categories**
- List user's categories
- Response: Category object array

**POST /categories**
- Create category
- Body: `{name}`
- Response: Category object

**PUT /categories/:id** (NEW)
- Update category
- Body: `{name}`
- Response: Category object

**DELETE /categories/:id** (NEW)
- Delete category
- Response: `{message}`

#### **Stats (2 endpoints)**

**GET /stats**
- Dashboard statistics
- Response: `{total_done, in_progress, overdue, total}`

**GET /stats/activity**
- Daily activity (last 7 days)
- Response: `[{date, count}]`

#### **Teams (5 endpoints)** - NEW

**GET /teams**
- List user's teams (owned + member)
- Response: Team object array

**POST /teams**
- Create team
- Body: `{name, description}`
- Response: Team object

**POST /teams/:id/members**
- Add member to team
- Body: `{email, role}`
- Response: TeamMember object

**GET /teams/:id/members**
- List team members
- Response: TeamMember object array

**DELETE /teams/:id/members/:user_id** (NEW)
- Remove team member
- Response: `{message}`

#### **Notifications (3 endpoints)** - NEW

**GET /notifications**
- List user notifications
- Query: `?unread=true&limit=20`
- Response: Notification object array

**PATCH /notifications/:id/read**
- Mark notification as read
- Response: Notification object

**DELETE /notifications/:id**
- Delete notification
- Response: `{message}`

#### **Settings (3 endpoints)** - NEW

**GET /settings**
- Get user settings
- Response: User settings object

**PUT /settings/profile**
- Update profile
- Body: `{name, email}`
- Response: User object

**PUT /settings/onboarding**
- Mark onboarding as complete
- Body: `{}`
- Response: `{message: "Onboarding completed"}`

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

### Password Reset Flow (NEW)
```
User clicks "Forgot Password"
    ↓
User enters email
    ↓
POST /auth/password-reset
    ↓
Backend: Generate token → Send email via SendGrid
    ↓
Email contains link: /reset-password/:token
    ↓
User clicks link
    ↓
Frontend: Validate token → Show reset form
    ↓
User enters new password
    ↓
POST /auth/reset-password/:token
    ↓
Backend: Validate token → Hash password → Update user → Mark token used
    ↓
Redirect to login
```

### Onboarding Flow (NEW)
```
User completes registration
    ↓
Redirect to onboarding wizard
    ↓
Step 1: Complete profile (name, photo)
    ↓
Step 2: Create initial categories
    ↓
Step 3: Create sample tasks
    ↓
Step 4: Completion - Mark onboarded=true
    ↓
Redirect to dashboard
```

---

## REAL-TIME SYNCHRONIZATION

### WebSocket Events (NEW)

**Server → Client Events:**
```
task_created: {task object}
  - Broadcast when new task created in team

task_updated: {task object, changed_fields}
  - Broadcast when task updated

task_status_changed: {task_id, status, user}
  - Broadcast when task status changes (Kanban)

notification_received: {notification object}
  - Send when user receives notification

user_online: {user_id}
  - Broadcast when user comes online

user_offline: {user_id}
  - Broadcast when user goes offline

dashboard_updated: {stats object}
  - Send updated stats to dashboard
```

**Client → Server Events:**
```
connect
  - Establish connection

disconnect
  - Close connection

join_task: {task_id}
  - Join real-time updates for task

leave_task: {task_id}
  - Leave real-time updates for task

join_team: {team_id}
  - Join team real-time channel

leave_team: {team_id}
  - Leave team real-time channel

typing_indicator: {task_id}
  - Show user is typing in task
```

### Real-time Dashboard Updates (NEW)
- Dashboard auto-refreshes when tasks change
- Statistics update in real-time
- New notifications appear instantly
- Team member status updates live

### Supabase Real-time Fallback
- If WebSocket unavailable, use Supabase real-time subscriptions
- Automatic reconnection logic
- Offline queue for pending changes

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
Broadcast: task_created via WebSocket
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
Broadcast: task_status_changed via WebSocket
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

### Team Collaboration Flow (NEW)
```
User 1 creates task in Team
    ↓
Backend: Create → Broadcast task_created to team members
    ↓
User 2, 3, 4 receive WebSocket event
    ↓
Frontend: Add task to store → Update UI
    ↓
Team members see new task in real-time
```

### Notification Flow (NEW)
```
Event triggered (task assigned, team invite, etc.)
    ↓
Backend: Create notification → Save to DB → Send WebSocket
    ↓
Frontend: Receive notification_received event
    ↓
Show toast notification
    ↓
Add to notification center
    ↓
Update unread count
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
- **WebSocket**: `wss://momentum-api.render.com/socket.io`

### Environment Variables

**Backend (.env.production):**
```
FLASK_ENV=production
DATABASE_URL=postgresql://user:pass@db.supabase.com/momentum
JWT_SECRET_KEY=your-secret-key
JWT_ACCESS_TOKEN_EXPIRES=900  # 15 minutes
JWT_REFRESH_TOKEN_EXPIRES=604800  # 7 days
SENDGRID_API_KEY=your-sendgrid-key
SENDGRID_FROM_EMAIL=noreply@momentum.app
CORS_ORIGINS=https://momentum.vercel.app
SOCKETIO_MESSAGE_QUEUE=redis://redis-url
```

**Frontend (.env.production):**
```
VITE_API_BASE_URL=https://momentum-api.render.com
VITE_WEBSOCKET_URL=wss://momentum-api.render.com/socket.io
```

### Monitoring
- UptimeRobot pings backend every 10 minutes (prevents cold starts)
- Vercel auto-scales frontend
- Supabase provides monitoring dashboard
- WebSocket connection health checks

---

## 6-WEEK SPRINT PLAN

### Week 1 - Setup & Auth + Password Reset
**Backend Person:**
- [ ] Flask app factory setup
- [ ] Configuration management (dev/test/prod)
- [ ] User model + database (with onboarded flag)
- [ ] JWT implementation (register, login, refresh endpoints)
- [ ] Password reset model + endpoints
- [ ] SendGrid integration
- [ ] Basic error handling
- [ ] WebSocket setup (Flask-SocketIO)

**Frontend Person:**
- [ ] React + Vite setup
- [ ] Routing setup (React Router)
- [ ] Landing page
- [ ] Login page + form
- [ ] Register page + form
- [ ] Password reset request page
- [ ] Password reset form (with token validation)
- [ ] Protected route wrapper
- [ ] Mock API (for testing locally)

**Friday Sync:**
- [ ] Define final API contract together
- [ ] Test login/register/password-reset end-to-end
- [ ] Demo working authentication

---

### Week 2 - Onboarding + Task CRUD
**Backend Person:**
- [ ] Task model + database
- [ ] Category model + database
- [ ] Onboarding endpoints (mark complete)
- [ ] GET /tasks (with pagination)
- [ ] POST /tasks (create)
- [ ] PUT /tasks/:id (edit)
- [ ] DELETE /tasks/:id (delete)
- [ ] Unit tests for all endpoints

**Frontend Person:**
- [ ] Onboarding wizard (multi-step)
- [ ] Step 1: Profile completion
- [ ] Step 2: Create categories
- [ ] Step 3: Create sample tasks
- [ ] Step 4: Completion screen
- [ ] Task list page (table/grid view)
- [ ] TaskCard component (display task)
- [ ] TaskModal component (create/edit form)
- [ ] Edit/Delete buttons

**Deliverable:** Registration → Onboarding → Task CRUD working

---

### Week 3 - Kanban & Search + Notifications
**Backend Person:**
- [ ] Notification model + database
- [ ] PATCH /tasks/:id/status endpoint
- [ ] GET /tasks search parameter
- [ ] GET /tasks filter by status
- [ ] GET /tasks filter by category
- [ ] GET /notifications endpoint
- [ ] PATCH /notifications/:id/read endpoint
- [ ] WebSocket handlers (task_created, task_updated, task_status_changed)
- [ ] Tests for search/filter/pagination/notifications

**Frontend Person:**
- [ ] Kanban board page (3 columns)
- [ ] KanbanColumn component
- [ ] KanbanCard component
- [ ] Drag-and-drop setup (dnd-kit)
- [ ] Status update on drop (with optimistic UI)
- [ ] Filter UI (category, status)
- [ ] Search box (with debounce)
- [ ] Notification Center component
- [ ] Toast notifications
- [ ] WebSocket connection hook
- [ ] Real-time task updates

**Deliverable:** Kanban board with drag-drop, search, real-time notifications

---

### Week 4 - Dashboard + Stats + Teams
**Backend Person:**
- [ ] Team model + database
- [ ] TeamMember model + database
- [ ] Team routes (create, list, add member, remove member)
- [ ] GET /stats endpoint (done, in-progress, overdue counts)
- [ ] GET /stats/activity endpoint (last 7 days)
- [ ] Ensure calculations are correct
- [ ] Unit tests

**Frontend Person:**
- [ ] Dashboard page
- [ ] StatsCard components (4 cards)
- [ ] ProgressChart component (Recharts)
- [ ] Quick-add task button
- [ ] Team Management page
- [ ] Team creation modal
- [ ] Team invite modal
- [ ] Member list with roles
- [ ] Real-time dashboard refresh

**Deliverable:** Full app demo with teams, stats, real-time dashboard

---

### Week 5 - Deployment + Settings
**Backend Person:**
- [ ] Settings routes (profile, onboarding status)
- [ ] Create Supabase project + database
- [ ] Configure environment variables
- [ ] Deploy to Render
- [ ] Set up GitHub Actions CI/CD
- [ ] Configure Render auto-deploy
- [ ] Test production API
- [ ] WebSocket production config

**Frontend Person:**
- [ ] Settings page (profile, preferences)
- [ ] Create Vercel project
- [ ] Set environment variables (API_BASE_URL, WEBSOCKET_URL)
- [ ] Configure auto-deploy from GitHub
- [ ] Test Vercel preview URLs
- [ ] Verify API calls work in production
- [ ] Test WebSocket in production

**Deliverable:** Live URLs working (momentum-api.render.com, momentum.vercel.app)

---

### Week 6 - QA & Polish
**Backend Person:**
- [ ] Security audit (JWT validation, input validation)
- [ ] Generate API documentation (Swagger/Postman)
- [ ] Code review + cleanup
- [ ] Final testing
- [ ] WebSocket stress testing

**Frontend Person:**
- [ ] Cross-browser testing
- [ ] Mobile responsive check
- [ ] Error handling & edge cases
- [ ] UI polish + animations
- [ ] Accessibility check
- [ ] WebSocket reconnection testing

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
6. **WebSocket (Socket.IO)** - Real-time collaboration + notifications
7. **SendGrid** - Reliable email service for password reset
8. **Tailwind CSS** - Utility-first, responsive by default
9. **dnd-kit** - Modern drag-drop library
10. **Zustand** - Lightweight state management
11. **Multi-step Onboarding** - Improved user activation

---

## NEXT STEPS

1. **Backend Developer**: Start Flask setup with JWT + WebSocket
2. **Frontend Developer**: Start React + Vite setup with routing
3. **Friday**: Sync on API contract
4. **Monday**: Begin parallel development

Good luck! 🚀
