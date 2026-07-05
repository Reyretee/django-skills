---
description: Audit a Django project for outdated patterns and prepare it for upgrading to Django 5.x/6.0
argument-hint: "[target version, e.g. 6.0 — defaults to latest]"
---

Audit this Django project for legacy patterns and produce an upgrade plan toward `$ARGUMENTS` (default: the latest stable Django).

## Workflow

1. Determine the current Django version (`requirements*.txt`, `pyproject.toml`, `poetry.lock`, or `pip show django` if an environment is active) and the installed third-party Django packages.
2. Read `references/core.md` from the django-best-practices skill; also read `references/deployment.md` if async/server topics come up.
3. Scan the codebase for deprecated and legacy patterns, verifying each hit in context:
   - `url()` / `re_path` used where `path()` fits; `ugettext*` → `gettext*`; `ifequal`/`{% load staticfiles %}`
   - `USE_TZ = False`, naive datetimes, `datetime.now()` instead of `timezone.now()`
   - `pytz` usage (removed in Django 5.0 in favor of `zoneinfo`)
   - Old-style `Meta.index_together`/`unique_together` where `Meta.indexes`/`UniqueConstraint` should be used
   - `NullBooleanField`, positional `ForeignKey` args without `on_delete`
   - Signals/receivers doing work that Django's newer built-ins now cover
4. Check version-specific opportunities for the target version — new features worth adopting (e.g. Django 5.x: `db_default`, `GeneratedField`, composite primary keys, `LoginRequiredMiddleware`, `{% querystring %}`; Django 6.0: built-in background Tasks framework, native Content Security Policy support, template partials). Recommend adoption only where the project actually has the corresponding hand-rolled equivalent.
5. Check third-party package compatibility with the target version (known blockers: unmaintained packages pinned to old Django).

## Report format

1. **Current state**: Django version, Python version, notable dependencies
2. **Blockers**: things that will break on upgrade — `file:line`, what breaks, the fix
3. **Deprecations**: patterns that still work but are deprecated — with replacements
4. **Modernization opportunities**: new features that would simplify existing code, each with a before/after snippet
5. **Step-by-step upgrade plan**: recommended intermediate versions (one major/LTS at a time), test checkpoints, and the order of code changes

Do not modify any files or bump any dependencies — report only, offer to execute the plan as a follow-up.
