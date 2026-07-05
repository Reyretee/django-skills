# Django Testing - Best practices for writing tests including test classes, fixtures, factories, mocking, coverage, and organization.

## TestCase vs SimpleTestCase vs TransactionTestCase

**Wrong:**
```python
from django.test import TestCase

# Using TestCase for tests that don't need a database
class UtilsTest(TestCase):  # Unnecessary DB setup/teardown overhead
    def test_format_currency(self):
        self.assertEqual(format_currency(1000), '$10.00')
```

**Correct:**
```python
from django.test import SimpleTestCase, TestCase, TransactionTestCase


# SimpleTestCase — no database access, fastest
class UtilsTest(SimpleTestCase):
    def test_format_currency(self):
        from myapp.utils import format_currency
        self.assertEqual(format_currency(1000), '$10.00')


# TestCase — wraps each test in a transaction (rolled back after), uses DB
class ProductModelTest(TestCase):
    def test_str(self):
        from myapp.models import Product
        product = Product.objects.create(name='Widget', price=9.99)
        self.assertEqual(str(product), 'Widget')


# TransactionTestCase — actually commits to DB, needed for testing transactions
class TransferTest(TransactionTestCase):
    def test_atomic_transfer(self):
        # Test code that uses transaction.atomic() or select_for_update()
        ...
```

> **Why:** `SimpleTestCase` is fastest (no DB). `TestCase` wraps each test in a transaction for isolation. `TransactionTestCase` commits for real — needed when testing transaction behavior.

## Client and RequestFactory

**Wrong:**
```python
from django.test import TestCase

class ViewTest(TestCase):
    def test_product_list(self):
        # Using Client for a test that only needs to test view logic, not middleware
        response = self.client.get('/products/')
        # Client runs the full middleware stack — slow for unit tests
```

**Correct:**
```python
from django.test import TestCase, RequestFactory
from django.contrib.auth import get_user_model
from myapp.views import product_list

User = get_user_model()


class ViewTest(TestCase):
    def setUp(self):
        self.factory = RequestFactory()
        self.user = User.objects.create_user(username='test', password='test')

    def test_product_list_with_client(self):
        # Client — full integration test (middleware, URL routing, templates)
        self.client.login(username='test', password='test')
        response = self.client.get('/products/')
        self.assertEqual(response.status_code, 200)
        self.assertContains(response, 'Products')

    def test_product_list_with_factory(self):
        # RequestFactory — unit test the view function directly
        request = self.factory.get('/products/')
        request.user = self.user
        response = product_list(request)
        self.assertEqual(response.status_code, 200)
```

> **Why:** `Client` tests the full stack (middleware, URL resolution, templates). `RequestFactory` creates bare request objects for testing views in isolation — faster for unit tests.

## Client query_params and HTML Assertions (Django 5.1+)

**Wrong:**
```python
class SearchViewTest(TestCase):
    def test_search(self):
        # Hand-building query strings — encoding bugs with spaces and special chars
        response = self.client.get('/items/?q=' + query)

    def test_item_removed_from_page(self):
        response = self.client.get('/items/')
        # Fragile substring check — breaks on whitespace or attribute order
        self.assertNotIn('<li>Widget</li>', response.content.decode())
```

**Correct:**
```python
class SearchViewTest(TestCase):
    def test_search(self):
        # query_params handles URL encoding (Django 5.1+)
        response = self.client.get('/items/', query_params={'q': 'blue widget'})
        self.assertEqual(response.status_code, 200)

    def test_post_with_query_params(self):
        # Works for POST too — query string stays separate from the body
        response = self.client.post(
            '/items/', {'name': 'Widget'}, query_params={'ref': 'promo'}
        )

    def test_item_removed_from_page(self):
        response = self.client.get('/items/')
        # HTML-aware negative assertion (Django 5.1+)
        self.assertNotInHTML('<li>Widget</li>', response.content.decode())
```

> **Why:** `query_params` URL-encodes values for you and, on POST/PUT, keeps query string arguments cleanly separate from the request body. `assertNotInHTML` parses the HTML, so it isn't fooled by whitespace or attribute ordering the way substring checks are.

## Fixtures

