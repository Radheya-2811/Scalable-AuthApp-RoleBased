# ⚡ TaskFlow API

A production-ready REST API built with **FastAPI + SQLite + JWT**, featuring role-based access control and a full-featured vanilla JS frontend.

---

## Tech Stack

| Layer    | Technology |
|----------|------------|
| Backend  | Python 3.11+, FastAPI |
| Auth     | JWT (`python-jose`), bcrypt (`passlib`) |
| Database | SQLite via SQLAlchemy ORM |
| Frontend | Vanilla HTML + CSS + Fetch API |
| Docs     | Swagger UI (auto-generated at `/docs`) |

---

## Project Structure

```
taskflow/
├── backend/
│   ├── app/
│   │   ├── main.py               # FastAPI app, CORS, startup
│   │   ├── core/
│   │   │   ├── config.py         # Settings (env vars)
│   │   │   └── security.py       # JWT + password hashing
│   │   ├── db/
│   │   │   └── database.py       # SQLAlchemy engine + session
│   │   ├── models/
│   │   │   ├── user.py           # User ORM model
│   │   │   └── task.py           # Task ORM model
│   │   ├── schemas/
│   │   │   ├── auth.py           # Pydantic request/response schemas
│   │   │   └── task.py
│   │   └── api/
│   │       ├── deps.py           # Auth dependencies
│   │       └── v1/
│   │           ├── router.py     # Combines all routers
│   │           └── endpoints/
│   │               ├── auth.py   # /auth/register, /auth/login
│   │               ├── users.py  # /users/me
│   │               ├── tasks.py  # /tasks CRUD
│   │               └── admin.py  # /admin/* (admin only)
│   ├── .env
│   └── requirements.txt
└── frontend/
    └── index.html                # Single-file SPA
```

---

## Quick Start

### 1. Clone & install

```bash
git clone <your-repo-url>
cd taskflow/backend

python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate

pip install -r requirements.txt
```

### 2. Configure environment

```bash
cp .env .env.local
# Edit .env — change SECRET_KEY before deploying!
```

### 3. Run the API

```bash
uvicorn app.main:app --reload
# API running at http://localhost:8000
# Swagger UI at http://localhost:8000/docs
```

### 4. Open the frontend

Open `frontend/index.html` in your browser — or serve it:

```bash
cd ../frontend
python -m http.server 3000
# Visit http://localhost:3000
```

---

## API Reference

### Base URL: `http://localhost:8000/api/v1`

#### Auth (public)

| Method | Endpoint         | Description         |
|--------|-----------------|---------------------|
| POST   | `/auth/register` | Register new user   |
| POST   | `/auth/login`    | Login → get JWT     |

#### Users (authenticated)

| Method | Endpoint     | Description        |
|--------|--------------|--------------------|
| GET    | `/users/me`  | Get own profile    |

#### Tasks (authenticated)

| Method | Endpoint        | Description                          |
|--------|-----------------|--------------------------------------|
| POST   | `/tasks`        | Create task                          |
| GET    | `/tasks`        | List tasks (own; admin sees all)     |
| GET    | `/tasks/{id}`   | Get single task                      |
| PATCH  | `/tasks/{id}`   | Partial update                       |
| DELETE | `/tasks/{id}`   | Delete task                          |

#### Admin only

| Method | Endpoint                        | Description          |
|--------|---------------------------------|----------------------|
| GET    | `/admin/users`                  | List all users       |
| PATCH  | `/admin/users/{id}/role`        | Change user role     |
| PATCH  | `/admin/users/{id}/deactivate`  | Deactivate user      |

---

## Role-Based Access

| Action               | `user` | `admin` |
|----------------------|--------|---------|
| Register / Login     | ✅     | ✅      |
| CRUD own tasks       | ✅     | ✅      |
| View all users' tasks| ❌     | ✅      |
| Manage user roles    | ❌     | ✅      |
| Deactivate users     | ❌     | ✅      |

**Make yourself admin** — after registering, use the Swagger UI at `/docs` or SQLite directly:

```bash
sqlite3 backend/taskflow.db "UPDATE users SET role='admin' WHERE username='yourname';"
```

---

## Security Practices

- Passwords hashed with **bcrypt** (cost factor 12) — never stored in plain text
- JWT tokens signed with HS256 — expire after 30 minutes
- `Authorization: Bearer <token>` header required on all protected routes
- Input validation via **Pydantic v2** — all fields typed, length-limited, pattern-matched
- Password policy enforced: min 8 chars, 1 uppercase, 1 digit
- CORS configurable per environment
- SQL injection impossible — SQLAlchemy ORM with parameterized queries

---

## Database Schema

```
users
├── id           INTEGER PK
├── username     VARCHAR(50) UNIQUE
├── email        VARCHAR(100) UNIQUE
├── hashed_password VARCHAR
├── role         ENUM(user, admin)
├── is_active    BOOLEAN
└── created_at   DATETIME

tasks
├── id           INTEGER PK
├── title        VARCHAR(200)
├── description  TEXT
├── status       ENUM(todo, in_progress, done)
├── priority     ENUM(low, medium, high)
├── owner_id     FK → users.id
├── created_at   DATETIME
└── updated_at   DATETIME
```

---

## Scalability Notes

See [`SCALABILITY.md`](./SCALABILITY.md) for full details. Summary:

- **API versioning** (`/api/v1/`) — add v2 without breaking existing clients
- **Modular structure** — add new modules (e.g. `comments`, `projects`) by adding a folder under `api/v1/endpoints/`
- **Database** — swap SQLite → PostgreSQL by changing one line in `.env` (`DATABASE_URL`)
- **Caching** — add Redis in front of `GET /tasks` to cache per-user task lists
- **Horizontal scaling** — stateless JWT auth means any number of API instances can run behind a load balancer
- **Docker** — add `Dockerfile` + `docker-compose.yml` for containerized deployment
- **Background jobs** — plug in Celery + Redis for async tasks (email notifications, exports)

---

## Running Tests (example)

```bash
pip install pytest httpx
pytest tests/
```

---

## Environment Variables

| Variable                     | Default     | Description                  |
|------------------------------|-------------|------------------------------|
| `SECRET_KEY`                 | (change me) | JWT signing secret           |
| `ALGORITHM`                  | `HS256`     | JWT algorithm                |
| `ACCESS_TOKEN_EXPIRE_MINUTES`| `30`        | Token lifetime               |
| `DATABASE_URL`               | `sqlite:///./taskflow.db` | DB connection |
