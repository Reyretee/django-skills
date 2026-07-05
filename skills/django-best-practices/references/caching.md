# Django Caching - Best practices for configuring and using Django's cache framework including Redis, Memcached, and invalidation strategies.

## Cache Framework Configuration

**Wrong:**
```python
# No cache configuration — every cache.get() silently returns None
# Or using LocMemCache in production (not shared between workers)
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.locmem.LocMemCache',
    }
}
```

**Correct:**
```python
# settings/local.py — dummy cache for development
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.dummy.DummyCache',
    }
}

# settings/production.py — real cache
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': os.environ['REDIS_URL'],
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
        },
    }
}
```

> **Why:** `DummyCache` in development means caching never hides bugs. `LocMemCache` is per-process — useless with multiple Gunicorn workers. Use Redis or Memcached in production.

## Redis Cache Backend

**Wrong:**
```python
# Using the built-in Redis backend without connection pooling
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.redis.RedisCache',
        'LOCATION': 'redis://localhost:6379',
        # No timeout, no key prefix, no error handling
    }
}
```

**Correct:**
```python
# pip install django-redis

CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': os.environ.get('REDIS_URL', 'redis://127.0.0.1:6379/0'),
        'TIMEOUT': 300,  # Default 5 minutes
        'KEY_PREFIX': 'myapp',
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
            'SOCKET_CONNECT_TIMEOUT': 5,
            'SOCKET_TIMEOUT': 5,
            'IGNORE_EXCEPTIONS': True,  # Cache failures don't crash the app
            'CONNECTION_POOL_KWARGS': {'max_connections': 50},
        },
    }
}
```

> **Why:** django-redis provides connection pooling, key prefixing, and error handling. `IGNORE_EXCEPTIONS=True` means cache failures degrade gracefully instead of crashing.

## Memcached

**Wrong:**
```python
# Using Memcached without understanding its limitations
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.memcached.PyMemcacheCache',
        'LOCATION': '127.0.0.1:11211',
    }
}
# Then storing objects larger than 1MB or expecting persistence
```

**Correct:**
```python
# pip install pymemcache

CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.memcached.PyMemcacheCache',
        'LOCATION': '127.0.0.1:11211',
        'OPTIONS': {
            'no_delay': True,
            'connect_timeout': 3,
            'timeout': 3,
        },
    }
}

# Use Redis instead of Memcached when you need:
# - Persistence, data structures (lists, sets), pub/sub, or values > 1MB
```

> **Why:** Memcached is fast and simple but has a 1MB value limit, no persistence, and limited data types. Redis is more versatile. Use Memcached only if you already have the infrastructure.

## Per-View Cache

**Wrong:**
```python
from django.views.decorators.cache import cache_page

# Caching a view that depends on the current user
@cache_page(60 * 15)
def dashboard(request):
    return render(request, 'dashboard.html', {
        'user': request.user,  # All users see the same cached page!
    })
```

**Correct:**
```python
from django.views.decorators.cache import cache_page
from django.views.decorators.vary import vary_on_cookie


@cache_page(60 * 15)  # Cache for 15 minutes
def product_list(request):
    # Public page — safe to cache for all users
    products = Product.objects.filter(is_active=True)
    return render(request, 'products/list.html', {'products': products})


@cache_page(60 * 5)
@vary_on_cookie  # Different cache per session/user
def dashboard(request):
    return render(request, 'dashboard.html', {'user': request.user})
```

> **Why:** `@cache_page` caches the entire response. Use it for public pages. For user-specific pages, add `@vary_on_cookie` so each user gets their own cache entry.

## HTTP Caching & Conditional Processing

**Wrong:**
```python
# Recomputing and re-sending a large response on every request,
# even though the underlying data hasn't changed
def report(request, pk):
    report = Report.objects.get(pk=pk)
    data = build_expensive_report(report)  # Rebuilt every time
    return JsonResponse(data)  # Full payload re-sent even when unchanged
```

**Correct:**
```python
from django.views.decorators.http import condition, etag, last_modified


def report_last_modified(request, pk):
    # Cheap query — just the timestamp, not the whole report
    return Report.objects.values_list('updated_at', flat=True).get(pk=pk)


@condition(last_modified_func=report_last_modified)
def report(request, pk):
    # Only runs when the client's copy is stale — otherwise Django returns 304
    report = Report.objects.get(pk=pk)
    return JsonResponse(build_expensive_report(report))


# @etag / @last_modified are shortcuts when you only need one validator
@last_modified(report_last_modified)
def report_v2(request, pk):
    ...
```

```python
# Per-view Cache-Control headers for browsers and CDNs
from django.views.decorators.cache import cache_control, never_cache
from django.views.decorators.vary import vary_on_cookie, vary_on_headers


@cache_control(max_age=3600, public=True)  # Browsers/CDNs may cache for 1 hour
def pricing_page(request):
    ...


@never_cache  # Cache-Control: no-store — for sensitive or always-fresh pages
def account_settings(request):
    ...


@cache_control(max_age=300, private=True)
@vary_on_cookie  # Cache key includes the Cookie header — one entry per user
def dashboard(request):
    ...


@vary_on_headers('Accept-Language')  # Separate cache entries per language
def localized_page(request):
    ...
```

