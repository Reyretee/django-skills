# Django Models & ORM Best Practices

Reference for model definition, field types, relationships, meta options, methods, managers, abstract/proxy models, and custom QuerySet methods.

## Model Definition

**Wrong:**
```python
from django.db import models

class product(models.Model):
    n = models.CharField(max_length=100)
    d = models.CharField(max_length=5000)
    p = models.FloatField()
    active = models.CharField(max_length=5, default='yes')
```

**Correct:**
```python
from django.db import models
from django.utils.translation import gettext_lazy as _


class Product(models.Model):
    name = models.CharField(_('name'), max_length=255)
    description = models.TextField(_('description'), blank=True)
    price = models.DecimalField(_('price'), max_digits=10, decimal_places=2)
    is_active = models.BooleanField(_('active'), default=True)

    class Meta:
        verbose_name = _('product')
        verbose_name_plural = _('products')

    def __str__(self):
        return self.name
```

> **Why:** Use descriptive field names, correct field types (DecimalField for money, BooleanField for flags, TextField for long text), and verbose_name for admin/i18n support.

## Field Types

**Wrong:**
```python
from django.db import models

class Event(models.Model):
    title = models.TextField()  # Short text in TextField
    price = models.FloatField()  # Float for money — rounding errors
    event_date = models.DateTimeField()  # Only need date, not time
    email = models.CharField(max_length=255)  # No validation
```

**Correct:**
```python
from django.db import models


class Event(models.Model):
    title = models.CharField(max_length=255)  # CharField for short text
    price = models.DecimalField(max_digits=10, decimal_places=2)  # Exact precision
    event_date = models.DateField()  # DateField when time isn't needed
    email = models.EmailField()  # Built-in email validation
```

> **Why:** CharField has max_length enforced at DB level. FloatField causes rounding errors with currency. DateField vs DateTimeField — pick what you actually need.

## null vs blank

**Wrong:**
```python
from django.db import models

class Profile(models.Model):
    nickname = models.CharField(max_length=100, null=True, blank=True)
    # Two empty states: NULL and '' — every query must check both
    bio = models.TextField(null=True)
```

**Correct:**
```python
from django.db import models


class Profile(models.Model):
    nickname = models.CharField(max_length=100, blank=True)  # Empty is always ''
    bio = models.TextField(blank=True)
    # null=True belongs on non-string types
    birth_date = models.DateField(null=True, blank=True)
    referred_by = models.ForeignKey(
        'self',
        null=True,
        blank=True,
        on_delete=models.SET_NULL,
        related_name='referrals',
    )
```

> **Why:** `null=True` on CharField/TextField creates two distinct empty states (NULL and `''`), so `filter(nickname='')` silently misses NULL rows. Use `blank=True` alone for optional strings; reserve `null=True` for non-string types — dates, numbers, and foreign keys — where NULL is the only way to represent "no value".

## TextChoices and IntegerChoices

**Wrong:**
```python
from django.db import models

class Order(models.Model):
    status = models.CharField(max_length=20)  # No choices — any string goes

# Magic strings scattered across the codebase — typos fail silently
# Order.objects.filter(status='payed')
```

**Correct:**
```python
from django.db import models
from django.utils.translation import gettext_lazy as _


class Order(models.Model):
    class Status(models.TextChoices):
        DRAFT = 'draft', _('Draft')
        PAID = 'paid', _('Paid')
        SHIPPED = 'shipped', _('Shipped')

    status = models.CharField(
        max_length=20,
        choices=Status.choices,
        default=Status.DRAFT,
    )

# Reference the enum, never magic strings
paid_orders = Order.objects.filter(status=Order.Status.PAID)
order.get_status_display()  # Human-readable label: 'Paid'
```

> **Why:** TextChoices/IntegerChoices give a single source of truth for allowed values, validation in forms and admin, autocompletion, and `get_<field>_display()` for labels. A magic string like `'payed'` matches nothing and fails silently at runtime.

## ForeignKey

**Wrong:**
```python
from django.db import models

class Comment(models.Model):
    post = models.ForeignKey('Post', on_delete=models.CASCADE)
    # No related_name — default is comment_set
    # No db_index consideration
    user = models.ForeignKey('auth.User', on_delete=models.CASCADE)
    # Hardcoded auth.User — breaks with custom user model
```

