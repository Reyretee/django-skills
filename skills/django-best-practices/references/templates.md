Django Templates best practices: template inheritance, built-in tags and filters, custom template tags, context processors, and template security.

## Template Inheritance

**Wrong:**
```html
<!-- Every template repeats the full HTML structure -->
<!-- products/list.html -->
<!DOCTYPE html>
<html>
<head><title>Products</title><link rel="stylesheet" href="/static/style.css"></head>
<body>
  <nav>...</nav>
  <h1>Products</h1>
  <!-- content -->
  <footer>...</footer>
</body>
</html>

<!-- articles/list.html — same boilerplate duplicated -->
<!DOCTYPE html>
<html>
<head><title>Articles</title><link rel="stylesheet" href="/static/style.css"></head>
<body>
  <nav>...</nav>
  <h1>Articles</h1>
  <!-- content -->
  <footer>...</footer>
</body>
</html>
```

**Correct:**
```html
<!-- templates/base.html -->
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>{% block title %}My Site{% endblock %}</title>
  {% block extra_css %}{% endblock %}
</head>
<body>
  <nav>{% include "includes/nav.html" %}</nav>
  <main>
    {% block content %}{% endblock %}
  </main>
  <footer>{% include "includes/footer.html" %}</footer>
  {% block extra_js %}{% endblock %}
</body>
</html>

<!-- templates/products/list.html -->
{% extends "base.html" %}

{% block title %}Products{% endblock %}

{% block content %}
  <h1>Products</h1>
  {% for product in products %}
    <div>{{ product.name }}</div>
  {% endfor %}
{% endblock %}
```

> **Why:** Template inheritance eliminates duplication. `base.html` defines the skeleton, child templates override specific blocks. Use `{% include %}` for reusable fragments.

## Template Partials (Django 6.0)

**Wrong:**
```django
<!-- todos/_todo_item.html — a separate include file for every snippet -->
<li id="todo-{{ todo.pk }}">{{ todo.title }}</li>

<!-- todos/list.html — fragment lives far from where it's used -->
{% for todo in todos %}
  {% include "todos/_todo_item.html" %}
{% endfor %}
```

```python
# views.py — returning the full page to an HTMX request
def toggle_todo(request, pk):
    todo = toggle(pk)
    return render(request, 'todos/list.html', {'todos': Todo.objects.all()})
    # HTMX only wanted the one <li> — the client swaps in a whole page
```

**Correct:**
```django
<!-- todos/list.html — fragment defined inline, right where it's used -->
{% partialdef todo-item %}
  <li id="todo-{{ todo.pk }}">{{ todo.title }}</li>
{% endpartialdef %}

{% block content %}
  <ul>
    {% for todo in todos %}
      {% partial todo-item %}
    {% endfor %}
  </ul>
{% endblock %}
```

```python
# views.py — partials are addressable as "template.html#partial-name"
# in render(), get_template(), and {% include %}
def toggle_todo(request, pk):
    todo = toggle(pk)
    if request.headers.get('HX-Request'):
        # Return just the fragment for the AJAX request
        return render(request, 'todos/list.html#todo-item', {'todo': todo})
    # Full page otherwise
    return render(request, 'todos/list.html', {'todos': Todo.objects.all()})
```

> **Why:** Partials keep a fragment's markup in one place — no separate `_fragment.html` file per snippet, no duplicated markup — and make it addressable as `"list.html#todo-item"`, the canonical HTMX/AJAX pattern of returning just the fragment for partial-page swaps. On Django < 6.0, the `django-template-partials` package provides the same syntax.

## Template Tags

**Wrong:**
```html
<!-- Hardcoded URLs and static paths -->
<a href="/products/{{ product.id }}/">{{ product.name }}</a>
<img src="/static/images/logo.png">

<!-- No empty list handling -->
{% for item in items %}
  <li>{{ item.name }}</li>
{% endfor %}
```

**Correct:**
```html
{% load static %}

<!-- Named URLs — won't break when URL patterns change -->
<a href="{% url 'products:detail' pk=product.pk %}">{{ product.name }}</a>
<img src="{% static 'images/logo.png' %}" alt="Logo">

<!-- Handle empty lists -->
{% for item in items %}
  <li>{{ item.name }}</li>
{% empty %}
  <li>No items found.</li>
{% endfor %}

<!-- Use {% with %} to avoid repeated expensive lookups -->
{% with total=order.get_total %}
  <p>Total: ${{ total }}</p>
  <p>Tax: ${{ total|floatformat:2 }}</p>
{% endwith %}
```

> **Why:** `{% url %}` and `{% static %}` generate correct URLs regardless of deployment config. `{% empty %}` handles the no-results case. `{% with %}` caches computed values. Inside loops, use `forloop.counter`, `forloop.first`/`forloop.last`, and `forloop.length` (Django 6.0) instead of computing positions in the view.

## {% querystring %} (Django 5.1+)

**Wrong:**
```django
<!-- Hand-building the query string drops the user's active filters -->
<a href="?page={{ page_obj.next_page_number }}">Next</a>
<!-- On /products/?q=shoes&sort=price, clicking Next loses q and sort -->
```

**Correct:**
```django
<!-- Preserves all current GET params, replaces only page -->
<a href="{% querystring page=page_obj.next_page_number %}">Next</a>

<!-- Remove a parameter by setting it to None -->
<a href="{% querystring q=None %}">Clear search</a>
```

> **Why:** `{% querystring %}` merges into the current request's GET parameters instead of replacing them, and handles URL encoding for you. As of Django 6.0 the output always starts with `?` and the tag also accepts multiple dict arguments.

## Template Filters

