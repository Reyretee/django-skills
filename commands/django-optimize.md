---
description: Find and fix Django ORM performance problems — N+1 queries, missing indexes, inefficient querysets
argument-hint: "[path or app name — defaults to the whole project]"
---

Hunt for ORM and query performance problems in this Django project using the django-best-practices skill.

## Scope

Target: `$ARGUMENTS` (defaults to the whole project — views, serializers, admin, tasks, template usage).

## Workflow

1. Read `references/queries.md` and `references/models.md` from the django-best-practices skill first.
2. Trace data access paths, focusing on code that runs per-request or per-row:
   - **N+1 queries**: loops or serializer fields accessing FK/M2M relations without `select_related`/`prefetch_related`; template loops over unoptimized querysets; DRF `SerializerMethodField`/nested serializers querying per object; admin `list_display` without `list_select_related`
   - **Python-side work the DB should do**: filtering/sorting/aggregating in Python, `len(qs)` instead of `.count()`, `.count() > 0` or `if qs:` where `.exists()` fits, loading full instances where `.values_list()`/`.only()` suffices
   - **Write inefficiencies**: per-row `.save()` in loops instead of `bulk_create`/`bulk_update`, read-modify-write instead of `F()` expressions, missing `transaction.atomic()` around multi-step writes
   - **Missing indexes**: fields used in frequent `filter`/`order_by` without `db_index` or `Meta.indexes`; missing composite indexes for common filter pairs
   - **Large-data handling**: unbounded querysets, missing `.iterator()` for big scans, unpaginated endpoints
3. Verify each candidate by reading the model definitions — confirm the relation types (FK vs M2M determines `select_related` vs `prefetch_related`) before recommending a fix.

## Report format

For each finding: **`file:line`** — estimated impact (queries saved or rows avoided) — current snippet → optimized snippet — one-sentence why.

Order findings by impact. End with a summary of total estimated query reduction and any suggested `Meta.indexes` additions as a ready-to-paste code block (note that index changes need a migration).

Do not modify any files — report only, offer to apply fixes as a follow-up.
