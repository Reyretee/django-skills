# django-skills

A Claude Code plugin for Django best practices: a knowledge skill, a senior Django developer agent, and five slash commands. Covers 28 topic areas and 290+ sub-topics through wrong vs. correct code examples — from project structure to deployment, current through Django 6.0.

When installed, Claude Code automatically applies these patterns when working on Django projects — writing better queries, avoiding N+1 problems, using correct field types, following security best practices, and more.

## Installation

### As a Plugin (recommended — skill + agent + commands)

```
/plugin marketplace add ahmetcdincer/django-skills
/plugin install django-skills@django-skills
```

### Skill Only

```bash
npx skills add ahmetcdincer/django-skills --skill django-best-practices
```

### Skill + Agent

```bash
npx skills add ahmetcdincer/django-skills --skill django-best-practices
mkdir -p .claude/agents && curl -o .claude/agents/django-developer.md https://raw.githubusercontent.com/ahmetcdincer/django-skills/main/agents/django-developer.md
```

**Windows (PowerShell):**

```powershell
npx skills add ahmetcdincer/django-skills --skill django-best-practices
New-Item -ItemType Directory -Force -Path .claude\agents | Out-Null; Invoke-WebRequest -Uri "https://raw.githubusercontent.com/ahmetcdincer/django-skills/main/agents/django-developer.md" -OutFile ".claude\agents\django-developer.md"
```

> **Note:** The `npx skills add` command only installs skills. The Django developer agent must be downloaded separately as shown above. The plugin installation includes everything (skill, agent, and commands) in one step.

## Commands

Installed as a plugin, these slash commands become available:

| Command | What it does |
|---------|--------------|
| `/django-review [path]` | Review Django code against the skill's best-practice patterns (defaults to the branch diff) |
| `/django-security [path]` | Security audit: settings hardening, CSRF/XSS/SQL injection, auth, uploads, `check --deploy` |
| `/django-optimize [path]` | Find N+1 queries, missing indexes, and inefficient querysets, with before/after fixes |
| `/django-new-app <name>` | Scaffold a new app following the project's conventions and the skill's structure rules |
| `/django-upgrade [version]` | Audit for deprecated patterns and produce an upgrade plan (5.x → 6.0 modernization included) |

## What It Does

This skill provides Claude Code with a senior Django developer's knowledge base. Every topic includes:

- A **wrong** example showing the common mistake and why it fails
- A **correct** example showing the production-ready approach
- A short **explanation** of why the correct approach is better

Claude uses this knowledge automatically when generating, reviewing, or refactoring Django code.

## File Structure

```
django-skills/
├── .claude-plugin/
│   ├── plugin.json                     # Plugin manifest
│   └── marketplace.json                # Marketplace entry for /plugin install
├── commands/                           # Slash commands (plugin install)
│   ├── django-review.md
│   ├── django-security.md
│   ├── django-optimize.md
│   ├── django-new-app.md
│   └── django-upgrade.md
├── skills/
│   └── django-best-practices/
│       ├── SKILL.md                    # Workflow, rules, and reference index
│       └── references/
│           ├── core.md                 # Project structure, settings, pinning, tooling
│           ├── models.md               # Field types, choices, relations, Meta, managers
│           ├── queries.md              # QuerySets, Q/F objects, aggregation, N+1, locking, FTS
│           ├── migrations.md           # Safe migrations, data migrations, squashing
│           ├── admin.md                # ModelAdmin, inlines, actions, performance
│           ├── views.md                # FBV, CBV, generic views, mixins
│           ├── pagination.md           # Paginator, querystring links, keyset pagination
│           ├── urls.md                 # path/re_path, namespaces, reverse with query
│           ├── templates.md            # Inheritance, partials, tags, filters, security
│           ├── forms.md                # ModelForms, validation, formsets
│           ├── messages.md             # Flash messages, SuccessMessageMixin
│           ├── auth.md                 # User models, LoginRequiredMiddleware, sessions
│           ├── middleware.md           # Request lifecycle, ordering, async
│           ├── static-media.md         # WhiteNoise, S3/CDN, file uploads
│           ├── security.md             # CSRF, XSS, SQL injection, HTTPS, CSP, signing
│           ├── signals.md              # pre/post_save, custom signals
│           ├── caching.md              # Redis, per-view, fragment, HTTP caching
│           ├── i18n.md                 # gettext_lazy, timezone support
│           ├── logging.md              # LOGGING config, built-in loggers, Sentry
│           ├── email.md                # send_mail, backends, HTML email, bulk sending
│           ├── testing.md              # pytest, factory_boy, mocking, query counts
│           ├── drf.md                  # Serializers, viewsets, permissions, testing APIs
│           ├── celery.md               # Celery + Django 6.0 native tasks (django.tasks)
│           ├── async.md                # Async views, async ORM, sync_to_async
│           ├── deployment.md           # Gunicorn, Docker, CI/CD, connection pooling
│           ├── channels.md             # WebSocket consumers, channel layers
│           ├── ecosystem.md            # Allauth, storages, sitemaps, feeds, redirects
│           └── architecture.md         # Service layer, DDD, SOLID, multi-DB
└── agents/
    └── django-developer.md             # Senior Django developer agent
```

