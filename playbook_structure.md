# Playbook: Project Structure

---

## Layout

```
project_root/
    manage.py
    requirements/
    config/                      ← project config (NOT a Django app)
        settings/
            base.py
            dev.py
            prod.py
        urls.py
        wsgi.py
        asgi.py
    appX/
        apis.py                  ← handles request/response only
        services.py              ← DB write operations, business logic
        selectors.py             ← complex read queries for specific API needs
        managers.py              ← reusable query vocabulary on models
        permissions.py           ← custom DRF permissions
        constants.py             ← app-specific constants/choices
        tasks.py                 ← Celery background tasks
        models.py
        serializers.py
        urls.py
    core/                        ← shared Django app (in INSTALLED_APPS)
        constants.py             ← only truly global constants
```

## Key Rules

- **If only one app uses it, it belongs in that app.**
- `config/` = how the project runs (settings, urls, wsgi). No business logic.
- `core/` = shared Django app (base models, global constants, shared permissions).

## config/ vs core/

| | `config/` | `core/` |
|---|---|---|
| What is it? | Project configuration | A Django app |
| Contains business logic? | No | Yes, shared logic |
| In `INSTALLED_APPS`? | No | Yes |
