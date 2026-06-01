# Playbook: API Layer

---

## 1. Custom Permissions

DRF permissions = Express middleware for authorization.

### Two Levels

| Level | Method | When it runs |
|---|---|---|
| **View-level** | `has_permission(request, view)` | Before anything — "is user logged in?" |
| **Object-level** | `has_object_permission(request, view, obj)` | After object is fetched — "is user the owner?" |

### Writing a Permission

```python
# orders/permissions.py
from rest_framework.permissions import BasePermission

class IsOrderOwner(BasePermission):
    def has_object_permission(self, request, view, obj):
        return obj.customer == request.user
```

### Per-Action Permissions — Two Approaches

**Approach 1: Separate views (default choice)**

```python
# orders/apis.py

class OrderDetailAPI(RetrieveAPIView):
    permission_classes = [IsAuthenticated, IsOrderOwner]
    ...

class OrderDeleteAPI(DestroyAPIView):
    permission_classes = [IsAuthenticated, IsAdminUser]
    ...
```

**Approach 2: `get_permissions()` override (when splitting causes too much duplication)**

```python
class OrderDetailAPI(RetrieveDestroyAPIView):
    def get_permissions(self):
        if self.request.method == 'DELETE':
            return [IsAuthenticated(), IsAdminUser()]
        return [IsAuthenticated(), IsOrderOwner()]
```

### When to use which

| Approach | When to use |
|---|---|
| Separate views | Default. Simpler, easier to test, fits service/selector pattern |
| `get_permissions()` | When splitting causes too much duplication |

### Where permissions live

- App-specific → `appX/permissions.py`
- Shared across apps → `core/permissions.py`

### Global Default — set once, forget safely

```python
# config/settings/base.py
REST_FRAMEWORK = {
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
}
```

If you use a single settings module, the same block goes in `config/settings.py`.

Now every view requires login by default. Override only for public endpoints:

```python
class RegisterAPI(CreateAPIView):
    permission_classes = [AllowAny]
```

### Combining Permissions — `|` and `&`

```python
# "owner OR admin can edit"
permission_classes = [IsAuthenticated, (IsOrderOwner | IsAdminUser)]
```

- `|` = OR — either one passes
- `&` = AND — both must pass (default when listed)

### Custom Error Messages

```python
class IsOrderOwner(BasePermission):
    message = "You can only access your own orders."

    def has_object_permission(self, request, view, obj):
        return obj.customer == request.user
```

### Django Model Permissions (admin-facing APIs only)

Django auto-creates 4 permissions per model: `add_X`, `change_X`, `delete_X`, `view_X`.

```python
from rest_framework.permissions import DjangoModelPermissions

class OrderAPI(ModelViewSet):
    permission_classes = [DjangoModelPermissions]
```

Auto-maps: `POST` → `add`, `PUT/PATCH` → `change`, `DELETE` → `delete`.

**Use for:** Admin-facing APIs with role-based access via Django groups.
**Not for:** Customer-facing APIs — use custom permissions like `IsOrderOwner`.

---

## 2. API Views — “APIView-first”

Default to `APIView` for explicit request/response handling.

Why:
- Keeps endpoints small and predictable.
- Makes it easy to enforce the service/selector split.
- Works nicely with per-endpoint serializers and schema generation.

### Pattern: nested serializers per endpoint

Each endpoint defines its own `InputSerializer` / `OutputSerializer`.

```python
from rest_framework import serializers, status
from rest_framework.response import Response
from rest_framework.views import APIView

from . import selectors, services


class AuctionCreateApi(APIView):
    class InputSerializer(serializers.Serializer):
        title = serializers.CharField(max_length=255)
        starts_at = serializers.DateTimeField()
        ends_at = serializers.DateTimeField()

    class OutputSerializer(serializers.Serializer):
        id = serializers.IntegerField()
        title = serializers.CharField()

    def post(self, request):
        serializer = self.InputSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)

        auction = services.auction_create(user=request.user, **serializer.validated_data)
        auction = selectors.auction_get(auction_id=auction.id)

        return Response(self.OutputSerializer(auction).data, status=status.HTTP_201_CREATED)
```

Rules:
- `GET` → validate `request.query_params`
- `POST/PUT/PATCH` → validate `request.data`
- Views should not contain business rules. If it’s “if/else domain logic”, it belongs in a service.

### Pagination (PageNumberPagination)

Keep pagination close to the endpoint if it’s only used there.

```python
from rest_framework.pagination import PageNumberPagination


class AuctionPagination(PageNumberPagination):
    page_size = 20
    page_size_query_param = 'page_size'
    max_page_size = 100
```

---

## 3. OpenAPI Schema — `InputSerializer` / `OutputSerializer`

If you use drf-spectacular, you can make it auto-detect the nested serializers.

```python
# config/settings.py
REST_FRAMEWORK = {
    'DEFAULT_SCHEMA_CLASS': 'core.schema.ApiAutoSchema',
}
```

Then views can declare `InputSerializer` / `OutputSerializer`, and the schema generator uses them.

Benefits:
- No manual `@extend_schema` for most endpoints.
- Request/response schema names stay unique per endpoint.

---

## 4. Unified Error Shape (exception handler)

Goal: every API error response follows one shape.

Example shape:

```json
{
  "message": "Validation error",
  "extra": {"fields": {"email": ["This field is required."]}}
}
```

Set it once:

```python
# config/settings.py
REST_FRAMEWORK = {
    'EXCEPTION_HANDLER': 'core.exceptions.api_exception_handler',
}
```

### Business errors: raise ApplicationError from services

```python
from core.exceptions import ApplicationError

raise ApplicationError(
    message='Auction duration must be at least 1 hour(s).',
    extra={'min_hours': 1},
)
```

Rules:
- Services raise `ApplicationError` for “valid request, but not allowed by business rules”.
- Views do not catch/translate errors. Let the exception handler produce the response.