**Wrong:**
```python
from django.test import TestCase

class OrderTest(TestCase):
    fixtures = ['users.json', 'products.json', 'orders.json']
    # JSON fixtures are hard to maintain, brittle, and don't evolve with model changes
    # They also create tight coupling between tests and specific data
```

**Correct:**
```python
from django.test import TestCase
from django.contrib.auth import get_user_model
from myapp.models import Product, Order

User = get_user_model()


class OrderTest(TestCase):
    @classmethod
    def setUpTestData(cls):
        # Created once for the whole TestCase — fast
        cls.user = User.objects.create_user(username='buyer', password='test')
        cls.product = Product.objects.create(name='Widget', price=9.99)

    def test_create_order(self):
        order = Order.objects.create(user=self.user, product=self.product)
        self.assertEqual(order.user, self.user)
```

> **Why:** `setUpTestData` creates test data once per TestCase class (not per test method), making it fast. JSON fixtures are fragile — they break when models change and are hard to read.

## pytest-django

**Wrong:**
```python
# Using unittest-style assertions with pytest
import pytest

@pytest.mark.django_db
class TestProduct:
    def test_create(self):
        from myapp.models import Product
        p = Product.objects.create(name='X', price=1)
        assert p is not None  # Too vague
```

**Correct:**
```python
# conftest.py
import pytest
from django.contrib.auth import get_user_model

User = get_user_model()


@pytest.fixture
def user(db):
    return User.objects.create_user(username='testuser', password='testpass')


@pytest.fixture
def auth_client(client, user):
    client.login(username='testuser', password='testpass')
    return client


# tests/test_views.py
import pytest
from myapp.models import Product


@pytest.mark.django_db
def test_product_list(auth_client):
    Product.objects.create(name='Widget', price=9.99)
    response = auth_client.get('/products/')
    assert response.status_code == 200
    assert b'Widget' in response.content


@pytest.mark.django_db
def test_product_create(auth_client):
    response = auth_client.post('/products/create/', {'name': 'Gadget', 'price': '19.99'})
    assert response.status_code == 302
    assert Product.objects.filter(name='Gadget').exists()
```

> **Why:** pytest-django provides `db`, `client`, `rf` (RequestFactory), and `admin_client` fixtures. Use `@pytest.mark.django_db` to enable DB access. Fixtures compose naturally.

## FactoryBoy

**Wrong:**
```python
# Creating test data manually in every test
def test_order_total(self):
    user = User.objects.create_user(username='u1', password='p')
    product1 = Product.objects.create(name='A', price=10)
    product2 = Product.objects.create(name='B', price=20)
    order = Order.objects.create(user=user)
    OrderItem.objects.create(order=order, product=product1, quantity=2)
    OrderItem.objects.create(order=order, product=product2, quantity=1)
    # 6 lines just for setup — repeated in every test
```

**Correct:**
```python
# pip install factory-boy

# factories.py
import factory
from django.contrib.auth import get_user_model
from myapp.models import Product, Order, OrderItem

User = get_user_model()


class UserFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = User

    username = factory.Sequence(lambda n: f'user{n}')
    email = factory.LazyAttribute(lambda o: f'{o.username}@example.com')


class ProductFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Product

    name = factory.Faker('word')
    price = factory.Faker('pydecimal', left_digits=3, right_digits=2, positive=True)


class OrderFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Order

    user = factory.SubFactory(UserFactory)


class OrderItemFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = OrderItem

    order = factory.SubFactory(OrderFactory)
    product = factory.SubFactory(ProductFactory)
    quantity = factory.Faker('random_int', min=1, max=5)


# Usage in tests
def test_order_total():
    order = OrderFactory()
    OrderItemFactory(order=order, product__price=10, quantity=2)
    OrderItemFactory(order=order, product__price=20, quantity=1)
    assert order.calculate_total() == 40
```

> **Why:** Factories create realistic test data with sensible defaults. `SubFactory` handles relationships. Override specific fields when they matter for the test.

## Mock Usage

**Wrong:**
```python
# Testing with real external services
def test_send_notification(self):
    send_sms('+1234567890', 'Hello')  # Actually sends an SMS during tests!
```

