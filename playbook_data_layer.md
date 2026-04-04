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
