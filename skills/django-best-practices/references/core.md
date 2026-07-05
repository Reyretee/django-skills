Django Core best practices: project structure, app design, settings, URLs, WSGI/ASGI, and management commands.

## Project Structure

**Wrong:**
```python
# Everything in one directory, no separation
myproject/
    models.py      # 5000 lines, all models here
    views.py       # 3000 lines, all views here
    urls.py
    settings.py
    manage.py
```

**Correct:**
```python
# config/ holds project-level settings, each app is focused
myproject/
    config/
        __init__.py
        settings/
            __init__.py
            base.py
            local.py
            production.py
        urls.py
        wsgi.py
        asgi.py
    apps/
        users/
            __init__.py
            models.py
            views.py
            urls.py
            admin.py
            tests/
                __init__.py
                test_models.py
                test_views.py
        orders/
            ...
    manage.py
    requirements/
        base.txt
        local.txt
        production.txt
```

> **Why:** Separating config from apps keeps the project navigable as it grows. Each app owns its own domain, making it testable and reusable independently.

## Dependency Pinning

**Wrong:**
```bash
# requirements.txt — unpinned, resolves differently on every install
django
celery
psycopg
# "Works on my machine" — prod silently gets the next major release
```

**Correct:**
```bash
# requirements/base.in — direct dependencies with loose constraints
django>=6.0,<6.1   # Django 6.0 requires Python 3.12-3.14
celery
psycopg[binary]

# Compile to an exact lock file and commit both files:
pip-compile requirements/base.in -o requirements/base.txt   # pip-tools
# or: uv pip compile requirements/base.in -o requirements/base.txt

# requirements/base.txt (generated) — every version and sub-dependency pinned
# django==6.0.1
#     via -r requirements/base.in
# ...
```

> **Why:** Declare direct dependencies in `*.in` files and let a compiler (pip-tools or uv) produce fully pinned `*.txt` lock files — same `base`/`local`/`production` split as the `requirements/` folder above. Committed lock files make installs reproducible, and upgrades become an explicit, reviewable diff.

## Tooling

**Wrong:**
```bash
# black + isort + flake8, each with its own config, drifting out of sync
# No pre-commit hooks — style nits reach code review; deprecations fixed by hand
```

**Correct:**
```toml
# pyproject.toml — ruff replaces black, isort, and flake8
[tool.ruff]
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "DJ"]  # DJ = flake8-django rules
```

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.12.0
    hooks:
      - id: ruff
      - id: ruff-format
  - repo: https://github.com/adamchainz/django-upgrade
    rev: 1.25.0
    hooks:
      - id: django-upgrade
        args: [--target-version, "6.0"]
```

> **Why:** Ruff is a single fast tool for both linting and formatting. Pre-commit hooks catch issues before they reach CI, and `django-upgrade` automatically rewrites deprecated Django patterns when you bump versions.

## App Structure

**Wrong:**
```python
# One giant app doing everything
# shop/models.py
from django.db import models

class User(models.Model): ...
class Product(models.Model): ...
class Order(models.Model): ...
class Payment(models.Model): ...
class Review(models.Model): ...
class Notification(models.Model): ...
class BlogPost(models.Model): ...
```

**Correct:**
```python
# Small, focused apps with clear boundaries
# users/models.py
from django.db import models

class User(models.Model): ...

# products/models.py
from django.db import models

class Product(models.Model): ...

# orders/models.py
from django.db import models
from django.conf import settings

class Order(models.Model):
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    ...
```

> **Why:** Small apps are easier to test, reuse, and reason about. Keep apps focused (3-8 models per app) — when one app exceeds ~15 models, it's a sign you need to split it.

## Settings Configuration

**Wrong:**
```python
# settings.py — everything in one file, secrets hardcoded
SECRET_KEY = 'django-insecure-abc123realkey'
DEBUG = True
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'PASSWORD': 'mydbpassword',
    }
}
```

**Correct:**
```python
# config/settings/base.py
import os
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent.parent

SECRET_KEY = os.environ.get('DJANGO_SECRET_KEY')

# Django 6.0 changed the default to BigAutoField — keep it explicitly pinned
# on existing projects to avoid surprise migrations
DEFAULT_AUTO_FIELD = 'django.db.models.BigAutoField'

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    # ...
]

