# Django ORM Query Best Practices

Reference for QuerySet evaluation, filtering, Q objects, F expressions, aggregation, subqueries, related object fetching, bulk operations, transactions, and N+1 problem detection.

## QuerySet Lazy Evaluation

**Wrong:**
```python
from myapp.models import Product

# Hits the DB immediately — fetches ALL products into memory
all_products = list(Product.objects.all())
# Then filters in Python
cheap = [p for p in all_products if p.price < 10]
```

**Correct:**
```python
from myapp.models import Product

# No DB hit yet — just builds the query
products = Product.objects.filter(price__lt=10)
# Still no DB hit
products = products.select_related('category')
# DB hit happens here — when you iterate, slice, or evaluate
for product in products:
    print(product.name)
```

> **Why:** QuerySets are lazy — they don't hit the database until evaluated (iteration, slicing, `list()`, `len()`, `bool()`). Chain filters freely without performance cost.

## filter() vs get()

**Wrong:**
```python
from myapp.models import User

# get() without exception handling — crashes on missing or duplicate rows
user = User.objects.get(email=email)

# filter() when you expect exactly one result
users = User.objects.filter(pk=user_id)
user = users[0]  # IndexError if empty
```

**Correct:**
```python
from django.shortcuts import get_object_or_404
from myapp.models import User

# In views — returns 404 if not found
user = get_object_or_404(User, pk=user_id)

# In business logic — handle the exception
from django.core.exceptions import ObjectDoesNotExist

try:
    user = User.objects.get(email=email)
except User.DoesNotExist:
    user = None

# Or use filter + first when "not found" is a normal case
user = User.objects.filter(email=email).first()  # Returns None if not found
```

> **Why:** `get()` raises `DoesNotExist` or `MultipleObjectsReturned`. Use `get_object_or_404` in views, `.first()` when absence is expected, and explicit try/except in services.

## exclude() Usage

**Wrong:**
```python
from myapp.models import Product

# Filtering in Python instead of at the database level
products = Product.objects.all()
active_products = [p for p in products if p.status != 'archived']
```

**Correct:**
```python
from myapp.models import Product

# Let the database do the filtering
active_products = Product.objects.exclude(status='archived')

# Combine with filter
featured = Product.objects.filter(is_featured=True).exclude(status='archived')
```

> **Why:** `exclude()` generates a SQL WHERE NOT clause, keeping filtering in the database where it's fast and memory-efficient.

## Q Objects

**Wrong:**
```python
from myapp.models import Product

# Cannot do OR with chained filter calls
# This is AND, not OR:
products = Product.objects.filter(category='electronics').filter(category='books')
```

**Correct:**
```python
from django.db.models import Q
from myapp.models import Product

# OR query
products = Product.objects.filter(
    Q(category='electronics') | Q(category='books')
)

# Complex: (category=electronics AND price<100) OR is_featured=True
products = Product.objects.filter(
    (Q(category='electronics') & Q(price__lt=100)) | Q(is_featured=True)
)

# NOT
products = Product.objects.filter(~Q(status='archived'))
```

> **Why:** Q objects enable OR, AND, and NOT combinations that chained `.filter()` calls cannot express. Use `|` for OR, `&` for AND, `~` for NOT.

## F Expressions

**Wrong:**
```python
from myapp.models import Product

# Race condition — reads, modifies in Python, writes back
product = Product.objects.get(pk=1)
product.view_count = product.view_count + 1
product.save()
```

**Correct:**
```python
from django.db.models import F
from myapp.models import Product

# Atomic update — database does the increment
Product.objects.filter(pk=1).update(view_count=F('view_count') + 1)

# Compare fields against each other
# Products where sale_price < original_price
discounted = Product.objects.filter(sale_price__lt=F('original_price'))
```

> **Why:** F expressions perform operations at the database level, avoiding race conditions from concurrent requests. Two users incrementing simultaneously won't lose a count.

## Aggregation

**Wrong:**
```python
from myapp.models import Order

# Aggregating in Python — loads all rows into memory
orders = Order.objects.all()
total = sum(o.total for o in orders)
average = total / len(orders)
```

