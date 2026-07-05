# Async Django Best Practices

Async view, ORM, and deployment best practices: when async pays off, avoiding event-loop blocking, `sync_to_async` bridging, async auth/sessions (Django 5.1/5.2), and async pagination (Django 6.0).

## When to Go Async

**Wrong:**
```python
# Converting a simple CRUD view to async "for performance"
async def product_detail(request, pk):
    # One DB query, no external I/O — async adds overhead here, not speed
    product = await Product.objects.aget(pk=pk)
    return render(request, 'products/detail.html', {'product': product})
```

**Correct:**
```python
import asyncio
import httpx

async def dashboard(request):
    # I/O-bound fan-out — three external calls run concurrently instead of serially
    async with httpx.AsyncClient() as client:
        weather, stocks, news = await asyncio.gather(
            client.get('https://api.weather.example/today'),
            client.get('https://api.stocks.example/portfolio'),
            client.get('https://api.news.example/headlines'),
        )
    return render(request, 'dashboard.html', {
        'weather': weather.json(),
        'stocks': stocks.json(),
        'news': news.json(),
    })
```

> **Why:** Async views pay off when a request spends its time waiting on external I/O — multiple API calls, streaming, long-polling. CPU-bound work and single-query CRUD gain nothing; each async view still runs one request's logic, just with extra event-loop machinery. Convert views that fan out, not everything.

## Async Views

**Wrong:**
```python
import requests

async def fetch_report(request):
    # Blocking call inside an async view — freezes the event loop
    # for EVERY in-flight request on this worker, not just this one
    data = requests.get('https://api.example.com/report').json()
    user = User.objects.get(pk=request.user.pk)  # Blocking ORM — same problem
    return JsonResponse({'report': data, 'user': user.email})
```

**Correct:**
```python
import httpx

async def fetch_report(request):
    async with httpx.AsyncClient(timeout=10) as client:
        response = await client.get('https://api.example.com/report')
    user = await User.objects.aget(pk=request.user.pk)  # Async ORM method
    return JsonResponse({'report': response.json(), 'user': user.email})
```

> **Why:** Under ASGI all requests on a worker share one event loop. A single blocking call (`requests`, `time.sleep`, sync ORM) stalls every concurrent request until it returns. Inside `async def`, every I/O operation must be awaitable — use `httpx.AsyncClient` and the async ORM API.

## Async ORM

**Wrong:**
```python
async def order_summary(request):
    # Sync ORM call in async context raises SynchronousOnlyOperation
    order = Order.objects.get(pk=1)

    # Iterating a queryset also triggers a sync query
    items = [item.name for item in order.items.all()]
    return JsonResponse({'items': items})
```

**Correct:**
```python
async def order_summary(request, pk):
    order = await Order.objects.aget(pk=pk)

    names = [item.name async for item in order.items.all()]

    total = await Order.objects.acount()
    exists = await Order.objects.filter(status='pending').aexists()

    await Order.objects.acreate(user_id=1, total=99)
    await Order.objects.filter(pk=pk).aupdate(status='shipped')
    await Order.objects.filter(status='cancelled').adelete()

    # For large result sets, stream with a server-side cursor
    async for order in Order.objects.filter(status='new').aiterator(chunk_size=500):
        ...

    return JsonResponse({'items': names, 'total': total, 'pending': exists})
```

> **Why:** Every sync queryset method has an `a`-prefixed twin (`aget`, `acreate`, `aupdate`, `adelete`, `acount`, `aexists`), and querysets support `async for` and `aiterator()`. Note these currently run the query in a thread — there is no async database driver yet — so the win is not blocking the event loop, not faster queries.

## sync_to_async and async_to_sync

**Wrong:**
```python
import asyncio

async def checkout(request):
    # Calling a sync helper directly blocks the loop (or raises if it touches the ORM)
    receipt = generate_receipt(order_id)

    # asyncio.run() inside a running event loop raises RuntimeError
    result = asyncio.run(fetch_prices())
    return JsonResponse({'receipt': receipt, 'prices': result})
```

