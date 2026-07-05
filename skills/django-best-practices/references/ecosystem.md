Django ecosystem reference covering essential third-party packages (DRF, django-filter, allauth, debug toolbar, storages, channels, celery results, guardian, import-export, unfold, environ, redis, Pillow) and built-in contrib apps people tend to reinvent (sitemaps, syndication feeds, redirects).

## 25. Django Ecosystem

### Django REST Framework

**Wrong:**
```python
# Building a REST API from scratch with JsonResponse
from django.http import JsonResponse

def api_products(request):
    products = list(Product.objects.values())
    return JsonResponse(products, safe=False)
    # No serialization, no validation, no pagination, no auth
```

**Correct:**
```python
# pip install djangorestframework

# settings.py
INSTALLED_APPS = [..., 'rest_framework']
REST_FRAMEWORK = {
    'DEFAULT_PERMISSION_CLASSES': ['rest_framework.permissions.IsAuthenticated'],
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 20,
}
```

> **Why:** DRF provides serialization, validation, authentication, permissions, pagination, throttling, and a browsable API. Don't rebuild these from scratch.

### Django Filter

**Wrong:**
```python
# Manual query parameter parsing for filtering
def product_list(request):
    qs = Product.objects.all()
    if request.GET.get('min_price'):
        qs = qs.filter(price__gte=request.GET['min_price'])
    # Repeated for every filter...
```

**Correct:**
```python
# pip install django-filter

# settings.py
INSTALLED_APPS = [..., 'django_filters']

# filters.py
import django_filters
from .models import Product

class ProductFilter(django_filters.FilterSet):
    class Meta:
        model = Product
        fields = {
            'price': ['gte', 'lte'],
            'category': ['exact'],
            'name': ['icontains'],
        }
```

> **Why:** django-filter generates filter forms and querysets declaratively. Integrates with both DRF and standard Django views.

### Django Allauth

**Wrong:**
```python
# Implementing OAuth from scratch with requests-oauthlib
# Handling token exchange, user creation, email verification manually
```

**Correct:**
```python
# pip install django-allauth

# settings.py
INSTALLED_APPS = [
    ...,
    'allauth',
    'allauth.account',
    'allauth.socialaccount',
    'allauth.socialaccount.providers.google',
    'allauth.socialaccount.providers.github',
]

AUTHENTICATION_BACKENDS = [
    'django.contrib.auth.backends.ModelBackend',
    'allauth.account.auth_backends.AuthenticationBackend',
]

ACCOUNT_EMAIL_REQUIRED = True
ACCOUNT_EMAIL_VERIFICATION = 'mandatory'
ACCOUNT_AUTHENTICATION_METHOD = 'email'

# urls.py
urlpatterns = [
    path('accounts/', include('allauth.urls')),
]
```

> **Why:** Allauth handles email/password registration, email verification, password reset, and 50+ OAuth providers. Battle-tested security you shouldn't rewrite.

### Django Debug Toolbar

**Wrong:**
```python
# Printing queries manually
from django.db import connection
print(len(connection.queries))
```

**Correct:**
```python
# pip install django-debug-toolbar

# settings/local.py
INSTALLED_APPS += ['debug_toolbar']
MIDDLEWARE.insert(0, 'debug_toolbar.middleware.DebugToolbarMiddleware')
INTERNAL_IPS = ['127.0.0.1']

# urls.py (development only)
if settings.DEBUG:
    import debug_toolbar
    urlpatterns = [path('__debug__/', include(debug_toolbar.urls))] + urlpatterns
```

> **Why:** Shows SQL queries, duplicates, time per query, template rendering, cache operations, and signals — all in a sidebar panel. Essential for finding N+1 queries.

### Django Storages

**Wrong:**
```python
# Writing custom S3 integration per project
import boto3
# ... 50 lines of upload/download/delete code
```

**Correct:**
```python
# pip install django-storages boto3

# settings.py — S3
STORAGES = {
    'default': {'BACKEND': 'storages.backends.s3boto3.S3Boto3Storage'},
}
AWS_STORAGE_BUCKET_NAME = 'my-bucket'

# For Google Cloud Storage:
# pip install django-storages google-cloud-storage
# STORAGES = {'default': {'BACKEND': 'storages.backends.gcloud.GoogleCloudStorage'}}
# GS_BUCKET_NAME = 'my-bucket'

# For Azure:
# pip install django-storages azure-storage-blob
# STORAGES = {'default': {'BACKEND': 'storages.backends.azure_storage.AzureStorage'}}
# AZURE_CONTAINER = 'my-container'
```

> **Why:** django-storages provides a unified API for S3, GCS, Azure, and more. Your models and forms work unchanged regardless of storage backend.

### Django Channels

