# Django Logging Best Practices

Logging configuration best practices: module-level loggers, the LOGGING dictConfig, Django's built-in loggers, production error visibility, propagation, and lazy formatting without secrets.

## Module-Level Logger, Not print()

**Wrong:**
```python
def process_order(order):
    print(f'Processing order {order.pk}')  # Lost in prod, no level, no timestamp

    import logging
    logging.warning('Payment failed')  # Root logger — no module context
```

**Correct:**
```python
import logging

logger = logging.getLogger(__name__)  # e.g. 'shop.services.orders'


def process_order(order):
    logger.info('Processing order %s', order.pk)
    try:
        charge(order)
    except PaymentError:
        logger.exception('Payment failed for order %s', order.pk)  # Includes traceback
        raise
```

> **Why:** `print()` has no level, timestamp, or routing, and often disappears under production servers. `getLogger(__name__)` names the logger after the module, so records show where they came from and you can tune levels per app (`shop.services`) in `LOGGING`. Use `logger.exception()` inside `except` blocks to capture the traceback.

## LOGGING dictConfig

**Wrong:**
```python
LOGGING = {
    'version': 1,
    'disable_existing_loggers': True,  # Kills Django's own loggers
    'handlers': {
        'file': {
            'class': 'logging.FileHandler',
            'filename': '/var/log/app.log',  # Files in containers vanish on restart
        },
    },
    'root': {'handlers': ['file'], 'level': 'DEBUG'},  # Hardcoded, noisy everywhere
}
```

**Correct:**
```python
import os

LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,  # Always False — keep Django's defaults
    'formatters': {
        'verbose': {
            'format': '{levelname} {asctime} {name} {message}',
            'style': '{',
        },
    },
    'handlers': {
        'console': {  # stdout — the right target for containers/12-factor
            'class': 'logging.StreamHandler',
            'formatter': 'verbose',
        },
    },
    'root': {
        'handlers': ['console'],
        'level': 'WARNING',
    },
    'loggers': {
        'django': {
            'handlers': ['console'],
            'level': os.environ.get('DJANGO_LOG_LEVEL', 'INFO'),
            'propagate': False,
        },
        'myapp': {  # Your project's apps
            'handlers': ['console'],
            'level': os.environ.get('DJANGO_LOG_LEVEL', 'INFO'),
            'propagate': False,
        },
    },
}
```

> **Why:** `disable_existing_loggers: True` silences loggers created before settings load — including Django's — and is almost never what you want. Log to stdout so the platform (Docker, Kubernetes, systemd) handles collection, and drive the level from an env var so you can turn on DEBUG without a deploy.

## Know the Built-In Loggers

**Wrong:**
```python
# Reinventing request/error logging in middleware
class LoggingMiddleware:
    def __call__(self, request):
        response = self.get_response(request)
        if response.status_code >= 400:
            logger.error('Error on %s', request.path)  # django.request already does this
        return response
```

**Correct:**
```python
LOGGING = {
    # ...
    'loggers': {
        # 5xx -> ERROR, 4xx -> WARNING, automatically
        'django.request': {
            'handlers': ['console'],
            'level': 'WARNING',
            'propagate': False,
        },
        # SuspiciousOperation: bad Host headers, tampered sessions, etc.
        'django.security': {
            'handlers': ['console'],
            'level': 'WARNING',
            'propagate': False,
        },
        # Every SQL query at DEBUG level — dev-only, requires DEBUG=True
        'django.db.backends': {
            'handlers': ['console'],
            'level': 'DEBUG' if DEBUG else 'INFO',
            'propagate': False,
        },
        # django.server: runserver's request log (dev only)
    },
}
```

> **Why:** Django already logs unhandled exceptions and 4xx/5xx responses via `django.request`, security rejections via `django.security.*`, and SQL via `django.db.backends`. Configure these loggers instead of duplicating them — and never enable SQL logging in production, it is high-volume and can leak query parameters.

