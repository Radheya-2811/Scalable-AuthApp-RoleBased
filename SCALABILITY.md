# Scalability Notes — TaskFlow

## Current Architecture

```
Client (Browser)
      │
      ▼
FastAPI (single instance)
      │
      ▼
SQLite (single file DB)
```

Simple, zero-dependency, perfect for development and small deployments.

---

## Path to Production Scale

### 1. Database → PostgreSQL

Change one line in `.env`:

```env
DATABASE_URL=postgresql://user:password@db-host:5432/taskflow
```

SQLAlchemy handles the rest. PostgreSQL gives you:
- Concurrent writes (SQLite has write locks)
- Connection pooling
- Read replicas for scaling reads

### 2. Caching with Redis

Add Redis in front of expensive queries:

```python
# Cache GET /tasks per user for 60 seconds
@router.get("/tasks")
async def list_tasks(...):
    cache_key = f"tasks:user:{current_user.id}"
    cached = await redis.get(cache_key)
    if cached:
        return json.loads(cached)
    tasks = db.query(Task)...
    await redis.setex(cache_key, 60, json.dumps(tasks))
    return tasks
```

Invalidate on write (POST/PATCH/DELETE).

### 3. Horizontal Scaling (Load Balancer)

Because JWT is **stateless**, any instance can validate any token:

```
          ┌──────────────────┐
          │   Load Balancer  │  (nginx / AWS ALB)
          └──────┬───────────┘
        ┌────────┴────────┐
   FastAPI #1        FastAPI #2     (identical, stateless)
        └────────┬────────┘
           PostgreSQL + Redis   (shared state)
```

Scale instances horizontally with zero code changes.

### 4. Docker Deployment

```dockerfile
# Dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```yaml
# docker-compose.yml
services:
  api:
    build: ./backend
    ports: ["8000:8000"]
    depends_on: [db, redis]
    environment:
      DATABASE_URL: postgresql://postgres:secret@db:5432/taskflow
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: secret
  redis:
    image: redis:7-alpine
```

### 5. Async Background Jobs (Celery)

For long-running tasks (email notifications, CSV exports):

```python
from celery import Celery
celery = Celery("taskflow", broker="redis://localhost:6379/0")

@celery.task
def send_task_reminder(user_id: int, task_id: int):
    ...  # send email
```

### 6. API Versioning

Already in place: `/api/v1/`. Add v2 without breaking v1 clients:

```
app/api/
├── v1/   ← existing clients
└── v2/   ← new features, breaking changes
```

### 7. Adding New Modules

The folder structure is intentionally modular. To add a `projects` module:

```
app/
├── models/projects.py       # SQLAlchemy model
├── schemas/projects.py      # Pydantic schemas
└── api/v1/endpoints/projects.py  # Router
```

Then one line in `router.py`:
```python
api_router.include_router(projects.router)
```

---

##  Table

| Concern              | Current           | Production Path              |
|----------------------|-------------------|------------------------------|
| Database             | SQLite            | PostgreSQL + read replicas   |
| Caching              | None              | Redis                        |
| Scaling              | Single instance   | Load balancer + N instances  |
| Background jobs      | None              | Celery + Redis               |
| Containerization     | None              | Docker + Compose / Kubernetes|
| Secrets management   | .env file         | AWS Secrets Manager / Vault  |
| Monitoring           | Uvicorn logs      | Prometheus + Grafana         |
| Rate limiting        | None              | slowapi or nginx limit_req   |
