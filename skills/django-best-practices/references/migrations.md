# Django Migrations Best Practices

Reference for makemigrations workflow, safe production usage, custom migrations, data migrations, dependencies, squashing, and dangerous operations.

## makemigrations Best Practices

**Wrong:**
```python
# Making one giant migration after weeks of model changes
# 0001_initial.py — 50 operations, impossible to review or rollback
python manage.py makemigrations  # After changing 15 models at once
```

**Correct:**
```bash
# Small, focused migrations after each logical change
python manage.py makemigrations users --name add_phone_field
python manage.py makemigrations orders --name add_shipping_address
```

```python
# Review the generated migration before committing
# users/migrations/0003_add_phone_field.py
from django.db import migrations, models

class Migration(migrations.Migration):
    dependencies = [
        ('users', '0002_add_email_verified'),
    ]
    operations = [
        migrations.AddField(
            model_name='user',
            name='phone',
            field=models.CharField(blank=True, max_length=20),
        ),
    ]
```

> **Why:** Small migrations are easy to review, debug, and rollback. Name them descriptively so you can understand the migration history at a glance.

## Safe Migration Usage in Production

**Wrong:**
```bash
# Running migrate blindly on production without checking
python manage.py migrate --run-syncdb
# Or running migrations that aren't backward compatible while old code is still serving
```

**Correct:**
```bash
# Always check what will run first
python manage.py showmigrations
python manage.py migrate --plan

# In production deployments:
# 1. Deploy new code (backward compatible)
# 2. Run migrations
# 3. Deploy code that uses new schema
```

```python
# Make AddField migrations backward-compatible
migrations.AddField(
    model_name='order',
    name='tracking_number',
    field=models.CharField(max_length=100, blank=True, default=''),
    # blank=True + default='' means old code won't break
)
```

> **Why:** Never assume migrations are safe to run blindly. Use `--plan` to preview, and ensure backward compatibility so old code keeps working during the migration window.

## Custom Migrations

**Wrong:**
```python
# Running raw SQL without a reverse operation
from django.db import migrations

class Migration(migrations.Migration):
    operations = [
        migrations.RunSQL("UPDATE users SET is_active = true WHERE last_login IS NOT NULL"),
        # No reverse — can't roll back this migration
    ]
```

**Correct:**
```python
from django.db import migrations


def activate_recent_users(apps, schema_editor):
    User = apps.get_model('users', 'User')
    User.objects.filter(last_login__isnull=False).update(is_active=True)


def deactivate_recent_users(apps, schema_editor):
    User = apps.get_model('users', 'User')
    User.objects.filter(last_login__isnull=False).update(is_active=False)


class Migration(migrations.Migration):
    dependencies = [
        ('users', '0005_add_is_active'),
    ]
    operations = [
        migrations.RunPython(activate_recent_users, deactivate_recent_users),
    ]
```

> **Why:** Always provide both forward and reverse functions. Use `apps.get_model()` to access the model as it exists at that migration point, not the current model.

## Data Migrations

**Wrong:**
```python
from django.db import migrations
from myapp.models import User  # Importing the current model directly

def populate_data(apps, schema_editor):
    # This breaks if the model changes later
    for user in User.objects.all():
        user.full_name = f'{user.first_name} {user.last_name}'
        user.save()
```

**Correct:**
```python
from django.db import migrations


def populate_full_name(apps, schema_editor):
    User = apps.get_model('users', 'User')
    users = User.objects.all()
    for user in users:
        user.full_name = f'{user.first_name} {user.last_name}'
    User.objects.bulk_update(users, ['full_name'], batch_size=1000)


def reverse_full_name(apps, schema_editor):
    User = apps.get_model('users', 'User')
    User.objects.all().update(full_name='')


class Migration(migrations.Migration):
    dependencies = [
        ('users', '0006_add_full_name'),
    ]
    operations = [
        migrations.RunPython(populate_full_name, reverse_full_name),
    ]
```

> **Why:** Always use `apps.get_model()` in data migrations — it returns the model at that point in migration history. Direct imports reference the current model, which may differ.

## Migration Dependencies

