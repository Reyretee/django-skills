# Deployment and performance best practices covering Gunicorn, uWSGI, Daphne, Nginx, Docker, environment variables, CI/CD, zero-downtime deployment, health checks, query optimization, caching, and async views.

## Gunicorn Configuration

### Production-ready Gunicorn setup

**Wrong:**
```bash
# Running Django's development server in production
python manage.py runserver 0.0.0.0:8000
# Or Gunicorn with default settings
gunicorn config.wsgi
```

**Correct:**
```python
# gunicorn.conf.py
import multiprocessing

bind = '0.0.0.0:8000'
workers = multiprocessing.cpu_count() * 2 + 1
worker_class = 'gthread'  # Or 'gevent' for high-concurrency
threads = 4
timeout = 30
keepalive = 5
max_requests = 1000
max_requests_jitter = 50
accesslog = '-'
errorlog = '-'
loglevel = 'info'
```

```bash
gunicorn config.wsgi:application -c gunicorn.conf.py
```

> **Why:** `workers = 2 * CPU + 1` is the rule of thumb. `max_requests` recycles workers to prevent memory leaks. `gthread` gives threads within workers for I/O-bound apps.

## uWSGI Alternative

### Properly configured uWSGI with socket mode

**Wrong:**
```bash
# Running uWSGI without proper configuration
uwsgi --http :8000 --module config.wsgi
# Missing worker management, logging, and process recycling
```

**Correct:**
```ini
; uwsgi.ini
[uwsgi]
module = config.wsgi:application
master = true
processes = 4
threads = 2
socket = /tmp/uwsgi.sock
chmod-socket = 660
vacuum = true
die-on-term = true
max-requests = 5000
harakiri = 30
```

```bash
uwsgi --ini uwsgi.ini
```

> **Why:** uWSGI is a mature alternative to Gunicorn. Use socket mode behind Nginx. `harakiri` kills stuck workers. `max-requests` prevents memory leaks. Gunicorn is simpler; uWSGI is more configurable.

## Daphne (ASGI)

### ASGI servers for WebSocket and async support

**Wrong:**
```bash
# Using Gunicorn for a Django Channels project
gunicorn config.wsgi  # WebSocket connections will fail
```

**Correct:**
```bash
# For ASGI apps (Channels, async views, WebSocket)
pip install daphne

# Run with Daphne
daphne -b 0.0.0.0 -p 8000 config.asgi:application

# Or with Uvicorn (faster)
pip install uvicorn
uvicorn config.asgi:application --host 0.0.0.0 --port 8000 --workers 4
```

> **Why:** ASGI servers (Daphne, Uvicorn) handle WebSocket and async views. Uvicorn is faster; Daphne is the official Channels server. Use ASGI only when you need async features.

## Nginx Configuration

### Nginx as reverse proxy with static file serving

**Wrong:**
```nginx
# Proxying everything through Django, including static files
server {
    location / {
        proxy_pass http://127.0.0.1:8000;
    }
    # Static files served by Django — slow
}
```

**Correct:**
```nginx
upstream django {
    server 127.0.0.1:8000;
}

server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    client_max_body_size 10M;

    location /static/ {
        alias /app/staticfiles/;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    location /media/ {
        alias /app/media/;
        expires 7d;
    }

    location / {
        proxy_pass http://django;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

> **Why:** Nginx serves static/media files directly (fast, with caching headers). Django only handles dynamic requests. `X-Forwarded-Proto` lets Django know it's behind HTTPS.

## Docker

### Multi-stage Dockerfile with docker-compose

**Wrong:**
```dockerfile
# Single-stage Dockerfile with development dependencies
FROM python:3.12
COPY . /app
RUN pip install -r requirements.txt  # Includes dev dependencies
CMD python manage.py runserver 0.0.0.0:8000  # Development server!
```

**Correct:**
```dockerfile
# Multi-stage Dockerfile
FROM python:3.12-slim AS base
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
WORKDIR /app