**Correct:**
```python
from django.db.models import Avg, Count, Sum, Max, Min
from myapp.models import Order

# aggregate() returns a dict — single result for the whole queryset
result = Order.objects.aggregate(
    total_revenue=Sum('total'),
    avg_order=Avg('total'),
    order_count=Count('id'),
)
# result = {'total_revenue': Decimal('50000.00'), 'avg_order': ..., 'order_count': 500}

# annotate() adds a computed column to each row
from myapp.models import Customer
customers = Customer.objects.annotate(
    total_spent=Sum('orders__total'),
    order_count=Count('orders'),
).filter(order_count__gte=5)
```

> **Why:** `aggregate()` produces a single summary dict. `annotate()` adds computed columns per row. Both execute in SQL — no Python-side iteration needed.

## StringAgg and AnyValue (Django 6.0)

**Wrong:**
```python
# Deprecated import — postgres-only, and its kwarg was `ordering`
from django.contrib.postgres.aggregates import StringAgg

# Or worse: fetching every row and string-joining in Python
names = ', '.join(
    p.name for p in Product.objects.filter(category_id=category_id)
)
```

**Correct:**
```python
from django.db.models import AnyValue, Count, StringAgg, Value
from myapp.models import Category

# Comma-separated related names, computed in SQL — works on all databases
categories = Category.objects.annotate(
    product_names=StringAgg(
        'products__name',
        delimiter=Value(', '),
        order_by='products__name',
    ),
)

# AnyValue — pick a non-grouped column without adding it to GROUP BY
report = Category.objects.values('parent_id').annotate(
    product_count=Count('products'),
    sample_name=AnyValue('name'),
)
```

> **Why:** Django 6.0 promotes `StringAgg` to `django.db.models` and it works on all supported databases; the `django.contrib.postgres` version is deprecated, and its `ordering` kwarg is now `order_by`. `AnyValue` lets the database pick one representative value per group instead of post-processing rows in Python.

## Annotation with Subquery

**Wrong:**
```python
from myapp.models import Author, Book

# N+1 — one query per author to get their latest book
for author in Author.objects.all():
    latest = Book.objects.filter(author=author).order_by('-published').first()
    print(author.name, latest.title if latest else 'No books')
```

**Correct:**
```python
from django.db.models import OuterRef, Subquery
from myapp.models import Author, Book

latest_book = Book.objects.filter(
    author=OuterRef('pk')
).order_by('-published')

authors = Author.objects.annotate(
    latest_book_title=Subquery(latest_book.values('title')[:1])
)
for author in authors:
    print(author.name, author.latest_book_title)
```

> **Why:** Subquery runs a correlated subquery inside a single SQL statement — one query instead of N+1.

## Subquery and Exists

**Wrong:**
```python
from myapp.models import User, Order

# Fetches all orders just to check existence
users_with_orders = []
for user in User.objects.all():
    if Order.objects.filter(user=user).count() > 0:
        users_with_orders.append(user)
```

**Correct:**
```python
from django.db.models import Exists, OuterRef
from myapp.models import User, Order

# Exists — efficient boolean subquery
users_with_orders = User.objects.filter(
    Exists(Order.objects.filter(user=OuterRef('pk')))
)

# Annotate with existence flag
users = User.objects.annotate(
    has_orders=Exists(Order.objects.filter(user=OuterRef('pk')))
)
```

> **Why:** `Exists` generates a SQL EXISTS subquery, which stops scanning as soon as it finds one match — far more efficient than COUNT.

## Raw SQL

**Wrong:**
```python
from django.db import connection

# SQL injection vulnerability — string formatting user input
def search_products(query):
    cursor = connection.cursor()
    cursor.execute(f"SELECT * FROM products WHERE name LIKE '%{query}%'")
    return cursor.fetchall()
```

**Correct:**
```python
from django.db import connection
from myapp.models import Product

# Parameterized raw SQL — safe from injection
def search_products(query):
    return Product.objects.raw(
        "SELECT * FROM products_product WHERE name LIKE %s",
        [f'%{query}%']
    )

# Or with connection.cursor for non-model queries
def get_stats():
    with connection.cursor() as cursor:
        cursor.execute(
            "SELECT category, COUNT(*) FROM products_product GROUP BY category"
        )
        return cursor.fetchall()
```

