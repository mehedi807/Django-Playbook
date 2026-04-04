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
