# Background task best practices covering Celery setup, Django 6.0 native tasks (django.tasks), task definition, retries, routing, periodic tasks, Django-Q, Huey, and idempotency.

## Celery Setup and Configuration

### Proper Celery project structure and settings integration

**Wrong:**
```python
# celery.py in the wrong location, missing autodiscover
from celery import Celery
app = Celery('myproject')
# Forgetting to configure the broker, or hardcoding credentials
app.conf.broker_url = 'redis://localhost:6379/0'
```

**Correct:**
```python
# config/celery.py
import os
from celery import Celery

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.settings.production')

app = Celery('config')
app.config_from_object('django.conf:settings', namespace='CELERY')
app.autodiscover_tasks()


# config/__init__.py
from .celery import app as celery_app
__all__ = ('celery_app',)


# settings.py
CELERY_BROKER_URL = os.environ.get('CELERY_BROKER_URL', 'redis://127.0.0.1:6379/0')
CELERY_RESULT_BACKEND = os.environ.get('CELERY_RESULT_BACKEND', 'redis://127.0.0.1:6379/1')
CELERY_ACCEPT_CONTENT = ['json']
CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'
CELERY_TIMEZONE = 'UTC'
```

> **Why:** `config_from_object` with `namespace='CELERY'` reads all `CELERY_*` settings from Django settings. `autodiscover_tasks` finds `tasks.py` in each installed app automatically.

## Django 6.0 Native Tasks (django.tasks)

### Configuring the right backend per environment

**Wrong:**
```python
# Shipping the built-in backend to production and expecting background work
TASKS = {
    'default': {
        'BACKEND': 'django.tasks.backends.immediate.ImmediateBackend',
    }
}
# ImmediateBackend executes the task inline, in-process, during enqueue() —
# nothing is offloaded, no queue, no worker. Fine for dev, useless in production.
```

**Correct:**
```python
# settings/dev.py — run tasks inline so failures surface immediately
TASKS = {
    'default': {'BACKEND': 'django.tasks.backends.immediate.ImmediateBackend'},
}

# settings/test.py — record enqueued tasks without executing them
TASKS = {
    'default': {'BACKEND': 'django.tasks.backends.dummy.DummyBackend'},
}

# tests.py — assert against the dummy backend
from django.tasks import default_task_backend

def test_signup_enqueues_welcome_email(self):
    self.client.post('/signup/', data)
    result = default_task_backend.results[0]  # results sit in READY state
    assert result.task.name == 'send_welcome_email'
    default_task_backend.clear()

# settings/production.py — a third-party backend with a durable queue + worker,
# e.g. the django-tasks reference package's database backend
TASKS = {
    'default': {'BACKEND': 'django_tasks.backends.database.DatabaseBackend'},
}
```

> **Why:** Django 6.0 ships the tasks *interface* plus two dev-oriented backends: ImmediateBackend runs tasks inline (dev and integration tests), DummyBackend stores results in READY state so tests can assert on `backend.results`. Production offloading requires a third-party backend that provides a durable queue and a worker process.

### Defining, enqueueing, and reading results

**Wrong:**
```python
from myapp.tasks import send_welcome_email

def register(request):
    send_welcome_email(user.id)  # Calling the function directly — runs synchronously
    send_welcome_email.enqueue(user.id)  # Or discarding the result — no way to check status
```

**Correct:**
```python
# myapp/tasks.py
from django.tasks import task

@task(priority=10, queue_name='emails')
def send_welcome_email(user_id): ...

@task(takes_context=True)
def sync_account(context, account_id):
    if context.attempt > 1:  # Context exposes retry metadata
        logger.warning('Retrying account sync, attempt %s', context.attempt)


# views.py
result = send_welcome_email.enqueue(user.id)          # Sync view
result = await send_welcome_email.aenqueue(user.id)   # Async view

# Per-call override without redefining the task
send_welcome_email.using(priority=20, queue_name='urgent').enqueue(user.id)

# Result API
result.id            # Unique id — store it to check on the task later
result.status        # READY / RUNNING / SUCCESSFUL / FAILED
result.refresh()     # Re-fetch state from the backend (or await result.arefresh())
result.return_value  # Available once SUCCESSFUL
result.errors        # On failure: each error has .exception_class and .traceback
```

> **Why:** `@task()` turns a plain function into an enqueueable task; `enqueue()`/`aenqueue()` return a result handle rather than running inline. `takes_context=True` injects a context object (attempt number, task result) and `.using()` overrides priority or queue per call.