**Correct:**
```python
from unittest.mock import patch, MagicMock
from django.test import TestCase, override_settings
from myapp.services import process_order


class OrderServiceTest(TestCase):
    @patch('myapp.services.send_sms')
    @patch('myapp.services.charge_payment')
    def test_process_order(self, mock_charge, mock_sms):
        mock_charge.return_value = {'status': 'success', 'charge_id': 'ch_123'}

        order = OrderFactory(total=100)
        process_order(order)

        mock_charge.assert_called_once_with(amount=100, currency='usd')
        mock_sms.assert_called_once()
        order.refresh_from_db()
        self.assertEqual(order.status, 'completed')

    @override_settings(EMAIL_BACKEND='django.core.mail.backends.locmem.EmailBackend')
    def test_sends_confirmation_email(self):
        from django.core import mail
        order = OrderFactory()
        process_order(order)
        self.assertEqual(len(mail.outbox), 1)
        self.assertIn('confirmation', mail.outbox[0].subject.lower())
```

> **Why:** Mock external services (SMS, payments, APIs) to keep tests fast and deterministic. Use Django's `locmem` email backend to capture emails in tests.

## Asserting Query Counts

**Wrong:**
```python
def test_product_list(self):
    response = self.client.get('/products/')
    self.assertEqual(response.status_code, 200)
    # Passes even if the view runs 300 queries (N+1) —
    # performance regressions slip through unnoticed
```

**Correct:**
```python
from django.db import connection
from django.test import TestCase
from django.test.utils import CaptureQueriesContext


class ProductListTest(TestCase):
    def test_list_query_count(self):
        # Fails if the view runs more (or fewer) than 3 queries
        with self.assertNumQueries(3):
            self.client.get('/products/')

    def test_cached_path_skips_db(self):
        self.client.get('/products/')  # Warm the cache
        with CaptureQueriesContext(connection) as ctx:
            self.client.get('/products/')
        # The cached code path must hit the DB zero times
        self.assertEqual(len(ctx.captured_queries), 0)
```

> **Why:** `assertNumQueries` turns N+1 regressions into test failures instead of production incidents. `CaptureQueriesContext` records the actual SQL so you can inspect it or assert a cached path never touches the database; for app-wide auditing, `connection.execute_wrapper()` can hook every query.

## Test Coverage and Targets

**Wrong:**
```bash
# Running tests without coverage measurement
python manage.py test
# No idea which code paths are untested
```

**Correct:**
```bash
# pip install coverage pytest-cov

# With pytest
pytest --cov=apps --cov-report=html --cov-report=term-missing

# With Django's test runner
coverage run manage.py test
coverage report -m
coverage html  # Open htmlcov/index.html
```

```ini
# setup.cfg or pyproject.toml
[tool:pytest]
DJANGO_SETTINGS_MODULE = config.settings.test
addopts = --cov=apps --cov-report=term-missing --cov-fail-under=80

[coverage:run]
omit =
    */migrations/*
    */tests/*
    manage.py
```

> **Why:** Aim for 80%+ coverage on business logic. Don't chase 100% — focus on critical paths (payments, auth, data mutations). Exclude migrations and test files from coverage reports.

## Test Organization

**Wrong:**
```python
# All tests in a single tests.py file per app
# apps/orders/tests.py — 2000 lines, mixing unit/integration/e2e tests
```

**Correct:**
```
apps/orders/
    tests/
        __init__.py
        test_models.py          # Unit tests — model methods, validation
        test_views.py           # Integration tests — request/response cycle
        test_services.py        # Unit tests — business logic
        test_api.py             # API endpoint tests
        test_integration.py     # Cross-app integration tests
        factories.py            # Test data factories
        conftest.py             # Shared fixtures
```

```python
# test_models.py — focused on model behavior
from django.test import TestCase
from .factories import OrderFactory


class OrderModelTest(TestCase):
    def test_calculate_total(self):
        ...

    def test_cannot_ship_unpaid_order(self):
        ...
```

> **Why:** Split tests by scope and layer. Small test files are easier to navigate and run selectively (`pytest tests/test_models.py`). Shared factories live in a dedicated file.