> **Why:** Never use string formatting with SQL. Always pass parameters separately so the database driver handles escaping. This is your primary defense against SQL injection.

## PostgreSQL Full-Text Search

**Wrong:**
```python
from myapp.models import Article

# Sequential scan over large text columns — slow, no stemming, no ranking
results = Article.objects.filter(body__icontains='django migrations')
```

**Correct:**
```python
from django.contrib.postgres.search import (
    SearchQuery, SearchRank, SearchVector, TrigramSimilarity,
)
from myapp.models import Article

# Ad-hoc search with ranking
query = SearchQuery('django migrations')
vector = SearchVector('title', weight='A') + SearchVector('body', weight='B')
results = (
    Article.objects
    .annotate(rank=SearchRank(vector, query))
    .filter(rank__gte=0.3)
    .order_by('-rank')
)

# For real tables: persist the vector and index it
from django.contrib.postgres.indexes import GinIndex
from django.contrib.postgres.search import SearchVectorField

class Article(models.Model):
    ...
    search_vector = SearchVectorField(null=True)

    class Meta:
        indexes = [GinIndex(fields=['search_vector'])]

# Fuzzy matching for typos (requires the pg_trgm extension)
results = (
    Article.objects
    .annotate(similarity=TrigramSimilarity('title', 'djnago'))
    .filter(similarity__gt=0.3)
    .order_by('-similarity')
)
```

> **Why:** `icontains` can't use a normal B-tree index and does no stemming or relevance ranking. Postgres full-text search handles both, and a persistent `SearchVectorField` with a GIN index keeps it fast on real tables. Trigram similarity covers fuzzy/typo matching.

## select_related

**Wrong:**
```python
from myapp.models import Comment

# N+1 problem — each comment triggers a query for post and user
comments = Comment.objects.all()
for comment in comments:
    print(comment.post.title, comment.user.username)
# This generates 1 + N*2 queries
```

**Correct:**
```python
from myapp.models import Comment

# Single query with JOINs
comments = Comment.objects.select_related('post', 'user').all()
for comment in comments:
    print(comment.post.title, comment.user.username)
# This generates 1 query with JOINs
```

> **Why:** `select_related` uses SQL JOINs to fetch related ForeignKey/OneToOne objects in a single query. Use it whenever you access FK relations in a loop.

## prefetch_related

**Wrong:**
```python
from myapp.models import Author

# N+1 — each author triggers a separate query for their books
authors = Author.objects.all()
for author in authors:
    for book in author.books.all():  # One query per author
        print(book.title)
```

**Correct:**
```python
from django.db.models import Prefetch
from myapp.models import Author, Book

# Two queries total: one for authors, one for all related books
authors = Author.objects.prefetch_related('books')

# Custom Prefetch for filtering or ordering the related set
authors = Author.objects.prefetch_related(
    Prefetch(
        'books',
        queryset=Book.objects.filter(is_published=True).order_by('-published'),
        to_attr='published_books',
    )
)
for author in authors:
    for book in author.published_books:
        print(book.title)
```

> **Why:** `prefetch_related` does a separate query for the related objects and joins them in Python. Use it for ManyToMany and reverse ForeignKey relations. `Prefetch` objects let you customize the related query.

## bulk_create and bulk_update

**Wrong:**
```python
from myapp.models import LogEntry

# One INSERT per iteration — extremely slow for large batches
for data in large_dataset:
    LogEntry.objects.create(message=data['message'], level=data['level'])
```

**Correct:**
```python
from myapp.models import LogEntry

# Single query with batch INSERT
entries = [
    LogEntry(message=data['message'], level=data['level'])
    for data in large_dataset
]
LogEntry.objects.bulk_create(entries, batch_size=1000)

# bulk_update for existing objects
products = list(Product.objects.filter(category='sale'))
for product in products:
    product.price = product.price * Decimal('0.9')
Product.objects.bulk_update(products, ['price'], batch_size=1000)
```

> **Why:** `bulk_create` and `bulk_update` reduce thousands of queries to a handful. Use `batch_size` to control memory usage and avoid exceeding DB parameter limits.