### Enqueue after commit with JSON-serializable arguments

**Wrong:**
```python
from django.db import transaction

def place_order(request):
    with transaction.atomic():
        order = Order.objects.create(...)
        process_order.enqueue(order)  # Model instances aren't JSON-serializable — rejected
        process_order.enqueue(order.pk)  # Still wrong: the worker may pick this up
        # before COMMIT, and the row won't exist yet
```

**Correct:**
```python
from functools import partial
from django.db import transaction

def place_order(request):
    with transaction.atomic():
        order = Order.objects.create(...)
        transaction.on_commit(partial(process_order.enqueue, order.pk))
```

> **Why:** Task arguments and return values must be JSON-serializable — tuples come back as lists, and model instances or datetimes are rejected outright — the same "pass IDs, not instances" rule as Celery. `transaction.on_commit` guarantees the row is visible before any worker runs the task. Define tasks in each app's `tasks.py`. Reach for django.tasks for simple offloading with a supported backend; Celery remains the choice for complex routing, rate limits, canvas/chains, and Beat schedules until the native framework matures.

## Task Definition

### Using shared_task with JSON-serializable arguments

**Wrong:**
```python
from celery import Celery
app = Celery()

@app.task
def send_email(user_id):
    from myapp.models import User
    user = User.objects.get(pk=user_id)
    # Passing the whole user object would fail — objects aren't JSON-serializable
```

**Correct:**
```python
from celery import shared_task


@shared_task(bind=True, name='orders.send_confirmation')
def send_order_confirmation(self, order_id):
    from apps.orders.models import Order
    from django.core.mail import send_mail

    order = Order.objects.select_related('user').get(pk=order_id)
    send_mail(
        subject=f'Order #{order.pk} Confirmation',
        message=f'Your order total: ${order.total}',
        from_email='shop@example.com',
        recipient_list=[order.user.email],
    )
```

> **Why:** Use `@shared_task` to avoid importing the Celery app directly. Pass IDs, not objects — task arguments must be JSON-serializable. `bind=True` gives access to `self` for retries.

## Task Retry

### Exponential backoff with autoretry

**Wrong:**
```python
from celery import shared_task

@shared_task
def charge_payment(order_id):
    # No retry logic — if payment gateway is temporarily down, the charge is lost
    import stripe
    order = Order.objects.get(pk=order_id)
    stripe.Charge.create(amount=int(order.total * 100), currency='usd')
```

**Correct:**
```python
from celery import shared_task


@shared_task(
    bind=True,
    autoretry_for=(ConnectionError, TimeoutError),
    retry_backoff=True,
    retry_backoff_max=600,
    max_retries=5,
    retry_jitter=True,
)
def charge_payment(self, order_id):
    from apps.orders.models import Order
    import stripe

    order = Order.objects.get(pk=order_id)
    try:
        charge = stripe.Charge.create(
            amount=int(order.total * 100),
            currency='usd',
            source=order.payment_token,
        )
        order.status = 'paid'
        order.charge_id = charge.id
        order.save(update_fields=['status', 'charge_id'])
    except stripe.error.CardError:
        order.status = 'payment_failed'
        order.save(update_fields=['status'])
        # Don't retry card errors — they're permanent failures
```

> **Why:** `autoretry_for` retries on specific exceptions. `retry_backoff=True` uses exponential backoff. `retry_jitter=True` adds randomness to prevent thundering herd.

## Task Routing

### Separate queues for different task priorities

**Wrong:**
```python
# All tasks on the same queue — a slow report blocks email sending
@shared_task
def send_email(user_id): ...

@shared_task
def generate_large_report(report_id): ...  # Takes 30 minutes, blocks the queue
```

**Correct:**
```python
from celery import shared_task

@shared_task(queue='emails')
def send_email(user_id): ...

@shared_task(queue='reports')
def generate_large_report(report_id): ...

# settings.py
CELERY_TASK_ROUTES = {
    'apps.notifications.tasks.*': {'queue': 'emails'},
    'apps.reports.tasks.*': {'queue': 'reports'},
}

# Start workers per queue:
# celery -A config worker -Q emails -c 4
# celery -A config worker -Q reports -c 2
```

> **Why:** Route tasks to separate queues so slow tasks don't block fast ones. Run dedicated workers per queue with appropriate concurrency.

## Periodic Tasks

### Celery Beat for scheduled tasks