FROM base AS builder
RUN pip install --no-cache-dir pip-tools
COPY requirements/production.txt .
RUN pip install --no-cache-dir -r production.txt

FROM base AS production
COPY --from=builder /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY --from=builder /usr/local/bin /usr/local/bin
COPY . .
RUN python manage.py collectstatic --noinput
EXPOSE 8000
CMD ["gunicorn", "config.wsgi:application", "-c", "gunicorn.conf.py"]
```

```yaml
# docker-compose.yml
services:
  web:
    build: .
    ports: ["8000:8000"]
    env_file: .env
    depends_on: [db, redis]
  db:
    image: postgres:16
    volumes: [postgres_data:/var/lib/postgresql/data]
    environment:
      POSTGRES_DB: myapp
      POSTGRES_PASSWORD: ${DB_PASSWORD}
  redis:
    image: redis:7-alpine
  celery:
    build: .
    command: celery -A config worker -l info
    env_file: .env
    depends_on: [db, redis]

volumes:
  postgres_data:
```

> **Why:** Multi-stage builds keep images small (no build tools in production). docker-compose defines the full stack. Never use `runserver` in production containers.

## Environment Variables

### Using django-environ for configuration

**Wrong:**
```python
# Hardcoding configuration in settings
SECRET_KEY = 'hardcoded-secret'
DATABASE_URL = 'postgres://user:pass@localhost/db'
```

**Correct:**
```python
# pip install django-environ

# settings/base.py
import environ

env = environ.Env(
    DEBUG=(bool, False),
    ALLOWED_HOSTS=(list, []),
)
environ.Env.read_env(BASE_DIR / '.env')

SECRET_KEY = env('SECRET_KEY')
DEBUG = env('DEBUG')
ALLOWED_HOSTS = env('ALLOWED_HOSTS')
DATABASES = {'default': env.db('DATABASE_URL')}
EMAIL_CONFIG = env.email('EMAIL_URL', default='consolemail://')
CACHES = {'default': env.cache('CACHE_URL', default='locmemcache://')}
```

```bash
# .env
SECRET_KEY=your-random-secret-key
DEBUG=False
ALLOWED_HOSTS=example.com,www.example.com
DATABASE_URL=postgres://user:pass@db:5432/myapp
CACHE_URL=redis://redis:6379/0
EMAIL_URL=smtp://user:pass@smtp.example.com:587
```

> **Why:** django-environ parses DATABASE_URL, CACHE_URL, etc. into Django's dict format. Keep `.env` out of git. Default values keep development simple.

## CI/CD (GitHub Actions)

### Automated testing pipeline with service containers

**Wrong:**
```yaml
# No CI — tests only run manually (or never)
```

**Correct:**
```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: postgres
        ports: ["5432:5432"]
        options: >-
          --health-cmd pg_isready --health-interval 10s --health-timeout 5s --health-retries 5
      redis:
        image: redis:7
        ports: ["6379:6379"]

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -r requirements/test.txt
      - run: python manage.py test --parallel
        env:
          DATABASE_URL: postgres://postgres:postgres@localhost/test_db
          DJANGO_SETTINGS_MODULE: config.settings.test
          SECRET_KEY: test-secret-key
```

> **Why:** CI runs tests on every push. Service containers provide Postgres and Redis. `--parallel` speeds up the test suite. Fail the pipeline before merging broken code.

## Zero-Downtime Deployment

### Graceful reload with backward-compatible migrations

**Wrong:**
```bash
# Stop server, deploy, start server — downtime!
systemctl stop gunicorn
git pull
pip install -r requirements.txt
python manage.py migrate
systemctl start gunicorn
```

**Correct:**
```bash
#!/bin/bash
# deploy.sh — zero-downtime deployment
set -e

# 1. Pull new code
git pull origin main

# 2. Install dependencies
pip install -r requirements/production.txt