**Wrong:**
```python
# Migration in orders app references users app without declaring dependency
from django.db import migrations, models

class Migration(migrations.Migration):
    dependencies = [
        ('orders', '0001_initial'),
        # Missing: ('users', '0003_add_email') — this will fail if run out of order
    ]
    operations = [
        migrations.AddField(
            model_name='order',
            name='user_email',
            field=models.CharField(max_length=255),
        ),
    ]
```

**Correct:**
```python
from django.db import migrations, models

class Migration(migrations.Migration):
    dependencies = [
        ('orders', '0001_initial'),
        ('users', '0003_add_email'),  # Explicit cross-app dependency
    ]
    operations = [
        migrations.AddField(
            model_name='order',
            name='user_email',
            field=models.CharField(max_length=255),
        ),
    ]
```

> **Why:** Django auto-detects most dependencies, but cross-app data migrations need explicit dependencies. Missing dependencies cause failures when migrations run in unexpected order.

## Migration Squashing

**Wrong:**
```bash
# Deleting old migrations and recreating from scratch
rm myapp/migrations/0001_*.py myapp/migrations/0002_*.py
python manage.py makemigrations myapp
# Breaks all existing databases that already applied those migrations
```

**Correct:**
```bash
# Squash a range of migrations safely
python manage.py squashmigrations myapp 0001 0010
# This creates 0001_squashed_0010_auto_*.py
# Old migrations are kept until all databases have migrated past them
# Then remove old migrations and update the squashed migration's replaces field
```

```python
# The squashed migration replaces the old ones
class Migration(migrations.Migration):
    replaces = [
        ('myapp', '0001_initial'),
        ('myapp', '0002_add_field'),
        # ...
        ('myapp', '0010_add_index'),
    ]
    operations = [
        # Consolidated operations
    ]
```

> **Why:** Squashing reduces migration count without breaking existing databases. The `replaces` attribute tells Django to skip individual migrations if the squashed one has been applied.

## Dangerous Migration Operations

**Wrong:**
```python
# Dropping a column that's still referenced by running code
class Migration(migrations.Migration):
    operations = [
        migrations.RemoveField(model_name='user', name='legacy_email'),
        # If old code is still deployed, this crashes immediately
    ]
```

**Correct:**
```python
# Step 1: Deploy code that stops reading/writing the field
# Step 2: Make the field nullable (safe, backward compatible)
class Migration(migrations.Migration):
    operations = [
        migrations.AlterField(
            model_name='user',
            name='legacy_email',
            field=models.CharField(max_length=255, null=True, blank=True),
        ),
    ]

# Step 3: After confirming no code uses it, remove the field
class Migration(migrations.Migration):
    operations = [
        migrations.RemoveField(model_name='user', name='legacy_email'),
    ]
```

> **Why:** Zero-downtime deployments require multi-step migrations: first make the field optional, deploy code that doesn't use it, then remove the column. Never drop a column that running code depends on.

> **Note:** `null=True` on a CharField is normally an anti-pattern (see the null vs blank rule in models.md) — it's acceptable here only as a transitional step of a zero-downtime removal. The final state drops the field (or drops null), so the two-empty-states problem never ships as permanent schema.

## AlterConstraint (Django 5.2+)

**Wrong:**
```python
from django.db import migrations, models

# Dropping and recreating a constraint just to rename or adjust it
class Migration(migrations.Migration):
    operations = [
        migrations.RemoveConstraint(model_name='order', name='amount_gt_0'),
        migrations.AddConstraint(
            model_name='order',
            constraint=models.CheckConstraint(
                condition=models.Q(amount__gt=0),
                name='order_amount_positive',
            ),
        ),
        # Recreating revalidates every row and takes locks it doesn't need
    ]
```

**Correct:**
```python
from django.db import migrations, models

class Migration(migrations.Migration):
    operations = [
        migrations.AlterConstraint(
            model_name='order',
            name='amount_gt_0',
            constraint=models.CheckConstraint(
                condition=models.Q(amount__gt=0),
                name='order_amount_positive',
            ),
        ),
    ]
```

> **Why:** `AlterConstraint` (Django 5.2+) updates a constraint definition in place without dropping and recreating it, so the table isn't rescanned and the constraint never stops being enforced. Prefer it over Remove+Add pairs when renaming or tightening constraints in zero-downtime deployments.