**Wrong:**
```python
# Using polling for real-time features
# setInterval(fetch('/api/messages/'), 2000)
```

**Correct:**
```python
# pip install channels channels-redis

# settings.py
INSTALLED_APPS = [..., 'channels']
ASGI_APPLICATION = 'config.asgi.application'
CHANNEL_LAYERS = {
    'default': {
        'BACKEND': 'channels_redis.core.RedisChannelLayer',
        'CONFIG': {'hosts': [('127.0.0.1', 6379)]},
    },
}
```

> **Why:** Channels adds WebSocket, HTTP long-polling, and background task support to Django. Use it for chat, notifications, live updates, and any real-time feature.

### Django Celery Results

**Wrong:**
```python
# No way to check if a Celery task completed or what it returned
result = my_task.delay(data)
# result.get() blocks — defeats the purpose of async tasks
```

**Correct:**
```python
# pip install django-celery-results

# settings.py
INSTALLED_APPS = [..., 'django_celery_results']
CELERY_RESULT_BACKEND = 'django-db'  # Store results in Django's database

# Usage
result = my_task.delay(data)
# Later — check status without blocking
from django_celery_results.models import TaskResult
task = TaskResult.objects.get(task_id=result.id)
print(task.status, task.result)
```

> **Why:** django-celery-results stores task results in the database, making them queryable via Django ORM. Useful for tracking task status in admin or dashboards.

### Django Guardian

**Wrong:**
```python
# Implementing object-level permissions from scratch
class Article(models.Model):
    owner = models.ForeignKey(User, on_delete=models.CASCADE)

def can_edit(user, article):
    return user == article.owner  # Doesn't scale for teams, shared access
```

**Correct:**
```python
# pip install django-guardian

# settings.py
INSTALLED_APPS = [..., 'guardian']
AUTHENTICATION_BACKENDS = [
    'django.contrib.auth.backends.ModelBackend',
    'guardian.backends.ObjectPermissionBackend',
]

# Usage
from guardian.shortcuts import assign_perm, get_objects_for_user

assign_perm('change_article', user, article)
assign_perm('view_article', team_group, article)

# Check: user.has_perm('change_article', article)
# Query: get_objects_for_user(user, 'articles.view_article')
```

> **Why:** Django Guardian provides per-object permissions (user X can edit article Y). Works with Django's built-in permission system and DRF.

### Django Import Export

**Wrong:**
```python
# Writing custom CSV import/export views for admin
import csv
from django.http import HttpResponse

def export_products(request):
    response = HttpResponse(content_type='text/csv')
    writer = csv.writer(response)
    for p in Product.objects.all():
        writer.writerow([p.name, p.price])
    return response
```

**Correct:**
```python
# pip install django-import-export

# admin.py
from import_export import resources
from import_export.admin import ImportExportModelAdmin
from .models import Product


class ProductResource(resources.ModelResource):
    class Meta:
        model = Product
        fields = ('id', 'name', 'price', 'category__name', 'is_active')


@admin.register(Product)
class ProductAdmin(ImportExportModelAdmin):
    resource_class = ProductResource
    list_display = ('name', 'price', 'is_active')
```

> **Why:** django-import-export adds CSV/Excel/JSON import and export buttons to the admin. Handles validation, preview, and rollback on import errors.

### Django Unfold

**Wrong:**
```python
# Using the default Django admin UI for a client-facing admin panel
# The default admin looks dated and lacks modern UX features
```

**Correct:**
```python
# pip install django-unfold

# settings.py
INSTALLED_APPS = [
    'unfold',  # Must be before django.contrib.admin
    'django.contrib.admin',
    ...
]

# admin.py
from unfold.admin import ModelAdmin

@admin.register(Product)
class ProductAdmin(ModelAdmin):
    list_display = ('name', 'price', 'is_active')
```

> **Why:** Django Unfold provides a modern, responsive admin UI with dark mode, improved navigation, and better form layouts — with minimal code changes.

### django-environ / python-decouple

**Wrong:**
```python
# Hardcoded settings, no environment variable support
SECRET_KEY = 'hardcoded'
DEBUG = True
```

**Correct:**
```python
# django-environ (more Django-specific)
# pip install django-environ
import environ
env = environ.Env()
environ.Env.read_env()
DATABASES = {'default': env.db()}

# python-decouple (simpler, framework-agnostic)
# pip install python-decouple
from decouple import config, Csv
SECRET_KEY = config('SECRET_KEY')
DEBUG = config('DEBUG', default=False, cast=bool)
ALLOWED_HOSTS = config('ALLOWED_HOSTS', cast=Csv())
```

> **Why:** Both read from `.env` files and environment variables. django-environ has Django-specific helpers (`env.db()`, `env.cache()`). python-decouple is simpler and framework-agnostic.