# 3. Run migrations (must be backward-compatible)
python manage.py migrate --noinput

# 4. Collect static files
python manage.py collectstatic --noinput

# 5. Graceful reload — finishes existing requests, starts new workers
kill -s HUP $(cat /tmp/gunicorn.pid)
# Or with systemd: systemctl reload gunicorn
```

> **Why:** Gunicorn's HUP signal spawns new workers with new code while old workers finish serving current requests. Migrations must be backward-compatible — new and old code run simultaneously.

## Health Check Endpoint

### Load balancer health check with database verification

**Wrong:**
```python
# No health check — load balancer has no way to know if the app is healthy
# Or: checking only if Django responds, not if the database is reachable
```

**Correct:**
```python
# views.py
from django.db import connection
from django.http import JsonResponse


def health_check(request):
    try:
        with connection.cursor() as cursor:
            cursor.execute("SELECT 1")
        status = 'healthy'
        code = 200
    except Exception as e:
        status = f'unhealthy: {e}'
        code = 503

    return JsonResponse({
        'status': status,
        'database': 'ok' if code == 200 else 'error',
    }, status=code)


# urls.py — no auth required
from django.urls import path
urlpatterns = [
    path('health/', health_check, name='health-check'),
]
```

> **Why:** Load balancers need a health endpoint to route traffic. Check the database connection at minimum. Return 503 when unhealthy so the load balancer stops sending traffic.

## Error Monitoring

### Capturing exceptions when DEBUG=False

**Wrong:**
```python
# DEBUG=False with no monitoring — errors return a bare 500 page and vanish;
# you learn about outages from angry users
DEBUG = False
```

**Correct:**
```python
# pip install sentry-sdk

# settings/production.py
import sentry_sdk

sentry_sdk.init(
    dsn=env('SENTRY_DSN'),
    traces_sample_rate=0.1,  # Sample 10% of transactions for performance data
    send_default_pii=False,
)
```

> **Why:** With `DEBUG=False` you are blind without error monitoring — Sentry (or an equivalent like Rollbar or self-hosted GlitchTip) captures the traceback, request data, and release info for every unhandled exception. Pair it with proper logging configuration — see `references/logging.md`.

## Django Debug Toolbar

### Development-only profiling setup

**Wrong:**
```python
# No profiling tool — guessing where performance problems are
# Or: Debug Toolbar enabled in production
INSTALLED_APPS = [
    'debug_toolbar',  # In base settings — loaded in production!
]
```

**Correct:**
```python
# settings/local.py only
INSTALLED_APPS += ['debug_toolbar']
MIDDLEWARE.insert(0, 'debug_toolbar.middleware.DebugToolbarMiddleware')
INTERNAL_IPS = ['127.0.0.1']

# urls.py
from django.conf import settings
if settings.DEBUG:
    import debug_toolbar
    urlpatterns = [
        path('__debug__/', include(debug_toolbar.urls)),
    ] + urlpatterns
```

> **Why:** Debug Toolbar shows SQL queries, template rendering time, cache hits, and signals per request. Only install in development — it exposes internal data and slows requests.

## General Query Optimization Principles

### Efficient querying patterns

**Wrong:**
```python
# Fetching all fields when you only need a few
users = User.objects.all()
names = [u.name for u in users]  # Loads all columns for each user

# Using .count() just to check existence
if User.objects.filter(email=email).count() > 0:
    ...
```

**Correct:**
```python
# Fetch only needed fields
names = User.objects.values_list('name', flat=True)

# Use .exists() instead of .count() > 0
if User.objects.filter(email=email).exists():
    ...

# Use .iterator() for large querysets to avoid loading all into memory
for user in User.objects.all().iterator(chunk_size=2000):
    process(user)
