Django URL Routing best practices: path/re_path usage, include(), namespaces, reverse/reverse_lazy, and custom converters.

## path() and re_path()

**Wrong:**
```python
from django.urls import re_path
from . import views

# Using regex when simple path converters would work
urlpatterns = [
    re_path(r'^products/(?P<pk>[0-9]+)/$', views.product_detail),
    re_path(r'^products/(?P<slug>[\w-]+)/$', views.product_by_slug),
]
```

**Correct:**
```python
from django.urls import path, re_path
from . import views

urlpatterns = [
    # path() with built-in converters — cleaner syntax
    path('products/<int:pk>/', views.product_detail, name='detail'),
    path('products/<slug:slug>/', views.product_by_slug, name='by-slug'),

    # re_path() only when you need complex regex
    re_path(r'^archive/(?P<year>[0-9]{4})-(?P<month>[0-9]{2})/$',
            views.archive, name='archive'),
]
```

> **Why:** `path()` is simpler and less error-prone. Only use `re_path()` when you need regex patterns that path converters can't express (like date formats or complex patterns).

## Separating App URLs with include()

**Wrong:**
```python
# config/urls.py — importing all views from all apps
from users.views import login_view, register_view, profile_view
from products.views import product_list, product_detail, product_create

urlpatterns = [
    path('login/', login_view),
    path('register/', register_view),
    path('profile/', profile_view),
    path('products/', product_list),
    path('products/<int:pk>/', product_detail),
    path('products/create/', product_create),
]
```

**Correct:**
```python
# config/urls.py
from django.urls import path, include

urlpatterns = [
    path('', include('apps.pages.urls')),
    path('users/', include('apps.users.urls')),
    path('products/', include('apps.products.urls')),
    path('api/v1/', include('apps.api.urls')),
]

# apps/products/urls.py
from django.urls import path
from . import views

app_name = 'products'
urlpatterns = [
    path('', views.product_list, name='list'),
    path('<int:pk>/', views.product_detail, name='detail'),
    path('create/', views.product_create, name='create'),
]
```

> **Why:** Each app owns its URLs. The root `urls.py` stays clean — just a list of `include()` calls. This makes apps portable and self-contained.

## URL Namespaces

**Wrong:**
```python
# No app_name, no namespaces — name collisions between apps
# users/urls.py
urlpatterns = [
    path('', views.list_view, name='list'),  # Conflicts with products:list
]

# products/urls.py
urlpatterns = [
    path('', views.list_view, name='list'),  # Same name!
]
```

**Correct:**
```python
# users/urls.py
from django.urls import path
from . import views

app_name = 'users'
urlpatterns = [
    path('', views.user_list, name='list'),
    path('<int:pk>/', views.user_detail, name='detail'),
]

# products/urls.py
from django.urls import path
from . import views

app_name = 'products'
urlpatterns = [
    path('', views.product_list, name='list'),
    path('<int:pk>/', views.product_detail, name='detail'),
]

# Usage in templates: {% url 'users:list' %} {% url 'products:detail' pk=1 %}
```

> **Why:** `app_name` creates a namespace so `reverse('users:list')` and `reverse('products:list')` are unambiguous. Required when using `include()` with named URLs.

## reverse() and reverse_lazy()

**Wrong:**
```python
from django.urls import reverse

class ArticleCreateView(CreateView):
    model = Article
    # reverse() is called at import time — before URL config is loaded
    success_url = reverse('articles:list')  # This crashes!
```

**Correct:**
```python
from django.urls import reverse, reverse_lazy
from django.views.generic import CreateView
from .models import Article


class ArticleCreateView(CreateView):
    model = Article
    success_url = reverse_lazy('articles:list')  # Evaluated when needed, not at import

    def get_success_url(self):
        # Or use reverse() inside a method — it's called at request time
        return reverse('articles:detail', kwargs={'pk': self.object.pk})
```

> **Why:** `reverse_lazy()` delays URL resolution until first access. Use it in class attributes and module-level code. `reverse()` is fine inside functions/methods that run at request time.

## reverse() with Query Strings and Fragments (Django 5.2+)

**Wrong:**
```python
from urllib.parse import quote
from django.urls import reverse

# Manual string-building — easy to get encoding wrong
url = f"{reverse('search')}?q={quote(q)}&page=2#results"
# Forget quote() on one value and 'a&b', 'c++', or unicode breaks the URL
```

**Correct:**
```python
from django.urls import reverse, reverse_lazy

url = reverse('search', query={'q': 'django', 'page': 2}, fragment='results')
# /search/?q=django&page=2#results — values URL-encoded for you

class DraftListView(ListView):
    # Works in reverse_lazy() too
    success_url = reverse_lazy('articles:list', query={'status': 'draft'})
```

> **Why:** The `query` and `fragment` arguments (Django 5.2+) handle URL encoding correctly, including special characters and multi-value parameters. Manual f-string concatenation is a recurring source of encoding bugs.

## URL Parameters and Custom Converters

**Wrong:**
```python
from django.urls import path
from . import views

urlpatterns = [
    # Accepting any string then validating in the view
    path('products/<slug>/', views.product_detail, name='detail'),
    # slug converter allows characters you might not want
]
```

**Correct:**
```python
# converters.py
class FourDigitYearConverter:
    regex = '[0-9]{4}'

    def to_python(self, value):
        return int(value)

    def to_url(self, value):
        return f'{value:04d}'


# urls.py
from django.urls import path, register_converter
from . import converters, views

register_converter(converters.FourDigitYearConverter, 'yyyy')

urlpatterns = [
    path('archive/<yyyy:year>/', views.archive_view, name='archive'),
    path('products/<uuid:id>/', views.product_detail, name='detail'),
]
```

> **Why:** Built-in converters (`int`, `str`, `slug`, `uuid`, `path`) handle common cases. Custom converters enforce URL patterns at the routing level, before your view code runs.