**Wrong:**
```python
# Using a cron job that calls manage.py — outside Celery's control
# crontab: */5 * * * * cd /app && python manage.py cleanup_expired
```

**Correct:**
```python
# settings.py
from celery.schedules import crontab

CELERY_BEAT_SCHEDULE = {
    'cleanup-expired-orders': {
        'task': 'orders.cleanup_expired',
        'schedule': crontab(minute='*/30'),  # Every 30 minutes
    },
    'send-daily-digest': {
        'task': 'notifications.send_digest',
        'schedule': crontab(hour=9, minute=0),  # Daily at 9 AM
    },
    'weekly-report': {
        'task': 'reports.generate_weekly',
        'schedule': crontab(hour=0, minute=0, day_of_week=1),  # Monday midnight
    },
}

# Start beat: celery -A config beat --loglevel=info
```

> **Why:** Celery Beat manages periodic tasks within the Celery ecosystem — monitoring, retries, and logging work the same as regular tasks. Use `django-celery-beat` if you need DB-managed schedules.

## Django-Q Alternative

### Simpler background tasks without separate broker config

**Wrong:**
```python
# Using threading for background tasks in Django
import threading

def my_view(request):
    t = threading.Thread(target=send_email, args=(user.id,))
    t.start()  # No retry, no monitoring, lost if server restarts
    return HttpResponse('ok')
```

**Correct:**
```python
# pip install django-q2

# settings.py
Q_CLUSTER = {
    'name': 'myproject',
    'workers': 4,
    'recycle': 500,
    'timeout': 60,
    'django_redis': 'default',  # Uses Django's cache backend
}

INSTALLED_APPS = [..., 'django_q']

# Usage
from django_q.tasks import async_task, schedule

# Fire and forget
async_task('myapp.tasks.send_email', user.id)

# With callback
async_task('myapp.tasks.process_order', order.id,
           hook='myapp.tasks.order_processed_callback')
```

> **Why:** Django-Q (django-q2 fork) is simpler than Celery — no separate broker config needed if you use Redis as Django's cache. Good for projects that don't need Celery's full feature set. On Django 6.0+, also weigh the native django.tasks framework (see above) before adding a third-party queue.

## Huey Lightweight Alternative

### Minimal background task setup for small projects

**Wrong:**
```python
# Using Celery for a simple project with 2-3 background tasks
# Celery + Redis + Beat + Flower = complex infrastructure for a small app
```

**Correct:**
```python
# pip install huey

# settings.py
INSTALLED_APPS = [..., 'huey.contrib.djhuey']

HUEY = {
    'huey_class': 'huey.RedisHuey',
    'name': 'myproject',
    'immediate': False,  # Set True for development (runs tasks synchronously)
}

# tasks.py
from huey.contrib.djhuey import task, periodic_task, crontab

@task()
def send_email(user_id):
    from myapp.models import User
    user = User.objects.get(pk=user_id)
    # send email...

@periodic_task(crontab(minute='0', hour='*/6'))
def cleanup():
    # runs every 6 hours
    ...

# Start: python manage.py run_huey
```

> **Why:** Huey is a lightweight alternative to Celery — single dependency, simple config. `immediate=True` in development runs tasks synchronously. Great for small-to-medium projects; on Django 6.0+ the native django.tasks framework covers similar simple cases without an extra dependency.

## Task Idempotency

### Ensuring tasks are safe to run more than once

**Wrong:**
```python
from celery import shared_task

@shared_task
def charge_order(order_id):
    order = Order.objects.get(pk=order_id)
    # If this task runs twice (retry, duplicate message), customer is charged twice!
    payment_gateway.charge(order.total)
    order.status = 'paid'
    order.save()
```

**Correct:**
```python
from celery import shared_task


@shared_task(bind=True)
def charge_order(self, order_id):
    from apps.orders.models import Order

    order = Order.objects.select_for_update().get(pk=order_id)

    # Idempotency check — skip if already processed
    if order.status == 'paid':
        return

    if order.charge_id:
        # Already charged but status wasn't updated — verify with gateway
        return

    charge = payment_gateway.charge(order.total, idempotency_key=f'order-{order.id}')
    order.charge_id = charge.id
    order.status = 'paid'
    order.save(update_fields=['charge_id', 'status'])
```

> **Why:** Tasks may run more than once due to retries, broker redelivery, or duplicate sends. Use idempotency keys and status checks to ensure safe re-execution.