**Correct:**
```python
from django.conf import settings
from django.db import models


class Comment(models.Model):
    post = models.ForeignKey(
        'blog.Post',
        on_delete=models.CASCADE,
        related_name='comments',
    )
    user = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
        related_name='comments',
    )
```

> **Why:** Always use `settings.AUTH_USER_MODEL` for user references. Explicit `related_name` makes reverse queries readable: `post.comments.all()` instead of `post.comment_set.all()`.

## ManyToMany with Through Model

**Wrong:**
```python
from django.db import models

class Project(models.Model):
    name = models.CharField(max_length=255)
    members = models.ManyToManyField('auth.User')
    # No way to store role, date_joined, or any extra data
```

**Correct:**
```python
from django.conf import settings
from django.db import models


class Project(models.Model):
    name = models.CharField(max_length=255)
    members = models.ManyToManyField(
        settings.AUTH_USER_MODEL,
        through='ProjectMembership',
        related_name='projects',
    )


class ProjectMembership(models.Model):
    class Role(models.TextChoices):
        OWNER = 'owner', 'Owner'
        EDITOR = 'editor', 'Editor'
        VIEWER = 'viewer', 'Viewer'

    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    project = models.ForeignKey(Project, on_delete=models.CASCADE)
    role = models.CharField(max_length=20, choices=Role.choices, default=Role.VIEWER)
    joined_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        unique_together = ('user', 'project')
```

> **Why:** Use a through model when you need metadata on the relationship (role, timestamps, permissions). You almost always need it eventually — add it early.

## OneToOneField

**Wrong:**
```python
from django.conf import settings
from django.db import models

class Profile(models.Model):
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    # ForeignKey allows multiple profiles per user — probably a bug
    bio = models.TextField(blank=True)
```

**Correct:**
```python
from django.conf import settings
from django.db import models


class Profile(models.Model):
    user = models.OneToOneField(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
        related_name='profile',
    )
    bio = models.TextField(blank=True)
    avatar = models.ImageField(upload_to='avatars/', blank=True)
```

> **Why:** OneToOneField enforces the one-to-one constraint at the database level. Access is cleaner too: `user.profile` instead of `user.profile_set.first()`.

## GenericForeignKey

**Wrong:**
```python
from django.contrib.contenttypes.fields import GenericForeignKey
from django.contrib.contenttypes.models import ContentType
from django.db import models

class Comment(models.Model):
    # GFK reached for even though the target models are known and finite
    content_type = models.ForeignKey(ContentType, on_delete=models.CASCADE)
    object_id = models.PositiveIntegerField()
    content_object = GenericForeignKey('content_type', 'object_id')
```

**Correct:**
```python
from django.db import models


class Comment(models.Model):
    # Explicit nullable FKs — DB integrity, JOINs, and select_related all work
    post = models.ForeignKey(
        'blog.Post',
        null=True,
        blank=True,
        on_delete=models.CASCADE,
        related_name='comments',
    )
    photo = models.ForeignKey(
        'gallery.Photo',
        null=True,
        blank=True,
        on_delete=models.CASCADE,
        related_name='comments',
    )

    class Meta:
        constraints = [
            models.CheckConstraint(
                condition=(
                    models.Q(post__isnull=False, photo__isnull=True)
                    | models.Q(post__isnull=True, photo__isnull=False)
                ),
                name='comment_exactly_one_target',
            ),
        ]
```

