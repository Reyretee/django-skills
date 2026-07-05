---
name: django-best-practices
description: >
  Django development best practices skill. Use this skill whenever working on
  Django projects — including creating models, writing views, building DRF APIs,
  designing URL patterns, writing tests, configuring settings, handling migrations,
  or reviewing Django code for correctness and performance. Also trigger for
  questions about ORM queries, N+1 problems, select_related/prefetch_related,
  authentication, permissions, middleware, signals, caching, Celery tasks,
  deployment, Docker, or Django project structure. If the task involves any
  Django or Django REST Framework code, activate this skill immediately.
compatibility: Designed for Claude Code. Requires Python and Django.
allowed-tools: Read Write Edit Bash(python:*) Bash(pip:*)
---

# Django Best Practices

A senior Django developer's knowledge base covering 28 topic areas — from ORM queries to deployment, current through Django 6.0. This skill provides wrong vs correct code examples and architectural guidance for building production-grade Django applications.

## When to use this skill

- Creating or modifying Django models, fields, or relations
- Writing views (FBV, CBV, or DRF ViewSets)
- Building or updating DRF serializers and API endpoints
- Writing ORM queries, fixing N+1 problems, or optimizing performance
- Creating or reviewing migrations
- Configuring Django settings, middleware, or URL routing
- Implementing authentication, permissions, or session management
- Setting up caching (Redis, Memcached, per-view, fragment)
- Writing or reviewing tests (Django TestCase, pytest, factories)
- Working with Django signals, forms, templates, or template tags
- Configuring static/media files, file uploads, or storage backends
- Setting up background jobs (Celery or Django 6.0's native django.tasks)
- Deploying Django (Gunicorn, Nginx, Docker, CI/CD)
- Working with Django Channels or WebSockets
- Writing async views or using the async ORM
- Implementing i18n/l10n or timezone support
- Reviewing code for security issues (CSRF, XSS, SQL injection, CSP)
- Paginating querysets, sending email, configuring logging, or flash messages
- Designing app architecture, service layers, or project structure

## Workflow

When this skill is activated, follow these steps:

1. **Identify the topic** — Determine which area of Django the task involves (models, views, queries, etc.)
2. **Read the relevant reference file** — Consult the reference table below and read the appropriate file before writing any code
3. **Apply the correct patterns** — Follow the "Correct" patterns from the reference file, avoid the "Wrong" anti-patterns
4. **Check for common mistakes** — Review the "Why" explanations to understand the reasoning behind each pattern
5. **Cross-reference related topics** — If the task spans multiple areas (e.g., views + queries + templates), read all relevant reference files

> Always read the relevant reference file before writing code for that topic.

## Key Principles

### Project Structure
- Separate config from apps. Use `config/settings/` with base, local, and production files.
- Keep apps small and focused (3-8 models per app). Split when an app exceeds 15 models.
- Always set `AUTH_USER_MODEL` before the first migration.
- Reference users via `settings.AUTH_USER_MODEL`, never `auth.User` directly.

### Models & ORM
- Use `DecimalField` for money, `BooleanField` for flags, `EmailField` for emails.
- Never `null=True` on `CharField`/`TextField` — use `blank=True` alone for optional strings.
- Use `TextChoices`/`IntegerChoices` for enumerated fields — never magic strings.
- Always set explicit `related_name` on ForeignKey and OneToOneField.
- Use `select_related` for ForeignKey/OneToOne, `prefetch_related` for ManyToMany/reverse FK.
- Use `F()` expressions for atomic updates to avoid race conditions.
- Use `Exists()` instead of `.count() > 0` for existence checks.
- Use `bulk_create` and `bulk_update` for batch operations.
- Wrap multi-step writes in `transaction.atomic()`.

### Queries
- Never filter in Python when the database can do it.
- Use `Q` objects for OR/NOT queries.
- Use `annotate` with `Subquery` instead of N+1 loops.
- Always parameterize raw SQL — never use f-strings.
- Use `.values_list()` when you don't need full model instances.
- Use `.exists()` instead of `.count() > 0`.
- Use `.iterator()` for large querysets.

### Views
- Use FBVs for simple one-off views, CBVs for CRUD with generics.
- Always use `LoginRequiredMixin` or `@login_required` for protected views.
- Override `get_queryset()` to enforce ownership in UpdateView/DeleteView.
- Use `get_object_or_404` in views, `.first()` when absence is expected.

### Forms & Validation
- Always use explicit `fields` in ModelForm — never `exclude` or `__all__`.
- Use `clean_<field>()` for single-field validation, `clean()` for cross-field.
- Use Django's built-in validators and write custom ones for domain rules.

### Security
- Never disable CSRF except for external webhooks with their own auth.
- Never use `mark_safe()` or `|safe` on user input — use `format_html()`.
- Never use f-strings in raw SQL — always parameterize.
- Use Argon2 for password hashing, PBKDF2 as fallback.
- Set `SECURE_SSL_REDIRECT`, `SECURE_HSTS_*`, and cookie security flags in production.
- Configure a Content Security Policy (`SECURE_CSP`, built-in since Django 6.0).
- Use `django.core.signing` for expiring tokens — never hand-rolled hashes.
- Rate-limit login and move the admin off `/admin/` on internet-facing sites.
- Run `manage.py check --deploy` before every deployment.

### Auth
- Prefer `LoginRequiredMiddleware` (Django 5.1+) with `@login_not_required` opt-outs over per-view mixins — secure by default.

### Admin
- Always configure `list_display`, `list_filter`, and `search_fields`.
- Use `list_select_related` to prevent N+1 queries in list views.
- Use `readonly_fields` for computed/automatic values.
- Override permission methods to restrict what staff can do.

### Templates
- Use template inheritance with `base.html` and `{% block %}` tags.
- Always use `{% url %}` and `{% static %}` — never hardcode paths.
- Use `{% empty %}` in for loops and `{% with %}` for expensive lookups.
- Never use `{% autoescape off %}` or `|safe` on user content.

### Migrations
- Make small, focused migrations with descriptive names.
- Always provide reverse operations for custom/data migrations.
- Use `apps.get_model()` in data migrations, never direct model imports.
- Multi-step approach for column removal: make nullable → deploy → remove.
- Use `--plan` to preview migrations before running in production.

### Testing
- Use `SimpleTestCase` for no-DB tests, `TestCase` for DB tests.
- Use `setUpTestData` for class-level test data (faster than `setUp`).
- Use factory_boy instead of JSON fixtures.
- Mock external services (SMS, payments, APIs).
- Use `@override_settings` and Django's `locmem` email backend for testing.
- Aim for 80%+ coverage on business logic.

### DRF
- Always use explicit `fields` in serializers — never `__all__`.
- Use separate read/write serializers for nested data.
- Set `IsAuthenticated` as the global default permission.
- Always paginate list endpoints.
- Use `django-filter` for declarative filtering.
- Optimize `get_queryset()` with `select_related`/`prefetch_related`.

### Signals
- Import signals in `AppConfig.ready()`, never at module level.
- Use `dispatch_uid` to prevent duplicate connections.
- Prefer explicit method calls over signals for tightly coupled logic.
- Use signals only for decoupled cross-app communication.

### Caching
- Use `DummyCache` in development, Redis in production.
- Use namespaced cache keys with timeouts.
- Invalidate targeted keys, never `cache.clear()`.
- Cache at the most granular level that makes sense.

### Deployment
- Never use `runserver` in production — use Gunicorn or Uvicorn.
- Use multi-stage Docker builds.
- Store secrets in environment variables, never in code.
- Use zero-downtime deployment with Gunicorn HUP signal.
- Always have a health check endpoint.

### Background Tasks
- Pass IDs to tasks, never model instances.
- Always implement retry logic with exponential backoff.
- Make tasks idempotent — safe to run more than once.
- Use `@shared_task` instead of importing the Celery app directly.
- Consider Django 6.0's native `django.tasks` before adding Celery for simple offloading.
- Enqueue from inside transactions via `transaction.on_commit()`.

### Everyday Essentials
- Paginate with `paginator.get_page()` and always `order_by()` first; preserve filters in links with `{% querystring %}`.
- Send email from background tasks with per-environment backends; always include a plain-text part.
- Configure `LOGGING` with `disable_existing_loggers: False` and use module-level `getLogger(__name__)` — never `print()`.
- Use the messages framework with the post/redirect/get pattern for user feedback.
- Pin dependencies with a lock file (pip-tools/uv); commit the compiled output.

## Reference Files

Read the relevant reference file before writing code for that topic.

| Topic | File | When to read |
|-------|------|-------------|
| Project structure & settings | `references/core.md` | Setting up a Django project, configuring settings, WSGI/ASGI, management commands |
| Model design | `references/models.md` | Creating or modifying models, fields, relations, Meta options, managers, abstract/proxy models |
| ORM queries | `references/queries.md` | Writing queries, fixing N+1, using Q/F objects, aggregation, subqueries, transactions |
| Migrations | `references/migrations.md` | Creating migrations, data migrations, squashing, zero-downtime schema changes |
| Django Admin | `references/admin.md` | Configuring admin, inline models, custom actions, fieldsets, admin security/performance |
| Views | `references/views.md` | Writing FBVs or CBVs, generic views, mixins, choosing between FBV and CBV |
| Pagination | `references/pagination.md` | Paginating querysets, page links that keep filters, keyset pagination, AsyncPaginator |
| URL routing | `references/urls.md` | Configuring URLs, namespaces, path converters, reverse/reverse_lazy |
| Templates | `references/templates.md` | Template inheritance, tags, filters, custom template tags, context processors, security |
| Forms | `references/forms.md` | Django forms, ModelForms, validation, custom validators, formsets, file upload security |
| Messages | `references/messages.md` | Flash messages after form submissions, SuccessMessageMixin, message levels and rendering |
| Authentication & sessions | `references/auth.md` | User models, auth backends, login/logout, password reset, permissions, groups, sessions, cookies |
| Middleware | `references/middleware.md` | Middleware order, custom middleware, exception handling, async middleware |
| Static & media files | `references/static-media.md` | Static/media configuration, collectstatic, WhiteNoise, S3/CDN, file uploads, storage backends |
| Security | `references/security.md` | CSRF, XSS, SQL injection, clickjacking, password hashing, HTTPS, SECRET_KEY, security checks |
| Signals | `references/signals.md` | pre/post_save, pre/post_delete, m2m_changed, custom signals, when to avoid signals |
| Caching | `references/caching.md` | Cache backends, Redis, per-view/fragment/low-level caching, invalidation strategies |
| Internationalization | `references/i18n.md` | i18n settings, gettext/gettext_lazy, translation tags, timezone support, locale structure |
| Testing | `references/testing.md` | TestCase types, Client/RequestFactory, fixtures, pytest-django, factory_boy, mocking, coverage |
| Django REST Framework | `references/drf.md` | Serializers, viewsets, routers, authentication, permissions, throttling, pagination, filtering |
| Background tasks | `references/celery.md` | Celery setup, Django 6.0 native tasks (django.tasks), task definition, retries, queues, periodic tasks, idempotency |
| Sending email | `references/email.md` | send_mail, HTML email, per-environment backends, bulk sending, header injection |
| Logging | `references/logging.md` | LOGGING config, built-in loggers, production error visibility, Sentry, what not to log |
| Async Django | `references/async.md` | Async views, async ORM (aget/acreate), sync_to_async, async auth/sessions, when async pays off |
| Deployment & performance | `references/deployment.md` | Gunicorn, Nginx, Docker, CI/CD, zero-downtime deploys, query optimization, indexing, async views |
| Django Channels | `references/channels.md` | ASGI setup, WebSocket consumers, channel layers, group messaging, WS authentication, SSE |
| Django ecosystem | `references/ecosystem.md` | DRF, django-filter, allauth, debug toolbar, storages, guardian, import-export, unfold, django-redis, sitemaps, syndication feeds, redirects |
| Architecture patterns | `references/architecture.md` | Custom fields, multi-DB, database routers, Jinja2, ORM internals, MTV, service layer, DDD, SOLID |
