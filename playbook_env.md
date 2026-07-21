# Playbook: Environment Configuration

---

## 1. The `.env` Approach

We use `django-environ` to manage configuration. This ensures a clean separation between code and secrets/environment-specific settings.

### Rule of thumb:
- **Never commit `.env` to version control.**
- **Always commit `.env.example`** with placeholder values to help other developers set up the project.

---

## 2. Shared Wrapper (`config/env.py`)

To keep imports clean and provide a single entry point for environment variables, create a simple wrapper:

```python
# config/env.py
import environ

env = environ.Env()
```

---

## 3. Usage in `settings.py`

Avoid calling `env(...)` directly for everything. Define them at the top of your settings file.

```python
from config.env import env

DEBUG = env.bool("DEBUG", default=False)
SECRET_KEY = env.str("SECRET_KEY")

# Email Configuration
EMAIL_HOST = env.str("EMAIL_HOST", default="smtp.gmail.com")
EMAIL_PORT = env.int("EMAIL_PORT", default=587)

# Third-party Services (Stripe, Google, etc.)
STRIPE_SECRET_KEY = env.str("STRIPE_SECRET_KEY", default="")
```

---

## 4. Environment Variables Categories

### Essential Django
- `DJANGO_ENV`: `dev`, `prod`, or `test`.
- `SECRET_KEY`: Long, random string.
- `ALLOWED_HOSTS`: Comma-separated list (e.g., `localhost,api.myapp.com`).

### Infrastructure
- `CELERY_BROKER_URL`: Connection string for Redis (e.g., `redis://:pass@redis:6379/0`).
- `DATABASE_URL`: Connection string if using Postgres (via `dj-database-url`).

### Third-Party Keys
- `STRIPE_SECRET_KEY` / `STRIPE_WEBHOOK_SECRET`
- `GOOGLE_OAUTH2_CLIENT_ID`
- `GEMINI_API_KEY` / `OPENAI_API_KEY`

---

## 5. Local Setup Flow

1. Copy the example file: `cp .env.example .env`.
2. Fill in the missing secrets (Stripe keys, DB passwords).
3. Ensure both `docker-compose.dev.yml` and `docker-compose.yml` point to the correct `.env` file via their `env_file` keys.