> **Why:** GenericForeignKeys defeat foreign key integrity (the database can't enforce that the target row exists), can't be JOINed, and don't work with `select_related`. Prefer explicit FKs or separate tables per relationship. If you must keep a GFK, `prefetch_related('content_object')` does work for batch fetching.

## Model Meta

**Wrong:**
```python
from django.db import models

class Article(models.Model):
    title = models.CharField(max_length=255)
    created_at = models.DateTimeField(auto_now_add=True)
    # No ordering, no indexes, no constraints
```

**Correct:**
```python
from django.db import models


class Article(models.Model):
    title = models.CharField(max_length=255)
    slug = models.SlugField(max_length=255)
    status = models.CharField(max_length=20)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ['-created_at']
        indexes = [
            models.Index(fields=['status', 'created_at']),
            models.Index(fields=['slug']),
        ]
        constraints = [
            models.UniqueConstraint(
                fields=['slug'],
                condition=models.Q(status='published'),
                name='unique_published_slug',
            ),
            models.CheckConstraint(
                condition=models.Q(title__gt=''),
                name='article_title_not_empty',
            ),
        ]
        verbose_name = 'article'
        verbose_name_plural = 'articles'
```

> **Why:** Indexes speed up queries you run often. Constraints enforce data integrity at the DB level. Default ordering saves repeating `.order_by()` everywhere.

> **Note:** `CheckConstraint` takes `condition=` — the old `check=` kwarg was deprecated in Django 5.1 and removed in 6.0. Also, Django 6.0 makes `BigAutoField` the implicit default primary key; existing projects should keep `DEFAULT_AUTO_FIELD = 'django.db.models.BigAutoField'` pinned explicitly in settings to avoid surprise migrations.

## Composite Primary Keys (Django 5.2+)

**Wrong:**
```python
from django.db import models

# Adopting a composite PK on a greenfield model for "purity"
class OrderItem(models.Model):
    pk = models.CompositePrimaryKey('product_id', 'order_id')
    product_id = models.IntegerField()
    order_id = models.CharField(max_length=20)
```

**Correct:**
```python
from django.db import models


# Greenfield: keep the surrogate BigAutoField PK + a uniqueness constraint
class OrderItem(models.Model):
    product = models.ForeignKey('shop.Product', on_delete=models.CASCADE)
    order = models.ForeignKey('shop.Order', on_delete=models.CASCADE)

    class Meta:
        constraints = [
            models.UniqueConstraint(
                fields=['product', 'order'],
                name='unique_product_per_order',
            ),
        ]


# Legacy database whose table already has a composite key
class LegacyOrderItem(models.Model):
    pk = models.CompositePrimaryKey('product_id', 'order_id')
    product_id = models.IntegerField()
    order_id = models.CharField(max_length=20)

    class Meta:
        managed = False
        db_table = 'order_items'

# item.pk is a tuple, and you can filter on it
item = LegacyOrderItem.objects.get(pk=(1, 'A755H'))
item.pk  # (1, 'A755H')
```

> **Why:** `CompositePrimaryKey` exists mainly for mapping legacy databases. Its caveats are severe: no ForeignKey can target a composite-PK model, it's not supported in the admin, it's excluded from ModelForms, and a table can't be migrated to or from a composite PK after creation. Greenfield models should keep a surrogate BigAutoField PK and enforce natural keys with a UniqueConstraint.

## Model Methods

**Wrong:**
```python
from django.db import models

class Order(models.Model):
    total = models.DecimalField(max_digits=10, decimal_places=2)
    status = models.CharField(max_length=20)
    # No __str__ — admin shows "Order object (1)"
    # No get_absolute_url — templates hardcode URLs
```

**Correct:**
```python
from django.db import models
from django.urls import reverse


class Order(models.Model):
    class Status(models.TextChoices):
        PENDING = 'pending', 'Pending'
        PAID = 'paid', 'Paid'
        SHIPPED = 'shipped', 'Shipped'

    total = models.DecimalField(max_digits=10, decimal_places=2)
    status = models.CharField(
        max_length=20,
        choices=Status.choices,
        default=Status.PENDING,
    )

    def __str__(self):
        return f'Order #{self.pk} — {self.total}'

    def get_absolute_url(self):
        return reverse('orders:detail', kwargs={'pk': self.pk})

    @property
    def is_paid(self):
        return self.status == self.Status.PAID

    def mark_as_shipped(self):
        if self.status != self.Status.PAID:
            raise ValueError('Only paid orders can be shipped')
        self.status = self.Status.SHIPPED
        self.save(update_fields=['status'])
```

> **Why:** `__str__` makes admin and debugging readable. `get_absolute_url` is used by Django's redirect shortcuts and admin. Methods encapsulate business logic on the model.

## Field Validators

**Wrong:**
```python
from django.core.validators import MinValueValidator
from django.db import models

class Product(models.Model):
    price = models.DecimalField(
        max_digits=10,
        decimal_places=2,
        validators=[MinValueValidator(0)],
    )

# Elsewhere — assuming the validator protects direct saves
product = Product(price=-5)
product.save()  # Saves fine! Validators do NOT run on .save()
```

**Correct:**
```python
from django.core.exceptions import ValidationError
from django.core.validators import MinValueValidator
from django.db import models
from django.utils.deconstruct import deconstructible


@deconstructible  # Lets the validator serialize into migrations
class MultipleOfValidator:
    def __init__(self, base):
        self.base = base

    def __call__(self, value):
        if value % self.base != 0:
            raise ValidationError(f'Must be a multiple of {self.base}.')


class Product(models.Model):
    price = models.DecimalField(
        max_digits=10,
        decimal_places=2,
        validators=[MinValueValidator(0)],
    )

    class Meta:
        constraints = [
            # DB-level enforcement — covers every write path, including bulk ops
            models.CheckConstraint(
                condition=models.Q(price__gte=0),
                name='product_price_non_negative',
            ),
        ]

    def save(self, *args, **kwargs):
        self.full_clean()  # Run field validators for critical invariants
        super().save(*args, **kwargs)
```

> **Why:** Field `validators=[...]` only run in `full_clean()` and ModelForm validation — never on a direct `.save()`. For critical invariants, call `full_clean()` in `save()` or enforce with database constraints. Decorate custom validator classes with `@deconstructible` so they serialize into migrations.

## Model Managers

**Wrong:**
```python
from django.db import models

class Article(models.Model):
    title = models.CharField(max_length=255)
    is_published = models.BooleanField(default=False)

# In views — filtering repeated everywhere
# Article.objects.filter(is_published=True)
# Article.objects.filter(is_published=True, category='tech')
```

**Correct:**
```python
from django.db import models


class PublishedManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().filter(is_published=True)


class Article(models.Model):
    title = models.CharField(max_length=255)
    is_published = models.BooleanField(default=False)

    objects = models.Manager()  # Default manager
    published = PublishedManager()  # Custom manager

# Usage: Article.published.all()
# Usage: Article.published.filter(category='tech')
```

> **Why:** Custom managers encapsulate common filters, keeping views DRY. Always keep `objects` as the default manager to avoid surprises with admin and related lookups.

## Abstract Models

**Wrong:**
```python
from django.db import models

class Article(models.Model):
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    title = models.CharField(max_length=255)

class Comment(models.Model):
    created_at = models.DateTimeField(auto_now_add=True)  # Duplicated
    updated_at = models.DateTimeField(auto_now=True)       # Duplicated
    body = models.TextField()
```

**Correct:**
```python
from django.db import models


class TimeStampedModel(models.Model):
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True


class Article(TimeStampedModel):
    title = models.CharField(max_length=255)


class Comment(TimeStampedModel):
    body = models.TextField()
```

> **Why:** Abstract models eliminate field duplication across models. `abstract = True` means no database table is created for the base class — fields are added to each child table.

## Proxy Models

**Wrong:**
```python
from django.db import models

class Order(models.Model):
    status = models.CharField(max_length=20)
    total = models.DecimalField(max_digits=10, decimal_places=2)

# Separate model with duplicate fields just for a different admin view
class PendingOrder(models.Model):
    status = models.CharField(max_length=20)
    total = models.DecimalField(max_digits=10, decimal_places=2)
```

**Correct:**
```python
from django.db import models


class Order(models.Model):
    status = models.CharField(max_length=20)
    total = models.DecimalField(max_digits=10, decimal_places=2)


class PendingOrderManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().filter(status='pending')


class PendingOrder(Order):
    objects = PendingOrderManager()

    class Meta:
        proxy = True
        verbose_name = 'pending order'
        verbose_name_plural = 'pending orders'
```

> **Why:** Proxy models share the same database table but allow different Python behavior, managers, and admin registrations. No data duplication, no migration overhead.

## Custom QuerySet Methods

**Wrong:**
```python
from django.db import models

# Filtering logic scattered across views
# views.py
def active_premium_users(request):
    users = User.objects.filter(is_active=True, plan='premium')
    ...

def dashboard(request):
    users = User.objects.filter(is_active=True, plan='premium')  # Duplicated
    ...
```

**Correct:**
```python
from django.db import models


class UserQuerySet(models.QuerySet):
    def active(self):
        return self.filter(is_active=True)

    def premium(self):
        return self.filter(plan='premium')

    def with_order_count(self):
        return self.annotate(order_count=models.Count('orders'))


class User(models.Model):
    is_active = models.BooleanField(default=True)
    plan = models.CharField(max_length=20)

    objects = UserQuerySet.as_manager()

# Usage — chainable: User.objects.active().premium().with_order_count()
```

> **Why:** QuerySet methods are chainable and reusable. `as_manager()` turns a QuerySet into a manager, giving you both custom methods and full QuerySet API.
