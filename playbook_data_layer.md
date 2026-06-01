# Playbook: Data Layer

---

## 1. Custom Managers & QuerySets

**Problem:** Same `.filter(...)` repeated across selectors, services, and tasks.

**Wrong fix:** Put it in a selector and import everywhere — breaks dependency direction.

**Right fix:** Model owns its query vocabulary via custom QuerySet.

```python
# products/managers.py
from django.db import models

class ProductQuerySet(models.QuerySet):
    def active(self):
        return self.filter(is_active=True)

    def in_stock(self):
        return self.filter(stock_count__gt=0)
```

```python
# products/models.py
from .managers import ProductQuerySet

class Product(models.Model):
    name = models.CharField(max_length=255)
    is_active = models.BooleanField(default=True)
    stock_count = models.IntegerField(default=0)

    objects = ProductQuerySet.as_manager()
```

**Usage — chainable everywhere:**
```python
Product.objects.active().in_stock()
Product.objects.active().in_stock().update(price=new_price)
Product.objects.active().count()
```

**Manager vs Selector:**
- Manager = reusable vocabulary (`.active()`, `.in_stock()`)
- Selector = specific API use-case that *uses* manager methods

```python
# products/selectors.py
def get_discounted_products_for_homepage(*, category_id, limit=10):
    return (
        Product.objects
        .active()
        .in_stock()
        .filter(category_id=category_id, discount__gt=0)
        .order_by('-discount')[:limit]
    )
```

---

## 2. Selectors (read-only query layer)

Selectors answer: “Given inputs, what data do we need?”

Rules:
- Read-only. No `.save()`, no side effects.
- Accept explicit inputs (no `request`).
- Return `QuerySet` for lists, or a model instance/`None` for detail fetches.
- Prefer `select_related()` / `prefetch_related()` / `annotate()` to avoid N+1.

Example:

```python
from django.db.models import Max


def auction_get(*, auction_id: int, user=None):
    return (
        Auction.objects
        .select_related('created_by', 'winner')
        .prefetch_related('images')
        .annotate(current_highest_bid=Max('bids__amount'))
        .filter(id=auction_id)
        .first()
    )
```

---

## 3. Services (write + business logic layer)

Services answer: “Perform this use-case safely and consistently.”

What belongs here:
- Creates/updates/deletes.
- Business validation.
- Orchestrating other services/selectors.
- Scheduling async work (Celery) after commit.

Rules:
- Services raise domain errors (`ApplicationError`) instead of returning “error dicts”.
- Wrap multi-step writes in `transaction.atomic()`.
- If an action triggers async side-effects, queue them using `transaction.on_commit(...)`.

### Shared helper: `model_update`

Use a single helper for safe partial updates:

```python
from core.services import model_update


instance, updated = model_update(
    instance=user_profile,
    fields=['phone', 'address', 'company'],
    data=payload,
)
```

This keeps:
- Update-field tracking
- `full_clean()` validation
- `updated_at` consistency

### Avoid circular imports (cross-app calls)

If a service needs another app’s model/config, import inside the function.

```python
def _get_min_bid_increment() -> int:
    from adminpanel.models import SiteConfiguration
    return SiteConfiguration.load().min_bid_increment
```

---

## 4. Tasks (Celery) — thin wrappers

Tasks should call services/selectors and return minimal output.

Patterns:
- `@shared_task(ignore_result=True)` for fire-and-forget.
- `autoretry_for` + `retry_backoff` for IO-heavy work.

```python
from celery import shared_task


@shared_task(bind=True, autoretry_for=(Exception,), retry_backoff=True, retry_kwargs={'max_retries': 3})
def complete_expired_auctions_task(self):
    count = auction_complete_expired()
    return f'Processed {count} expired auctions.'
```

### Queueing tasks from services

Prefer `current_app.send_task('app.tasks.task_name', kwargs={...})` when:
- You want to avoid importing the task function (keeps imports clean).
- You’re calling across apps.

If the task depends on newly-committed DB rows:

```python
from django.db import transaction
from celery import current_app


transaction.on_commit(
    lambda: current_app.send_task(
        'auctions.tasks.broadcast_new_bid_task',
        kwargs={'bid_id': bid.id, 'auction_id': auction.id},
    )
)
```

---

## 5. Realtime Events (Channels) — transport only

Rule: Consumers don’t implement business logic.

### Event naming convention

When sending group events, use dotted event types:

- Sent type: `auction.new_bid`
- Consumer handler: `auction_new_bid`

```python
async_to_sync(channel_layer.group_send)(
    f'auction.{auction_id}',
    {'type': 'auction.new_bid', 'payload': {...}},
)
```

Consumers receive and forward:

```python
class AuctionRealtimeConsumer(AsyncJsonWebsocketConsumer):
    async def auction_new_bid(self, event):
        await self.send_json({'type': 'auction.new_bid', 'payload': event.get('payload', {})})
```