# config/settings/local.py
from .base import *  # noqa: F401,F403

DEBUG = True
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'db.sqlite3',
    }
}

# config/settings/production.py
from .base import *  # noqa: F401,F403

DEBUG = False
SECRET_KEY = os.environ['DJANGO_SECRET_KEY']
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.environ['DB_NAME'],
        'USER': os.environ['DB_USER'],
        'PASSWORD': os.environ['DB_PASSWORD'],
        'HOST': os.environ['DB_HOST'],
    }
}
```

> **Why:** Splitting settings per environment prevents accidental DEBUG=True in production and keeps secrets out of version control.

## URL Configuration

**Wrong:**
```python
# config/urls.py — all URLs in one flat file
from django.urls import path
from users.views import login_view, profile_view
from products.views import product_list, product_detail
from orders.views import order_list

urlpatterns = [
    path('login/', login_view),
    path('profile/', profile_view),
    path('products/', product_list),
    path('products/<int:pk>/', product_detail),
    path('orders/', order_list),
]
```

**Correct:**
```python
# config/urls.py
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('users/', include('apps.users.urls', namespace='users')),
    path('products/', include('apps.products.urls', namespace='products')),
    path('orders/', include('apps.orders.urls', namespace='orders')),
]

# apps/products/urls.py
from django.urls import path
from . import views

app_name = 'products'

urlpatterns = [
    path('', views.product_list, name='list'),
    path('<int:pk>/', views.product_detail, name='detail'),
]
```

> **Why:** Namespaced includes keep URL definitions close to their apps and prevent name collisions. `reverse('products:detail', kwargs={'pk': 1})` is unambiguous.

## WSGI and ASGI

**Wrong:**
```python
# Using WSGI when you need WebSocket or async support
# wsgi.py serving a project with Django Channels
import os
from django.core.wsgi import get_wsgi_application

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.settings')
application = get_wsgi_application()
# Then wondering why WebSocket connections fail
```

**Correct:**
```python
# wsgi.py — for traditional HTTP-only projects
import os
from django.core.wsgi import get_wsgi_application

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.settings.production')
application = get_wsgi_application()

# asgi.py — when you need async views, WebSocket, or Channels
import os
from django.core.asgi import get_asgi_application

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.settings.production')
application = get_asgi_application()

# With Channels:
# import os
# from channels.routing import ProtocolTypeRouter, URLRouter
# from channels.auth import AuthMiddlewareStack
# from django.core.asgi import get_asgi_application
# os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.settings.production')
# application = ProtocolTypeRouter({
#     "http": get_asgi_application(),
#     "websocket": AuthMiddlewareStack(URLRouter(websocket_urlpatterns)),
# })
```

> **Why:** Use WSGI with Gunicorn for standard HTTP apps. Switch to ASGI with Daphne/Uvicorn only when you need async views, WebSockets, or long-lived connections.

## Custom Management Commands

**Wrong:**
```python
# Putting scripts in random .py files and running them with exec
# scripts/cleanup.py
import django
django.setup()
from myapp.models import Order
Order.objects.filter(status='expired').delete()
# Run with: python scripts/cleanup.py
```

**Correct:**
```python
# apps/orders/management/commands/cleanup_expired_orders.py
from django.core.management.base import BaseCommand
from apps.orders.models import Order


class Command(BaseCommand):
    help = 'Delete orders that have been expired for more than 30 days'

    def add_arguments(self, parser):
        parser.add_argument(
            '--dry-run',
            action='store_true',
            help='Show what would be deleted without deleting',
        )

    def handle(self, *args, **options):
        from django.utils import timezone
        cutoff = timezone.now() - timezone.timedelta(days=30)
        expired = Order.objects.filter(status='expired', updated_at__lt=cutoff)
        count = expired.count()

        if options['dry_run']:
            self.stdout.write(f'Would delete {count} expired orders')
            return

        expired.delete()
        self.stdout.write(self.style.SUCCESS(f'Deleted {count} expired orders'))
```

> **Why:** Management commands integrate with Django's setup, support arguments, and can be scheduled via cron or Celery beat. Random scripts break when settings paths change. Related: since Django 5.2, `manage.py shell` auto-imports all your models (plus `settings` in 6.0) — run it with `--verbosity=2` to see exactly what was imported.
