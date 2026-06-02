# Playbook: Docker & Infrastructure

---

## 1. Dockerfile (API & Workers)

**Goal:** Provide a consistent environment for Django, Celery, and other Python-based services.

Use `python:3.12-slim` for a balance between size and utility.

```dockerfile
FROM python:3.12-slim

# Prevents Python from writing pyc files to disc
ENV PYTHONDONTWRITEBYTECODE=1
# Prevents Python from buffering stdout and stderr
ENV PYTHONUNBUFFERED=1

WORKDIR /app

# Install system dependencies (build-essential for C extensions)
RUN apt-get update && apt-get install -y \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Install dependencies first for better caching
COPY requirements.txt /app/
RUN pip install --upgrade pip && pip install -r requirements.txt

# Copy project files
COPY . /app/
```

---

## 2. Docker Compose (Local Stack)

The `docker-compose.yml` orchestrates the API, background workers, and shared services (Redis).

### Services Overview

| Service | Purpose | Notes |
|---|---|---|
| `redis` | Message broker & Cache | Used by Celery and Channels |
| `web` | Django API | Runs `runserver` with auto-reload |
| `celery` | Background worker | Runs tasks from `appX/tasks.py` |
| `ngrok` | External Tunnel | Exposes local API for webhooks/mobile |

### Example Configuration

```yaml
services:
  redis:
    image: redis:7-alpine
    command: ["redis-server", "--requirepass", "${REDIS_PASSWORD}"]
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  web:
    build: .
    command: sh -c "python manage.py migrate && python manage.py runserver 0.0.0.0:8000"
    volumes:
      - .:/app
    ports:
      - "8000:8000"
    env_file:
      - .env
    depends_on:
      redis:
        condition: service_healthy

  celery:
    build: .
    command: celery -A config worker -l info
    volumes:
      - .:/app
    env_file:
      - .env
    depends_on:
      redis:
        condition: service_healthy
```

---

## 3. Connectivity Rules

- **Internal Hostnames:** Services talk to each other using service names (e.g., `REDIS_URL=redis://:password@redis:6379/0`).
- **Healthchecks:** Always use `depends_on` with `service_healthy` to ensure Redis is up before Django/Celery start.
- **Port Conflicts:** Ensure `ports` mappings (e.g., `8001:8000`) don't collide with other local projects.

---

## 4. Ngrok Tunneling

For testing webhooks (like Stripe or Twilio) or local mobile development, use an integrated `ngrok` service.

```yaml
  ngrok:
    image: ngrok/ngrok:latest
    ports: 
      - "4040:4040"
    environment:
      - NGROK_AUTHTOKEN=${NGROK_AUTHTOKEN}
    command: http web:8000 --domain=${NGROK_DOMAIN}
    depends_on:
      - web
```

**Workflow:**
1. Set `NGROK_AUTHTOKEN` in `.env`.
2. Start the stack: `docker compose up`.
3. Visit the dashboard at `localhost:4040` to see traffic.
