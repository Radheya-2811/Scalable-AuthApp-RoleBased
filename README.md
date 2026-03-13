# ⚡ TaskFlow API

> A secure, scalable REST API with JWT authentication and role-based access control — built with Python, FastAPI, and SQLite.



## Overview

 a production-ready backend API assignment built for the **Backend Developer Intern** position. It demonstrates:

- User registration and login with **bcrypt** password hashing
- **JWT-based authentication** with token expiry
- **Role-based access control** (user vs. admin)
- Full **CRUD** for a secondary entity (Tasks)
- **API versioning** (`/api/v1/`)
- Automatic **Swagger UI** documentation (via FastAPI)
- A **single-file frontend** (HTML + Fetch API) for testing all features

---

## Tech Stack

| Layer       | Technology                               |
|-------------|------------------------------------------|
| Language    | Python 3.11+                             |
| Framework   | FastAPI                                  |
| Auth        | JWT via `python-jose`, bcrypt via `passlib` |
| Database    | SQLite (via SQLAlchemy ORM)              |
| Validation  | Pydantic v2                              |
| Frontend    | Vanilla HTML + CSS + Fetch API           |
| API Docs    | Swagger UI (auto-generated at `/docs`)   |

---

## Project Structure

```
taskflow/
├── backend/
│   ├── app/
│   │   ├── main.py                    # App entry point, CORS, startup
│   │   ├── core/
│   │   │   ├── config.py              # Settings loaded from .env
│   │   │   └── security.py            # JWT creation/decoding, password hashing
│   │   ├── db/
│   │   │   └── database.py            # SQLAlchemy engine, session, Base
│   │   ├── models/
│   │   │   ├── user.py                # User ORM model
│   │   │   └── task.py                # Task ORM model
│   │   ├── schemas/
│   │   │   ├── auth.py                # Pydantic schemas for auth
│   │   │   └── task.py                # Pydantic schemas for tasks
│   │   └── api/
│   │       ├── deps.py                # Auth dependencies (get_current_user, require_admin)
│   │       └── v1/
│   │           ├── router.py          # Combines all routers under /api/v1
│   │           └── endpoints/
│   │               ├── auth.py        # POST /auth/register, POST /auth/login
│   │               ├── users.py       # GET /users/me
│   │               ├── tasks.py       # CRUD /tasks
│   │               └── admin.py       # Admin-only /admin/* endpoints
│   ├── .env                           # Local config (gitignored)
│   ├── .env.example                   # Template — copy to .env
│   └── requirements.txt
└── frontend/
    └── index.html                     # Single-file SPA
```

---


### Prerequisites

- Python 3.11+
- pip

### 1. Clone the repository

```bash
git clone https://github.com/your-username/taskflow.git
cd taskflow/backend
```

### 2. Create a virtual environment

```bash
python -m venv venv

# macOS / Linux
source venv/bin/activate

# Windows
venv\Scripts\activate
```

### 3. Install dependencies

```bash
pip install -r requirements.txt
```

### 4. Configure environment

```bash
cp .env.example .env
```

Open `.env` and set a strong `SECRET_KEY`:

```bash
# Generate one automatically:
python -c "import secrets; print(secrets.token_hex(32))"
```

### 5. Run the server

```bash
uvicorn app.main:app --reload
```

The API is now running at:

| URL | Description |
|-----|-------------|
| `http://localhost:8000` | Health check |
| `http://localhost:8000/docs` | Swagger UI (interactive docs) |
| `http://localhost:8000/redoc` | ReDoc documentation |

### 6. Open the frontend

```bash
cd ../frontend
python -m http.server 3000
```

Visit `http://localhost:3000` — or just open `frontend/index.html` directly in a browser.

---

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `SECRET_KEY` | *(required)* | JWT signing secret — **change before deploying** |
| `ALGORITHM` | `HS256` | JWT signing algorithm |
| `ACCESS_TOKEN_EXPIRE_MINUTES` | `30` | Token lifetime in minutes |
| `DATABASE_URL` | `sqlite:///./taskflow.db` | Database connection string |

To switch to PostgreSQL in production, update `DATABASE_URL`:

```env
DATABASE_URL=postgresql://user:password@localhost:5432/taskflow
```

---

## API Reference

**Base URL:** `http://localhost:8000/api/v1`



### Auth — Public

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/auth/register` | Register a new user |
| `POST` | `/auth/login` | Login and receive a JWT |

**Register example:**

```bash
curl -X POST http://localhost:8000/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username": "alice", "email": "alice@example.com", "password": "SecurePass1"}'
```

**Login example:**

```bash
curl -X POST http://localhost:8000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username": "alice", "password": "SecurePass1"}'
```

```json
{ "access_token": "eyJ...", "token_type": "bearer" }
```

### Users — Authenticated

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/users/me` | Get current user's profile |