## Production Error Visibility

**Wrong:**
```python
# DEBUG=False with no error reporting — 500s vanish into the void
DEBUG = False
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'root': {'handlers': [], 'level': 'ERROR'},  # Nothing configured
}
```

**Correct:**
```python
# Minimum viable: email tracebacks to ADMINS when DEBUG=False
ADMINS = [('Ops', 'ops@example.com')]
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'filters': {
        'require_debug_false': {'()': 'django.utils.log.RequireDebugFalse'},
    },
    'handlers': {
        'console': {'class': 'logging.StreamHandler'},
        'mail_admins': {
            'class': 'django.utils.log.AdminEmailHandler',
            'level': 'ERROR',
            'filters': ['require_debug_false'],  # Never email in dev
        },
    },
    'loggers': {
        'django.request': {
            'handlers': ['console', 'mail_admins'],
            'level': 'WARNING',
            'propagate': False,
        },
    },
}

# Better: real error monitoring with grouping, context, and rate limiting
import sentry_sdk

sentry_sdk.init(
    dsn=env('SENTRY_DSN'),
    traces_sample_rate=0.1,
    send_default_pii=False,
)
```

> **Why:** With `DEBUG=False`, users see a bare 500 page and you see nothing unless something is capturing errors. `AdminEmailHandler` behind `require_debug_false` is the built-in floor; a monitoring service like Sentry is strictly better — it deduplicates, adds request context, and doesn't drown your inbox during an incident.

## propagate=False to Prevent Double-Logging

**Wrong:**
```python
LOGGING = {
    # ...
    'root': {'handlers': ['console'], 'level': 'INFO'},
    'loggers': {
        'myapp': {
            'handlers': ['console'],
            'level': 'INFO',
            # propagate defaults to True — record hits 'myapp' handler,
            # then bubbles to root and hits 'console' AGAIN: every line twice
        },
    },
}
```

**Correct:**
```python
LOGGING = {
    # ...
    'root': {'handlers': ['console'], 'level': 'WARNING'},
    'loggers': {
        'myapp': {
            'handlers': ['console'],
            'level': 'INFO',
            'propagate': False,  # Handled here; don't bubble to root
        },
        # Alternative: attach handlers ONLY to root and let everything
        # propagate — then named loggers set levels but no handlers
        'noisy.library': {'level': 'WARNING'},  # No handlers, just a level gate
    },
}
```

> **Why:** Records propagate up the logger hierarchy to root by default. If both a named logger and root have handlers, every record is emitted twice. Either set `propagate: False` on loggers that own handlers, or centralize handlers on root and use named loggers purely for level control.

## Lazy Formatting, No Secrets

**Wrong:**
```python
# f-string formats eagerly — the string is built even when DEBUG is off
logger.debug(f'Cart contents: {expensive_serialize(cart)}')

# Logging credentials and PII
logger.info(f'Login attempt: user={email} password={password}')
logger.debug(f'Stripe payload: {request.body}')  # Card data into the logs
```

**Correct:**
```python
# %-style args are only interpolated if the record is actually emitted
logger.debug('Cart contents: %s', cart.pk)
logger.info('User %s logged in', user.pk)

# Log identifiers, never secrets — and gate genuinely expensive extras
if logger.isEnabledFor(logging.DEBUG):
    logger.debug('Cart detail: %s', expensive_serialize(cart))

logger.info('Login attempt for user_id=%s success=%s', user.pk, success)
# Never log: passwords, tokens, session keys, full card numbers,
# Authorization headers, or raw request bodies from payment webhooks
```

> **Why:** With `logger.info("... %s", value)` the formatting cost is paid only when the level is enabled; f-strings always pay it, including any `__str__`/serialization work. Logs are widely readable and long-retained — treat anything written to them as leaked, so log stable identifiers (`user.pk`), never credentials or PII.
