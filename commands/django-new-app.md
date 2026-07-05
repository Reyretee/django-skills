---
description: Scaffold a new Django app following best practices — proper structure, urls, tests layout, and registration
argument-hint: "<app_name> [short description of what the app does]"
---

Create a new Django app named per `$ARGUMENTS`, following the conventions in the django-best-practices skill.

## Workflow

1. Read `references/core.md` from the django-best-practices skill first.
2. Inspect the existing project layout before creating anything: where do current apps live (project root, `apps/`, `src/`)? Is there a `config/` project package? How are existing apps registered and routed? Match the existing conventions — do not impose a different structure on an established project.
3. Create the app with this structure (adapt to project conventions):

```
<app_name>/
├── __init__.py
├── apps.py              # AppConfig with explicit name (and signal imports in ready() if needed later)
├── admin.py
├── models.py
├── urls.py              # app_name = "<app_name>" for namespacing
├── views.py
├── managers.py          # only if custom managers are needed
├── services.py          # only if the project uses a service layer
├── migrations/
│   └── __init__.py
└── tests/
    ├── __init__.py
    ├── factories.py     # factory_boy factories (if the project uses factory_boy)
    ├── test_models.py
    └── test_views.py
```

4. Register the app in `INSTALLED_APPS` (use the dotted path matching the project layout) and include its `urls.py` in the root URLconf with a namespace.
5. If the user described what the app does, sketch the initial models following `references/models.md`: correct field types, explicit `related_name`, `__str__`, `Meta` with ordering/indexes/constraints, `settings.AUTH_USER_MODEL` for user references. Do NOT run `makemigrations` — leave migrations to the user so they can review the models first.
6. If the project uses DRF and the app needs an API, also add `serializers.py` and wire a router, following `references/drf.md`.

## Output

Show the created file tree, the settings/URLconf changes made, and the suggested next steps (review models → `makemigrations` → write first test).
