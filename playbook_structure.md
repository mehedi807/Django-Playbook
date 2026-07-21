# Playbook: Project Structure

---

## Layout

```
project_root/
    manage.py
    .env
    requirements.txt
    config/                      ← project config (NOT a Django app)
        settings/
            base.py
            dev.py
            prod.py
        env.py                ← django-environ wrapper
        celery.py             ← Celery app bootstrap
        urls.py
        wsgi.py
        asgi.py
    appX/
        apis.py               ← user-facing APIs (or split: apis_user/apis_admin)
        urls.py               ← include() endpoints (or split urls_user/urls_admin)
        selectors.py
        services.py
        tasks.py
        consumers.py          ← WebSocket consumer(s), if needed
        routing.py            ← websocket_urlpatterns, if needed
        managers.py           ← reusable query vocabulary on models, if needed
        permissions.py        ← custom DRF permissions, if needed
        models.py
        constants.py
    core/
        apps.py               ← AppConfig declaration for INSTALLED_APPS
        models.py             ← base abstract models
        exceptions.py         ← API exception handler + ApplicationError
        schema.py             ← drf-spectacular AutoSchema tweaks
        services.py           ← shared helpers (e.g., model_update)
        ws_auth.py            ← JWT auth middleware for websockets
        pagination.py
```

### Variant: Single `settings.py` (reference backend style)

If your project is small/medium and you want less file hopping, keep settings in one place.

```
project_root/
    manage.py
    .env
    requirements.txt
    config/
        settings.py           ← single settings module
        env.py                ← django-environ wrapper
        urls.py               ← top-level API routing
        celery.py             ← Celery app bootstrap
        asgi.py               ← Channels + JWT WS middleware wiring
        wsgi.py
    appX/
        apis.py               ← user-facing APIs (or split: apis_user/apis_admin)
        urls.py               ← include() endpoints (or split urls_user/urls_admin)
        selectors.py
        services.py
        tasks.py
        consumers.py          ← WebSocket consumer(s), if needed
        routing.py            ← websocket_urlpatterns, if needed
        managers.py           ← reusable query vocabulary on models, if needed
        permissions.py        ← custom DRF permissions, if needed
        models.py
        constants.py
    core/
        apps.py               ← AppConfig declaration for INSTALLED_APPS
        models.py             ← base abstract models
        exceptions.py         ← API exception handler + ApplicationError
        schema.py             ← drf-spectacular AutoSchema tweaks
        services.py           ← shared helpers (e.g., model_update)
        ws_auth.py            ← JWT auth middleware for websockets
```

## Key Rules

- **If only one app uses it, it belongs in that app.**
- `config/` = how the project runs (settings, urls, wsgi). No business logic.
- `core/` = shared Django app (base models, global constants, shared permissions).

## Naming Conventions

### `apis.py` vs `apis_user.py` / `apis_admin.py`

Use `apis.py` when the app exposes one audience.

Split by audience when it prevents accidental permission leaks:

- `apis_user.py` + `urls_user.py` → customer/user-facing endpoints
- `apis_admin.py` + `urls_admin.py` → staff/admin-facing endpoints

### `urls.py`

- App `urls.py` should only map URL → view.
- Project `config/urls.py` mounts apps under stable prefixes.

Example:

```python
# config/urls.py
urlpatterns = [
    path('api/auth/', include(('authentication.urls', 'authentication'), namespace='authentication')),
    path('api/auctions/', include(('auctions.urls', 'auctions'), namespace='auctions')),
]
```

## Background + Realtime (where code lives)

| Concern | Primary file | Notes |
|---|---|---|
| Background work | `appX/tasks.py` | Celery tasks should call services/selectors, not duplicate logic |
| Realtime WebSocket | `appX/consumers.py` | Consumer is transport-only; business logic stays in services |
| WebSocket routes | `appX/routing.py` | Defines `websocket_urlpatterns` |
| WebSocket auth | `core/ws_auth.py` | JWT auth middleware for `scope['user']` |

## Infrastructure (project root)

Common root files:

- `.env` + `.env.example` → local config
- `requirements.txt` (or `pyproject.toml`) → Python deps
- `Dockerfile` → container build
- `docker-compose.dev.yml` → local development stack
- `docker-compose.yml` → production stack

## config/ vs core/

| | `config/` | `core/` |
|---|---|---|
| What is it? | Project configuration | A Django app |
| Contains business logic? | No | Yes, shared logic |
| In `INSTALLED_APPS`? | No | Yes |