> **Why:** Server-side caching (above) saves work on your server; HTTP caching saves the round trip entirely. `@condition` sends a cheap 304 Not Modified when the client's copy is current — derive `Last-Modified` or an ETag from a lightweight query instead of rebuilding the response. `@cache_control` sets Cache-Control for browsers and CDNs; `@never_cache` opts sensitive pages out. Caution: caching authenticated responses without a `Vary` header leaks one user's page to another — any user-specific cached response needs `@vary_on_cookie` (or `private`/`no-store`).

## Template Fragment Cache

**Wrong:**
```html
<!-- Caching a fragment that includes user-specific content -->
{% load cache %}
{% cache 500 sidebar %}
  <div>Welcome, {{ user.username }}</div>  <!-- Same for all users! -->
  {% for article in recent_articles %}
    <a href="{{ article.url }}">{{ article.title }}</a>
  {% endfor %}
{% endcache %}
```

**Correct:**
```html
{% load cache %}

<!-- Cache per user using a vary argument -->
{% cache 300 sidebar user.pk %}
  <div>Welcome, {{ user.username }}</div>
  {% for article in user_articles %}
    <a href="{{ article.url }}">{{ article.title }}</a>
  {% endfor %}
{% endcache %}

<!-- Or cache only the expensive, non-personalized part -->
{% cache 600 recent_articles %}
  {% for article in recent_articles %}
    <a href="{% url 'articles:detail' pk=article.pk %}">{{ article.title }}</a>
  {% endfor %}
{% endcache %}
```

> **Why:** Template fragment caching is more granular than view caching. Pass a varying argument (like `user.pk`) to create per-user cache entries. Cache only expensive, non-personalized fragments.

## Low-Level Cache API

**Wrong:**
```python
from django.core.cache import cache

def get_product(pk):
    product = cache.get(pk)  # Generic key — collides with other cached objects
    if not product:
        product = Product.objects.get(pk=pk)
        cache.set(pk, product)  # No timeout — cached forever
    return product
```

**Correct:**
```python
from django.core.cache import cache
from .models import Product


def get_product(pk):
    cache_key = f'product:{pk}'
    product = cache.get(cache_key)
    if product is None:
        product = Product.objects.get(pk=pk)
        cache.set(cache_key, product, timeout=300)  # 5 minutes
    return product


# get_or_set — shortcut for the pattern above
def get_product_v2(pk):
    return cache.get_or_set(
        f'product:{pk}',
        lambda: Product.objects.get(pk=pk),
        timeout=300,
    )


# Cache with versioning
cache.set('product:1', data, version=2)
cache.get('product:1', version=2)
```

> **Why:** Use namespaced cache keys (`product:1`) to avoid collisions. Always set a timeout. `get_or_set` is a convenient shortcut for the cache-aside pattern.

## Cache Invalidation Strategies

**Wrong:**
```python
# Never invalidating — stale data served forever
cache.set('product_list', products)  # Never updated when products change

# Or invalidating everything on any change
cache.clear()  # Nukes the entire cache
```

**Correct:**
```python
from django.core.cache import cache
from django.db.models.signals import post_save, post_delete
from django.dispatch import receiver
from .models import Product


@receiver([post_save, post_delete], sender=Product)
def invalidate_product_cache(sender, instance, **kwargs):
    cache.delete(f'product:{instance.pk}')
    cache.delete('product_list')  # Invalidate the list too


# Or use cache versioning for bulk invalidation
def get_product_list():
    version = cache.get('product_list_version', 1)
    return cache.get_or_set(
        'product_list',
        lambda: list(Product.objects.all()),
        timeout=600,
        version=version,
    )

def invalidate_product_list():
    cache.incr('product_list_version')  # Old version naturally expires
```

> **Why:** Targeted invalidation deletes only affected keys. Version-based invalidation avoids thundering herd — old entries expire naturally while new ones are created.

## Cache Key Design

**Wrong:**
```python
# Ambiguous keys that collide across models/apps
cache.set('list', products)
cache.set('detail', product)
cache.set(str(pk), product)
```

**Correct:**
```python
from django.core.cache import cache


def make_cache_key(model_name, identifier, suffix=''):
    """Consistent cache key format: app:model:id:suffix"""
    key = f'myapp:{model_name}:{identifier}'
    if suffix:
        key += f':{suffix}'
    return key


# Usage
cache.set(make_cache_key('product', pk), product, timeout=300)
cache.set(make_cache_key('product', 'list', 'page:1'), page_data, timeout=60)
cache.set(make_cache_key('user', user_id, 'dashboard'), dashboard_data, timeout=120)
```

> **Why:** Structured cache keys prevent collisions, make debugging easier, and allow pattern-based invalidation. Include the app/model name, identifier, and any variant suffixes.