```

> **Why:** `values_list` avoids instantiating model objects. `exists()` stops at the first match (faster than count). `iterator()` streams results instead of loading all into memory.

## N+1 Detection and Solution

### Solving N+1 queries with select_related and prefetch_related

**Wrong:**
```python
def get_orders(request):
    orders = Order.objects.all()
    for order in orders:
        print(order.customer.name)       # N+1
        print(order.items.count())        # N+1
        for item in order.items.all():    # N+1
            print(item.product.name)      # N*M+1
```

**Correct:**
```python
from django.db.models import Count, Prefetch

def get_orders(request):
    orders = (
        Order.objects
        .select_related('customer')           # JOIN for FK
        .prefetch_related(
            Prefetch('items', queryset=OrderItem.objects.select_related('product'))
        )                                      # 2 queries for M2M + FK
        .annotate(item_count=Count('items'))   # Aggregate in SQL
    )
    for order in orders:
        print(order.customer.name)       # No extra query
        print(order.item_count)          # No extra query
        for item in order.items.all():   # No extra query
            print(item.product.name)     # No extra query
```

> **Why:** `select_related` = SQL JOIN (FK/O2O). `prefetch_related` = separate query + Python join (M2M/reverse FK). `Prefetch` object lets you chain `select_related` on the prefetched queryset.

## Database Indexing

### Strategic index placement for common query patterns

**Wrong:**
```python
class Order(models.Model):
    status = models.CharField(max_length=20)
    created_at = models.DateTimeField(auto_now_add=True)
    customer = models.ForeignKey('Customer', on_delete=models.CASCADE)
    # No indexes — queries on status and created_at do full table scans
```

**Correct:**
```python
from django.db import models


class Order(models.Model):
    status = models.CharField(max_length=20, db_index=True)
    created_at = models.DateTimeField(auto_now_add=True)
    customer = models.ForeignKey('Customer', on_delete=models.CASCADE)

    class Meta:
        indexes = [
            models.Index(fields=['status', 'created_at']),       # Composite index
            models.Index(fields=['-created_at']),                 # Descending index
            models.Index(
                fields=['status'],
                condition=models.Q(status='pending'),
                name='pending_orders_idx',                        # Partial index
            ),
        ]
```

> **Why:** Index columns used in `WHERE`, `ORDER BY`, and `JOIN`. Composite indexes match multi-column queries. Partial indexes are smaller and faster for common filtered queries.

## Caching Strategies

### Multi-level caching for different use cases

**Wrong:**
```python
# Caching at the wrong level — too broad or too narrow
@cache_page(3600)
def dashboard(request):
    # Entire page cached for 1 hour — user sees stale data
    ...
```

**Correct:**
```python
from django.core.cache import cache
from django.views.decorators.cache import cache_page


# Level 1: Query-level cache for expensive computations
def get_dashboard_stats():
    stats = cache.get('dashboard_stats')
    if stats is None:
        stats = Order.objects.aggregate(
            total=Sum('total'),
            count=Count('id'),
        )
        cache.set('dashboard_stats', stats, timeout=300)
    return stats


# Level 2: Fragment cache in templates for expensive partials
# {% cache 300 sidebar_widgets user.pk %}...{% endcache %}

# Level 3: Full page cache for public, static pages
@cache_page(60 * 60)
def about_page(request):
    return render(request, 'about.html')
```

> **Why:** Cache at the most granular level that makes sense. Query caching for expensive DB operations. Fragment caching for expensive template partials. Page caching for fully static pages.

## Deferred Fields

### Reducing memory usage with only(), defer(), and values()

**Wrong:**
```python
# Loading a TextField with 100KB of content when you only need the title
articles = Article.objects.all()
for article in articles:
    print(article.title)  # Also loaded: body (100KB), metadata, etc.
```

**Correct:**
```python
# only() — fetch only specified fields
articles = Article.objects.only('id', 'title', 'created_at')
for article in articles:
    print(article.title)  # Only these fields are loaded
    # article.body  # Would trigger an additional query (lazy load)

# defer() — fetch everything except specified fields
articles = Article.objects.defer('body', 'metadata')

