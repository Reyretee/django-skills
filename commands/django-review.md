---
description: Review Django code against the django-best-practices skill — models, queries, views, DRF, security, and more
argument-hint: "[path or app name — defaults to files changed on the current branch]"
---

Review Django code for best-practice violations using the django-best-practices skill.

## Scope

Target: `$ARGUMENTS`

- If a path or app name was given, review those files.
- If no argument was given, review the files changed on the current branch (`git diff --name-only` against the default branch, plus staged/unstaged changes). If the diff is empty or this isn't a git repo, ask the user what to review.
- Only review Python, template, and settings files that are part of the Django project.

## Workflow

1. Classify each file by topic (models, queries, views, serializers, forms, templates, settings, migrations, admin, tests, tasks, ...).
2. For each topic present, read the matching reference file from the django-best-practices skill (`references/<topic>.md`) before judging the code. The reference files define the "Wrong" anti-patterns to flag and the "Correct" patterns to recommend.
3. Check every file against the relevant patterns. Pay special attention to:
   - N+1 queries (missing `select_related`/`prefetch_related`), filtering in Python instead of the DB
   - Missing `related_name`, wrong field types, direct `auth.User` references
   - Unprotected views, missing ownership checks in `get_queryset()`
   - DRF serializers using `__all__`, unpaginated list endpoints, missing permissions
   - Raw SQL with f-strings, `mark_safe`/`|safe` on user input, `@csrf_exempt`
   - Non-atomic multi-step writes, signals imported at module level
   - Unsafe migrations (non-nullable columns without defaults on populated tables)
4. Verify each finding against the actual code — report only violations you can point to a specific line for. Do not pad the report with generic advice.

## Report format

For each finding:

- **`file:line`** — one-line summary
- Severity: critical (security/data loss) / warning (bug or performance) / suggestion (convention)
- The problematic snippet, the corrected version, and one sentence on why (taken from the reference file's reasoning)

End with a short summary table (findings by severity) and the top 3 fixes to do first. If the code is clean, say so plainly.

Do not modify any files — this command only reports. Offer to apply fixes as a follow-up.