## Transactions

**Wrong:**
```python
from myapp.models import Account

# No transaction — partial failure leaves inconsistent data
def transfer(from_id, to_id, amount):
    sender = Account.objects.get(pk=from_id)
    sender.balance -= amount
    sender.save()
    # If this crashes here, money is gone but not received
    receiver = Account.objects.get(pk=to_id)
    receiver.balance += amount
    receiver.save()
```

**Correct:**
```python
from django.db import transaction
from django.db.models import F
from myapp.models import Account


def transfer(from_id, to_id, amount):
    with transaction.atomic():
        Account.objects.filter(pk=from_id).update(
            balance=F('balance') - amount
        )
        Account.objects.filter(pk=to_id).update(
            balance=F('balance') + amount
        )

# select_for_update for row-level locking
def transfer_safe(from_id, to_id, amount):
    with transaction.atomic():
        sender = Account.objects.select_for_update().get(pk=from_id)
        if sender.balance < amount:
            raise ValueError('Insufficient funds')
        Account.objects.filter(pk=from_id).update(balance=F('balance') - amount)
        Account.objects.filter(pk=to_id).update(balance=F('balance') + amount)
```

> **Why:** `atomic()` ensures all-or-nothing execution. `select_for_update()` locks rows to prevent concurrent modifications. Both are essential for financial operations.

## select_for_update

**Wrong:**
```python
from myapp.models import Account

# Read-modify-write without a lock — two concurrent requests both read
# the same balance, and one update silently overwrites the other
def withdraw(account_id, amount):
    account = Account.objects.get(pk=account_id)
    if account.balance >= amount:
        account.balance -= amount
        account.save()
```

**Correct:**
```python
from django.db import transaction
from django.db.models import F
from myapp.models import Account, Job


def withdraw(account_id, amount):
    with transaction.atomic():  # select_for_update requires a transaction
        # Row is locked until the transaction commits or rolls back
        account = Account.objects.select_for_update().get(pk=account_id)
        if account.balance < amount:
            raise ValueError('Insufficient funds')
        account.balance -= amount
        account.save(update_fields=['balance'])


# Work queue — skip rows other workers already locked
def claim_next_job():
    with transaction.atomic():
        job = (
            Job.objects
            .select_for_update(skip_locked=True)
            .filter(status='pending')
            .order_by('created_at')
            .first()
        )
        ...

# nowait=True raises DatabaseError immediately instead of blocking
# of=('self',) locks only this table's rows, not select_related JOINed rows

# For a single-column atomic increment, prefer F() — no lock needed
Account.objects.filter(pk=account_id).update(balance=F('balance') - amount)
```

> **Why:** `select_for_update()` takes a pessimistic row lock so concurrent transactions serialize instead of losing writes — it only works inside `transaction.atomic()`. Use `skip_locked=True` for work queues, `nowait=True` to fail fast, and `of=('self',)` to avoid locking rows JOINed via `select_related`. For simple single-column updates, `F()` expressions (or optimistic locking with a version field) avoid holding a lock entirely.

## N+1 Problem Detection and Solution

**Wrong:**
```python
from myapp.models import Order

# View that looks simple but generates 101 queries
def order_list(request):
    orders = Order.objects.all()[:100]
    data = []
    for order in orders:
        data.append({
            'id': order.id,
            'customer': order.customer.name,      # +1 query each
            'product_count': order.items.count(),  # +1 query each
        })
    return JsonResponse(data, safe=False)
```

**Correct:**
```python
from django.db.models import Count
from myapp.models import Order


def order_list(request):
    orders = (
        Order.objects
        .select_related('customer')
        .annotate(product_count=Count('items'))
        [:100]
    )
    data = [
        {
            'id': order.id,
            'customer': order.customer.name,       # No extra query
            'product_count': order.product_count,   # No extra query
        }
        for order in orders
    ]
    return JsonResponse(data, safe=False)
```

> **Why:** Detect N+1 by counting queries with Django Debug Toolbar or `assertNumQueries`. Fix with `select_related` (FK/O2O), `prefetch_related` (M2M/reverse FK), and `annotate` (aggregates).
