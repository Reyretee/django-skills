# Django Pagination Best Practices

Django pagination best practices: Paginator in function-based views, ordering before pagination, ListView `paginate_by`, filter-preserving template links, elided page ranges, large-table performance, and async pagination.

## Paginator in Function-Based Views

**Wrong:**
```python
def article_list(request):
    # Manual slicing — no page metadata, no bounds checking
    offset = int(request.GET.get('offset', 0))
    articles = Article.objects.all()[offset:offset + 25]

    # Or: using page() with raw user input — raises on bad pages
    paginator = Paginator(Article.objects.all(), 25)
    page = paginator.page(request.GET.get('page'))  # PageNotAnInteger / EmptyPage
```

**Correct:**
```python
from django.core.paginator import Paginator


def article_list(request):
    articles = Article.objects.order_by('-published_at')
    paginator = Paginator(articles, 25)
    # get_page() clamps: non-integers -> page 1, out-of-range -> last page
    page_obj = paginator.get_page(request.GET.get('page'))
    return render(request, 'articles/list.html', {'page_obj': page_obj})
```

> **Why:** `paginator.page(number)` raises `PageNotAnInteger` or `EmptyPage` on invalid input, so `?page=abc` becomes a 500 unless you catch both. `get_page()` handles user-supplied page numbers gracefully. Manual slicing loses `has_next`, `num_pages`, and all page metadata.

## Always Order Before Paginating

**Wrong:**
```python
# No order_by() — the database may return rows in any order
paginator = Paginator(Product.objects.filter(is_active=True), 20)
# Django emits UnorderedObjectListWarning; rows can repeat or vanish
# between pages as the database reorders results
```

**Correct:**
```python
paginator = Paginator(
    Product.objects.filter(is_active=True).order_by('-created_at', 'pk'),
    20,
)
```

> **Why:** Without `ORDER BY`, the database returns rows in arbitrary order, so pages overlap or skip items nondeterministically — Django warns with `UnorderedObjectListWarning`. Add `pk` as a tiebreaker so rows with equal sort values still have a stable order.

## ListView with paginate_by

**Wrong:**
```python
class ArticleListView(ListView):
    model = Article

    # Reimplementing pagination by hand in get_context_data
    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        page = int(self.request.GET.get('page', 1))
        context['articles'] = Article.objects.all()[(page - 1) * 25:page * 25]
        return context
```

**Correct:**
```python
from django.views.generic import ListView


class ArticleListView(ListView):
    model = Article
    paginate_by = 25
    ordering = ['-published_at']
    # Context automatically includes: page_obj, paginator,
    # is_paginated, and the paginated object_list
```

```django
{% for article in object_list %}
  <h2>{{ article.title }}</h2>
{% endfor %}

{% if is_paginated %}
  Page {{ page_obj.number }} of {{ page_obj.paginator.num_pages }}
{% endif %}
```

> **Why:** `paginate_by` gives you validated pagination, 404s on bad pages, and `page_obj`/`paginator`/`is_paginated` in the context for free. Hand-rolled slicing in `get_context_data` bypasses all of it.

## Pagination Links That Preserve Filters

**Wrong:**
```django
<!-- Drops every existing GET param: search, filters, sort all reset -->
<a href="?page={{ page_obj.next_page_number }}">Next</a>
```

**Correct:**
```django
{# Django 5.1+ — querystring keeps existing params, overrides only page #}
{% if page_obj.has_previous %}
  <a href="{% querystring page=page_obj.previous_page_number %}">Previous</a>
{% endif %}

<span>Page {{ page_obj.number }} of {{ page_obj.paginator.num_pages }}</span>

{% if page_obj.has_next %}
  <a href="{% querystring page=page_obj.next_page_number %}">Next</a>
{% endif %}
```

> **Why:** `?page=2` replaces the whole querystring, so `?q=django&page=2` becomes `?page=2` and the user loses their search. The `{% querystring %}` tag merges the new `page` value into the current GET params.

## Elided Page Ranges for Long Lists

**Wrong:**
```django
<!-- 500 pages = 500 links rendered on every request -->
{% for num in page_obj.paginator.page_range %}
  <a href="{% querystring page=num %}">{{ num }}</a>
{% endfor %}
```

**Correct:**
```python
def article_list(request):
    paginator = Paginator(Article.objects.order_by('-published_at'), 25)
    page_obj = paginator.get_page(request.GET.get('page'))
    # Yields e.g. 1, 2, '…', 41, 42, 43, '…', 99, 100
    elided_range = paginator.get_elided_page_range(
        page_obj.number, on_each_side=2, on_ends=1,
    )
    return render(request, 'articles/list.html', {
        'page_obj': page_obj,
        'elided_range': elided_range,
    })
```

```django
{% for num in elided_range %}
  {% if num == page_obj.paginator.ELLIPSIS %}
    <span>{{ num }}</span>
  {% else %}
    <a href="{% querystring page=num %}">{{ num }}</a>
  {% endif %}
{% endfor %}
```

> **Why:** `get_elided_page_range()` collapses long page lists into a windowed range with ellipses, keeping navigation usable and templates fast regardless of page count.

## Large Tables: count() Cost and Keyset Pagination

**Wrong:**
```python
# Deep OFFSET forces the database to scan and discard 100,000 rows
Article.objects.order_by('-id')[100000:100025]

# And every Paginator page triggers a COUNT(*) over the whole table
paginator = Paginator(Article.objects.order_by('-id'), 25)  # count on millions of rows
```

**Correct:**
```python
# Keyset (cursor) pagination — seek by the last-seen key, no OFFSET
def article_feed(request):
    qs = Article.objects.order_by('-id')
    last_id = request.GET.get('after')
    if last_id:
        qs = qs.filter(id__lt=last_id)
    articles = list(qs[:25])
    next_cursor = articles[-1].id if len(articles) == 25 else None
    return render(request, 'articles/feed.html', {
        'articles': articles,
        'next_cursor': next_cursor,
    })


# If you must keep Paginator, avoid the exact COUNT(*) on huge tables
class FastCountPaginator(Paginator):
    @cached_property
    def count(self):
        return 1_000_000  # Approximate/cached count (e.g. from pg_class.reltuples)
```

> **Why:** `OFFSET 100000` makes the database walk past 100,000 rows before returning any, and `Paginator` adds a full `COUNT(*)` per request. Keyset pagination filters on an indexed column, so page 4,000 costs the same as page 1. DRF's `CursorPagination` implements this pattern for APIs.

## Async Views: AsyncPaginator

**Wrong:**
```python
async def article_list(request):
    paginator = Paginator(Article.objects.order_by('-id'), 25)
    # Synchronous count()/slicing runs blocking DB queries in an async view
    page = paginator.get_page(request.GET.get('page'))  # SynchronousOnlyOperation risk
```

**Correct:**
```python
# Django 6.0+
from django.core.paginator import AsyncPaginator


async def article_list(request):
    paginator = AsyncPaginator(Article.objects.order_by('-id'), 25)
    # Async counterparts: apage(), aget_page(), acount(), anum_pages()
    page_obj = await paginator.aget_page(request.GET.get('page'))
    return render(request, 'articles/list.html', {'page_obj': page_obj})
```

> **Why:** `Paginator` runs blocking ORM queries (`COUNT`, slicing), which raise `SynchronousOnlyOperation` or stall the event loop inside async views. `AsyncPaginator` exposes `aget_page()`/`apage()` that await the queries properly.