**Wrong:**
```html
<!-- Formatting in the view instead of the template -->
<!-- views.py: context['date'] = obj.created_at.strftime('%B %d, %Y') -->
<p>{{ date }}</p>

<!-- Truncating in Python -->
<!-- views.py: context['desc'] = obj.description[:100] + '...' -->
<p>{{ desc }}</p>
```

**Correct:**
```html
<!-- Built-in filters handle formatting -->
<p>{{ article.created_at|date:"F j, Y" }}</p>
<p>{{ article.description|truncatewords:20 }}</p>
<p>{{ article.body|linebreaks }}</p>
<p>{{ price|floatformat:2 }}</p>
<p>{{ name|default:"Anonymous" }}</p>
<p>{{ count|pluralize:"y,ies" }}</p>

<!-- django.contrib.humanize (add to INSTALLED_APPS) for friendly formatting -->
{% load humanize %}
<p>{{ view_count|intcomma }} views</p>
<p>Posted {{ article.created_at|naturaltime }}</p>
```

> **Why:** Template filters keep presentation logic in templates where it belongs. The view provides raw data, the template formats it for display. `django.contrib.humanize` adds human-friendly filters like `naturaltime` ("3 minutes ago") and `intcomma` ("1,234,567").

## Custom Template Tags

**Wrong:**
```python
# Passing computed data through every single view's context
# views.py
def product_list(request):
    return render(request, 'products/list.html', {
        'products': Product.objects.all(),
        'cart_count': request.session.get('cart', {}).items().__len__(),  # Repeated everywhere
    })
```

**Correct:**
```python
# templatetags/shop_tags.py
from django import template
from django.utils.safestring import mark_safe

register = template.Library()


@register.simple_tag(takes_context=True)
def cart_count(context):
    request = context['request']
    cart = request.session.get('cart', {})
    return len(cart)


@register.inclusion_tag('includes/recent_articles.html')
def recent_articles(count=5):
    from articles.models import Article
    return {'articles': Article.objects.order_by('-created_at')[:count]}


@register.filter
def currency(value):
    return f'${value:,.2f}'


@register.simple_block_tag
def alert(content, level='info'):
    # Django 5.2+ — tags that wrap content, no full Node class needed
    return mark_safe(f'<div class="alert alert-{level}">{content}</div>')
```

```html
{% load shop_tags %}
<span>Cart: {% cart_count %}</span>
{% recent_articles 3 %}
<p>{{ product.price|currency }}</p>
{% alert level="warning" %}Only {{ stock }} left in stock!{% endalert %}
```

> **Why:** Custom tags and filters encapsulate reusable template logic. `simple_tag` for computed values, `inclusion_tag` for reusable template fragments, `filter` for value transformation, and `simple_block_tag` (Django 5.2+) for tags that wrap a block of rendered content — previously that required writing a full `Node` class.

## Context Processors

**Wrong:**
```python
# Adding site_name to every view manually
def home(request):
    return render(request, 'home.html', {'site_name': 'My Site', ...})

def about(request):
    return render(request, 'about.html', {'site_name': 'My Site', ...})
```

**Correct:**
```python
# context_processors.py
from django.conf import settings


def site_context(request):
    return {
        'site_name': settings.SITE_NAME,
        'support_email': settings.SUPPORT_EMAIL,
    }


# settings.py
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
                'myapp.context_processors.site_context',  # Custom
            ],
        },
    },
]
```

> **Why:** Context processors inject variables into every template automatically. Use them for truly global data (site name, feature flags) — not for view-specific data.

## Template Security

**Wrong:**
```html
<!-- Disabling autoescaping carelessly -->
{% autoescape off %}
  {{ user_comment }}  <!-- XSS vulnerability if comment contains <script> -->
{% endautoescape %}

{{ user_bio|safe }}  <!-- Also dangerous — marks raw HTML as safe -->
```

**Correct:**
```html
<!-- Django autoescapes by default — trust it -->
<p>{{ user_comment }}</p>  <!-- <script> tags are escaped automatically -->

<!-- Only use |safe for content YOU control, never user input -->
{{ admin_announcement|safe }}  <!-- Only if set by trusted staff in admin -->

<!-- For user-generated HTML, use bleach to sanitize first -->
<!-- In the view: cleaned = bleach.clean(user_input, tags=['b', 'i', 'a']) -->
<p>{{ cleaned_html|safe }}</p>
```

> **Why:** Django's autoescaping prevents XSS by default. Never use `|safe` or `{% autoescape off %}` on user-supplied content. Sanitize HTML with a library like bleach before marking it safe.

## Custom Error Pages

**Wrong:**
```html
<!-- No custom templates — users see Django's bare default 404/500 pages
     in production -->

<!-- Or: templates/500.html depending on context processors -->
{% extends "base.html" %}
{% block content %}
  <h1>{{ site_name }} is having trouble</h1>  <!-- Empty — never rendered -->
  <p>Contact {{ support_email }}</p>          <!-- Also empty -->
{% endblock %}
```

**Correct:**
```html
<!-- templates/404.html and templates/500.html are used automatically
     when DEBUG=False — no configuration needed -->

<!-- templates/500.html — must be fully self-contained -->
<!DOCTYPE html>
<html lang="en">
<head><title>Server Error</title></head>
<body>
  <h1>Something went wrong</h1>
  <p>We've been notified and are looking into it.</p>
</body>
</html>
```

```python
# config/urls.py — handlers only needed for custom *views*
# (extra context, content negotiation); not for plain templates
handler404 = 'myapp.views.custom_404'
handler500 = 'myapp.views.custom_500'
```

> **Why:** Django picks up `templates/404.html` and `templates/500.html` automatically when `DEBUG=False`; `handler404`/`handler500` in the root urls.py are only for custom view logic. The 500 template is rendered with an empty context — no context processors run — so it must not reference `{{ request }}`, `{{ site_name }}`, or anything else injected by them.