### django-redis

**Wrong:**
```python
# Using Django's built-in Redis backend without connection pooling
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.redis.RedisCache',
        'LOCATION': 'redis://localhost:6379',
    }
}
```

**Correct:**
```python
# pip install django-redis

CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/0',
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
            'CONNECTION_POOL_KWARGS': {'max_connections': 50},
        },
    }
}

# Can also be used as session backend
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
SESSION_CACHE_ALIAS = 'default'
```

> **Why:** django-redis provides connection pooling, key prefixing, compression, and can serve as both cache and session backend. More production-ready than Django's built-in Redis backend.

### Pillow

**Wrong:**
```python
# Using ImageField without Pillow installed
class Photo(models.Model):
    image = models.ImageField(upload_to='photos/')
    # ImportError: No module named 'PIL'
```

**Correct:**
```python
# pip install Pillow

from django.db import models

class Photo(models.Model):
    image = models.ImageField(upload_to='photos/%Y/%m/')

    def save(self, *args, **kwargs):
        super().save(*args, **kwargs)
        # Resize on upload
        from PIL import Image
        img = Image.open(self.image.path)
        if img.width > 1920 or img.height > 1080:
            img.thumbnail((1920, 1080))
            img.save(self.image.path)
```

> **Why:** Pillow is required for `ImageField`. It validates that uploaded files are actual images and provides image processing (resize, crop, format conversion).

### Sitemaps (django.contrib.sitemaps)

**Wrong:**
```python
# Hand-writing sitemap.xml as a static file or a manual view
def sitemap_view(request):
    urls = [f"<url><loc>https://example.com{p.get_absolute_url()}</loc></url>"
            for p in Post.objects.all()]
    return HttpResponse(f"<urlset>{''.join(urls)}</urlset>",
                        content_type="application/xml")
    # No lastmod, wrong namespace, regenerated on every request
```

**Correct:**
```python
# settings.py
INSTALLED_APPS = [..., 'django.contrib.sitemaps', 'django.contrib.sites']

# sitemaps.py
from django.contrib.sitemaps import Sitemap
from blog.models import Post

class PostSitemap(Sitemap):
    changefreq = 'weekly'
    priority = 0.6

    def items(self):
        return Post.objects.filter(status=Post.Status.PUBLISHED)

    def lastmod(self, obj):
        return obj.updated_at

# urls.py
from django.contrib.sitemaps.views import sitemap
from django.views.decorators.cache import cache_page

sitemaps = {'posts': PostSitemap}
urlpatterns = [
    path('sitemap.xml', cache_page(3600)(sitemap), {'sitemaps': sitemaps},
         name='django.contrib.sitemaps.views.sitemap'),
]
```

> **Why:** The sitemaps framework generates a correct, namespaced sitemap.xml from your querysets, uses `get_absolute_url()` and `lastmod` automatically, and paginates past 50,000 URLs. Cache the view — search engine crawlers hit it repeatedly.

### Syndication Feeds (django.contrib.syndication)

**Wrong:**
```python
# Building RSS XML by hand with string templates
def rss_view(request):
    items = ''.join(f'<item><title>{p.title}</title></item>'
                    for p in Post.objects.all()[:20])  # Unescaped titles → broken XML
    return HttpResponse(f'<rss><channel>{items}</channel></rss>')
```

**Correct:**
```python
from django.contrib.syndication.views import Feed
from blog.models import Post

class LatestPostsFeed(Feed):
    title = 'My Blog'
    link = '/blog/'
    description = 'Latest posts.'

    def items(self):
        return Post.objects.filter(status=Post.Status.PUBLISHED)[:20]

    def item_title(self, item):
        return item.title

    def item_description(self, item):
        return item.summary

# urls.py
path('feed/', LatestPostsFeed(), name='post-feed')
```

> **Why:** The Feed class handles XML escaping, RSS/Atom formats, enclosures, and item links via `get_absolute_url()`. Hand-rolled feed XML breaks on the first title containing `&` or `<`.

### Legacy URLs (django.contrib.redirects)

**Wrong:**
```python
# Hardcoding old→new URL redirects as views, one per moved page
def old_about(request):
    return redirect('/company/about/', permanent=True)
```

**Correct:**
```python
# settings.py
INSTALLED_APPS = [..., 'django.contrib.sites', 'django.contrib.redirects']
MIDDLEWARE = [..., 'django.contrib.redirects.middleware.RedirectFallbackMiddleware']
# Then manage redirects as data in the admin — no deploy needed per redirect
```

> **Why:** The redirects app stores old→new URL mappings in the database and issues 301s via middleware only when a 404 would occur. Content editors can preserve SEO for moved pages without code changes.