# values() — returns dicts, not model instances (fastest)
articles = Article.objects.values('id', 'title')
```

> **Why:** `only()` and `defer()` reduce memory usage for large text/binary fields. `values()` skips model instantiation entirely. Use when you don't need the full model.

## Connection Pooling

### Reusing database connections for better performance

**Wrong:**
```python
# Django opens and closes a DB connection per request by default
# Under heavy load, this creates connection overhead
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'mydb',
        # No connection pooling
    }
}
```

**Correct:**
```python
# Option 1: Native connection pooling (Django 5.1+)
# Requires psycopg 3 with the pool extra: pip install "psycopg[binary,pool]"
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'mydb',
        'OPTIONS': {
            'pool': {
                'min_size': 2,
                'max_size': 4,
                'timeout': 10,
            },
        },
        # Don't combine 'pool' with CONN_MAX_AGE — the pool manages
        # connection lifetimes itself
    }
}

# Option 2: Persistent connections via CONN_MAX_AGE (any Django version)
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'mydb',
        'CONN_MAX_AGE': 60,          # Keep connections open for 60s
        'CONN_HEALTH_CHECKS': True,  # Verify the connection before reuse
    }
}

# Option 3: PgBouncer (external connection pooler)
# pgbouncer.ini:
# [databases]
# mydb = host=127.0.0.1 port=5432 dbname=mydb
# [pgbouncer]
# pool_mode = transaction
# max_client_conn = 1000
# default_pool_size = 20

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'HOST': '127.0.0.1',
        'PORT': '6432',  # PgBouncer port
    }
}
```

> **Why:** Connection pooling reuses database connections instead of opening/closing per request. Native pooling (Django 5.1+, psycopg 3 with `psycopg[pool]`) keeps warm connections inside each worker process — a modern complement or alternative to PgBouncer, which pools across many app servers. Pick one strategy per database: never combine `pool` with `CONN_MAX_AGE`.

## SQLite in Production

### Tuning SQLite for concurrent writes

**Wrong:**
```python
# Default SQLite settings under concurrent writes
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'db.sqlite3',
    }
}
# Rollback-journal mode: writers block readers — "database is locked" errors
```

**Correct:**
```python
# Django 5.1+ OPTIONS for the small-scale SQLite-in-production pattern
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'db.sqlite3',
        'OPTIONS': {
            'init_command': (
                'PRAGMA journal_mode=WAL;'
                'PRAGMA busy_timeout=5000;'
            ),
            'transaction_mode': 'IMMEDIATE',
        },
    }
}
```

> **Why:** WAL mode lets readers proceed while a write is in progress, `busy_timeout` makes writers wait for the lock instead of failing immediately, and `IMMEDIATE` transactions take the write lock up front to avoid mid-transaction lock errors. This makes single-server SQLite viable for modest production workloads.

## Async Views (Django 4.1+)

### Concurrent external API calls with async views

**Wrong:**
```python
# Using sync views for I/O-bound operations
import requests

def external_api_view(request):
    # Blocks the worker thread while waiting for external API
    response1 = requests.get('https://api1.example.com/data')
    response2 = requests.get('https://api2.example.com/data')
    # Sequential — takes sum of both response times
    return JsonResponse({...})
```

**Correct:**
```python
# Django 4.1+ async views
import asyncio
import httpx
from django.http import JsonResponse


async def external_api_view(request):
    async with httpx.AsyncClient() as client:
        # Concurrent — takes max of both response times
        response1, response2 = await asyncio.gather(
            client.get('https://api1.example.com/data'),
            client.get('https://api2.example.com/data'),
        )
    return JsonResponse({
        'api1': response1.json(),
        'api2': response2.json(),
    })
```

> **Why:** Async views don't block the worker thread during I/O waits. Use `asyncio.gather()` for concurrent external API calls. Requires ASGI server (Uvicorn/Daphne).