### Tasks — Authenticated

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/tasks` | Create a new task |
| `GET` | `/tasks` | List tasks — supports `?status=` and `?priority=` filters |
| `GET` | `/tasks/{id}` | Get a single task |
| `PATCH` | `/tasks/{id}` | Partially update a task |
| `DELETE` | `/tasks/{id}` | Delete a task |

**Task fields:**

| Field | Type | Values |
|-------|------|--------|
| `title` | string | 1–200 characters |
| `description` | string (optional) | up to 2000 characters |
| `status` | enum | `todo` · `in_progress` · `done` |
| `priority` | enum | `low` · `medium` · `high` |

### Admin — Admin Role Required

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/admin/users` | List all registered users |
| `PATCH` | `/admin/users/{id}/role?role=admin` | Promote or demote a user |
| `PATCH` | `/admin/users/{id}/deactivate` | Deactivate a user account |

**Promote yourself to admin** (first time setup):

```bash
sqlite3 backend/taskflow.db "UPDATE users SET role='admin' WHERE username='alice';"
```

---

## Role-Based Access Control

| Action | `user` | `admin` |
|--------|:------:|:-------:|
| Register / Login | ✅ | ✅ |
| View own profile | ✅ | ✅ |
| Create / edit / delete own tasks | ✅ | ✅ |
| View other users' tasks | ❌ | ✅ |
| List all users | ❌ | ✅ |
| Change user roles | ❌ | ✅ |
| Deactivate users | ❌ | ✅ |

---

## Database Schema

```
users
├── id               INTEGER  PRIMARY KEY
├── username         VARCHAR(50)   UNIQUE  NOT NULL
├── email            VARCHAR(100)  UNIQUE  NOT NULL
├── hashed_password  VARCHAR       NOT NULL
├── role             ENUM(user, admin)   DEFAULT 'user'
├── is_active        BOOLEAN  DEFAULT TRUE
└── created_at       DATETIME  DEFAULT now()

tasks
├── id               INTEGER  PRIMARY KEY
├── title            VARCHAR(200)  NOT NULL
├── description      TEXT
├── status           ENUM(todo, in_progress, done)  DEFAULT 'todo'
├── priority         ENUM(low, medium, high)  DEFAULT 'medium'
├── owner_id         INTEGER  FK → users.id
├── created_at       DATETIME  DEFAULT now()
└── updated_at       DATETIME  ON UPDATE now()
```

The database file (`taskflow.db`) is created automatically on first run — no migrations or setup needed.

---

## Security Practices

- **Password hashing** — bcrypt with cost factor 12. Passwords are never stored in plain text.
- **JWT authentication** — tokens signed with HS256, expire after 30 minutes, and carry the user's ID and role in the payload.
- **Password policy** — minimum 8 characters, at least one uppercase letter, at least one digit (enforced server-side by Pydantic).
- **Input validation** — all request bodies validated via Pydantic v2 before touching the database. Fields are type-checked, length-limited, and pattern-matched.
- **SQL injection prevention** — SQLAlchemy ORM uses parameterized queries exclusively.
- **Ownership checks** — users can only read and modify their own tasks. Admins bypass this restriction.
- **CORS** — configured in `main.py`. Tighten `allow_origins` to your frontend's domain before going to production.

---

## Frontend

The frontend is a single `index.html` file — no build step, no npm, no framework required.

**Features:**

- Register and login with inline error and success messages
- JWT token stored in `localStorage`, sent automatically with every API request
- Dashboard with live task statistics (total, to-do, in progress, done)
- Create, edit, and delete tasks with status and priority selectors
- Filter task list by status
- Admin panel (visible only to admin role) — list all users, promote/demote, deactivate accounts

---

## Scalability

See [`SCALABILITY.md`](./SCALABILITY.md) for the full write-up. Key points:

- **Database** — swap SQLite → PostgreSQL by changing one line in `.env`. SQLAlchemy handles the rest.
- **Horizontal scaling** — stateless JWT means any number of API instances run behind a load balancer with no shared session state.
- **Caching** — add Redis in front of `GET /tasks` to cache per-user results and reduce DB load.
- **Modular structure** — new features (e.g., `projects`, `comments`) are added by creating a file under `api/v1/endpoints/` and one line in `router.py`.
- **API versioning** — already in place at `/api/v1/`. Add `/api/v2/` without breaking existing clients.
- **Containerization** — add a `Dockerfile` + `docker-compose.yml` to run the API, database, and Redis as a single stack.

---

## Roadmap / Optional Enhancements

- [ ] Refresh tokens for extended sessions
- [ ] Email verification on registration
- [ ] Redis caching for task list queries
- [ ] Rate limiting with `slowapi`
- [ ] Pagination metadata in list responses
- [ ] Docker + docker-compose setup
- [ ] Pytest test suite with `httpx`
- [ ] CI/CD pipeline via GitHub Actions

---

## License

MIT
