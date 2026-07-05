---
description: Run a Django security audit — settings, CSRF/XSS/SQL injection, auth, uploads, and deployment hardening
argument-hint: "[optional path — defaults to the whole project]"
---

Run a security audit on this Django project using the django-best-practices skill.

## Scope

Target: `$ARGUMENTS` (defaults to the whole project).

## Workflow

1. Read `references/security.md` and `references/auth.md` from the django-best-practices skill first — they define the checklist and the correct patterns.
2. Locate the settings module(s) (`config/settings/`, `settings.py`, or similar) and audit:
   - `SECRET_KEY` hardcoded or committed; `DEBUG` handling; `ALLOWED_HOSTS`
   - HTTPS hardening: `SECURE_SSL_REDIRECT`, `SECURE_HSTS_SECONDS` (+ subdomains/preload), `SESSION_COOKIE_SECURE`, `CSRF_COOKIE_SECURE`, `SECURE_PROXY_SSL_HEADER`
   - Clickjacking (`X_FRAME_OPTIONS`), referrer policy, Content Security Policy configuration
   - Password hashers (Argon2 first), password validators, session settings
3. Grep the codebase for dangerous patterns and verify each hit in context:
   - `@csrf_exempt`, `csrf_exempt(`
   - `mark_safe(`, `|safe`, `{% autoescape off %}`
   - `.raw(`, `.extra(`, `cursor.execute(` with f-strings/`%`/`.format()` interpolation
   - `eval(`, `exec(`, `pickle.loads(`, `yaml.load(` without `SafeLoader`
   - Secrets in code: passwords, API keys, tokens (also check committed `.env` files)
4. Audit auth and access control: views missing `login_required`/`LoginRequiredMixin`/DRF permissions, object-level ownership checks in update/delete views, DRF endpoints with `AllowAny`.
5. Audit file uploads: extension/content-type validation, upload paths, serving user files.
6. If a manage.py and a working environment exist, run `python manage.py check --deploy` and include its output. If it can't run, note that and continue with the static audit.

## Report format

Group findings by severity (critical / high / medium / low). For each: **`file:line`**, the issue, the exact fix (code or setting), and why it matters. Findings must reference real lines — no hypothetical issues.

End with a prioritized remediation list and a "production deploy checklist" section showing which items pass and which fail.

Do not modify any files — report only, offer to apply fixes as a follow-up.