**Correct:**
```python
from asgiref.sync import sync_to_async, async_to_sync

async def checkout(request, order_id):
    # Wrap legacy sync code; thread_sensitive=True runs it in the shared
    # sync thread, which is required for code that touches the ORM
    receipt = await sync_to_async(generate_receipt, thread_sensitive=True)(order_id)

    # Already inside the loop — just await coroutines directly
    prices = await fetch_prices()
    return JsonResponse({'receipt': receipt, 'prices': prices})


def legacy_sync_view(request):
    # Going the other way: call async code from sync code
    prices = async_to_sync(fetch_prices)()
    return JsonResponse({'prices': prices})
```

> **Why:** `sync_to_async` moves blocking work off the event loop into a thread; `thread_sensitive=True` (the default) keeps ORM access on a single thread so connections behave. Never call `asyncio.run()` when a loop is already running — await the coroutine, or use `async_to_sync` from sync land.

## Async Auth and Sessions (Django 5.1/5.2)

**Wrong:**
```python
async def profile(request):
    # request.user triggers a synchronous DB fetch on first access —
    # blocks the loop inside an async view
    user = request.user

    # Sync session access — same problem
    theme = request.session.get('theme', 'light')
    return JsonResponse({'email': user.email, 'theme': theme})
```

**Correct:**
```python
from django.contrib.auth import aauthenticate, alogin
from django.contrib.auth.decorators import login_required


@login_required  # Async-capable since Django 5.1 — detects the async view
async def profile(request):
    user = await request.auser()  # Async user fetch, cached per request

    theme = await request.session.aget('theme', 'light')
    await request.session.aset('last_seen', now().isoformat())

    return JsonResponse({'email': user.email, 'theme': theme})


async def login_view(request):
    user = await aauthenticate(request, username=..., password=...)
    if user is not None:
        await alogin(request, user)


async def register(request):
    # Django 5.2: async manager and backend entry points
    user = await User.objects.acreate_user('alice', 'a@example.com', 'pw')
```

> **Why:** `request.user` and `request.session[...]` do lazy synchronous DB/session reads that block the loop. Django 5.1+ provides `request.auser()`, the async session API (`aget`, `aset`, etc.), async-capable `@login_required`, and `aauthenticate`/`alogin`; Django 5.2 adds `UserManager.acreate_user()` and `ModelBackend.aauthenticate()`.

## Async Pagination (Django 6.0)

**Wrong:**
```python
from django.core.paginator import Paginator

async def article_list(request):
    paginator = Paginator(Article.objects.all(), 25)
    page = paginator.get_page(request.GET.get('page'))  # Sync count + slice queries
    return render(request, 'articles/list.html', {'page': page})
```

**Correct:**
```python
from django.core.paginator import AsyncPaginator

async def article_list(request):
    paginator = AsyncPaginator(Article.objects.all(), 25)
    page = await paginator.aget_page(request.GET.get('page'))  # Returns AsyncPage
    articles = [article async for article in page]
    return render(request, 'articles/list.html', {'page': page, 'articles': articles})
```

> **Why:** `Paginator.get_page()` runs a COUNT and a slice query synchronously, blocking the loop. Django 6.0's `AsyncPaginator` and `AsyncPage` mirror the sync API with awaitable methods and async iteration.

## Deployment

**Wrong:**
```bash
# Async views deployed under WSGI — each request spins up its own
# event loop, adding overhead with none of the concurrency benefit
gunicorn config.wsgi:application
```

**Correct:**
```bash
# Serve through an ASGI server so async views share a real event loop
uvicorn config.asgi:application --workers 4
# or: daphne config.asgi:application
# or: hypercorn config.asgi:application
```

> **Why:** Async concurrency only materializes under ASGI (uvicorn, daphne, hypercorn); under WSGI Django runs each async view in a throwaway per-request event loop, so you pay the overhead and get zero fan-out benefit. For server setup see references/deployment.md; for WebSockets and long-lived connections see references/channels.md.