## Architecture

This repo follows the **progressive disclosure** pattern:

- **SKILL.md** — Slim workflow file with rules and a reference index. No inline code examples.
- **references/** — 28 self-contained reference files with wrong/correct code patterns per topic.
- **agents/django-developer.md** — A senior Django developer agent that preloads the skill and applies structured workflows.
- **commands/** — Task-shaped entry points (review, security audit, optimization, scaffolding, upgrade) that drive the skill.

Claude reads only what it needs: SKILL.md identifies the topic, then loads the relevant reference file on demand.

## Topics Covered

| #  | Topic                      | Reference File         |
|----|----------------------------|------------------------|
| 1  | Project Structure          | `references/core.md`   |
| 2  | Models & ORM               | `references/models.md` |
| 3  | ORM Queries                | `references/queries.md` |
| 4  | Migrations                 | `references/migrations.md` |
| 5  | Django Admin               | `references/admin.md` |
| 6  | Views                      | `references/views.md` |
| 7  | Pagination                 | `references/pagination.md` |
| 8  | URL Routing                | `references/urls.md` |
| 9  | Templates                  | `references/templates.md` |
| 10 | Forms                      | `references/forms.md` |
| 11 | Messages Framework         | `references/messages.md` |
| 12 | Authentication & Sessions  | `references/auth.md` |
| 13 | Middleware                 | `references/middleware.md` |
| 14 | Static & Media Files       | `references/static-media.md` |
| 15 | Security                   | `references/security.md` |
| 16 | Signals                    | `references/signals.md` |
| 17 | Caching                    | `references/caching.md` |
| 18 | Internationalization       | `references/i18n.md` |
| 19 | Logging                    | `references/logging.md` |
| 20 | Sending Email              | `references/email.md` |
| 21 | Testing                    | `references/testing.md` |
| 22 | Django REST Framework      | `references/drf.md` |
| 23 | Background Tasks           | `references/celery.md` |
| 24 | Async Django               | `references/async.md` |
| 25 | Deployment & Performance   | `references/deployment.md` |
| 26 | Django Channels            | `references/channels.md` |
| 27 | Django Ecosystem           | `references/ecosystem.md` |
| 28 | Architecture Patterns      | `references/architecture.md` |

## Example

Every sub-topic in the reference files follows this structure:

```markdown
### select_related

**Wrong:**
  comments = Comment.objects.all()
  for comment in comments:
      print(comment.post.title)  # N+1 queries

**Correct:**
  comments = Comment.objects.select_related('post').all()
  for comment in comments:
      print(comment.post.title)  # Single query with JOIN

> **Why:** select_related uses SQL JOINs to fetch related ForeignKey/OneToOne
> objects in a single query. Use it whenever you access FK relations in a loop.
```