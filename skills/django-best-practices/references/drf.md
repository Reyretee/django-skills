# Django Rest Framework (DRF) best practices for serializers, validators, relations, views, viewsets, routers, error handling, status codes, the request object, authentication, permissions, throttling, pagination, filtering, versioning, renderers, OpenAPI schemas, caching, performance, and testing.

## Serializers

### Using ModelSerializer over plain Serializer

**Wrong:**
```python
from rest_framework import serializers

class ProductSerializer(serializers.Serializer):
    # Manually defining every field — ignores the model definition
    id = serializers.IntegerField()
    name = serializers.CharField()
    price = serializers.FloatField()  # FloatField for money — rounding issues
    # Missing validation, missing many fields
```

**Correct:**
```python
from rest_framework import serializers
from .models import Product


class ProductSerializer(serializers.ModelSerializer):
    category_name = serializers.CharField(source='category.name', read_only=True)
    discounted_price = serializers.SerializerMethodField()

    class Meta:
        model = Product
        fields = ['id', 'name', 'price', 'category', 'category_name',
                  'discounted_price', 'is_active', 'created_at']
        read_only_fields = ['created_at']  # id is already read-only by default

    def get_discounted_price(self, obj):
        if obj.sale_price:
            return str(obj.sale_price)
        return str(obj.price)
```

> **Why:** `ModelSerializer` derives fields from the model, reducing duplication. Use `source` for dotted attribute access and `SerializerMethodField` for computed values.

## ModelSerializer Configuration

### Explicit field listing and custom validation

**Wrong:**
```python
from rest_framework import serializers
from .models import User

class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = '__all__'  # Exposes password hash, permissions, etc.
```

**Correct:**
```python
from rest_framework import serializers
from rest_framework.validators import UniqueValidator
from django.contrib.auth import get_user_model

User = get_user_model()


class UserSerializer(serializers.ModelSerializer):
    email = serializers.EmailField(
        required=True,
        validators=[UniqueValidator(queryset=User.objects.all())],
    )

    class Meta:
        model = User
        fields = ['id', 'username', 'email', 'date_joined']
        read_only_fields = ['date_joined']
```

> **Why:** Never use `fields = '__all__'` — it exposes internal fields. List fields explicitly. Use `UniqueValidator` instead of a manual `.exists()` check in `validate_email` — the manual check is race-prone and forgets to exclude the current instance on updates; `UniqueValidator` handles both automatically.

## Validators

### Built-in validators and object-level validation

**Wrong:**
```python
from rest_framework import serializers
from .models import Booking

class BookingSerializer(serializers.ModelSerializer):
    class Meta:
        model = Booking
        fields = ['room', 'date', 'start_time', 'end_time']

    def validate_room(self, value):
        # Manual duplicate check — race-prone, misses updates
        if Booking.objects.filter(room=value, date=self.initial_data.get('date')).exists():
            raise serializers.ValidationError('Room already booked.')
        return value
    # Cross-field check (start < end) crammed into a field validator
    # where the other field may not be available yet
```

**Correct:**
```python
from rest_framework import serializers
from rest_framework.validators import UniqueTogetherValidator
from .models import Booking


class BookingSerializer(serializers.ModelSerializer):
    class Meta:
        model = Booking
        fields = ['room', 'date', 'start_time', 'end_time']
        validators = [
            UniqueTogetherValidator(
                queryset=Booking.objects.all(),
                fields=['room', 'date'],
            ),
        ]

    def validate(self, data):
        # Object-level validation — all fields available together
        if data['start_time'] >= data['end_time']:
            raise serializers.ValidationError('start_time must be before end_time.')
        return data
```

> **Why:** Use `UniqueValidator` for single-field uniqueness and `UniqueTogetherValidator` for composite uniqueness — both exclude the current instance on updates automatically. Cross-field checks belong in object-level `validate()`, where all deserialized fields are available. Note: `ModelSerializer` auto-generates `UniqueValidator`s from model fields with `unique=True`, so don't duplicate them.

## Nested Serializers

### Separate read and write serializers for nested data

**Wrong:**
```python
from rest_framework import serializers
from .models import Order, OrderItem

class OrderItemSerializer(serializers.ModelSerializer):
    class Meta:
        model = OrderItem
        fields = '__all__'

class OrderSerializer(serializers.ModelSerializer):
    items = OrderItemSerializer(many=True)
    # Nested write is not supported by default — POST will fail
    class Meta:
        model = Order
        fields = ['id', 'user', 'items', 'total']
```

**Correct:**
```python
from django.db import transaction
from rest_framework import serializers
from .models import Order, OrderItem


class OrderItemSerializer(serializers.ModelSerializer):
    class Meta:
        model = OrderItem
        fields = ['id', 'product', 'quantity', 'unit_price']


class OrderReadSerializer(serializers.ModelSerializer):
    items = OrderItemSerializer(many=True, read_only=True)
    user_name = serializers.CharField(source='user.get_full_name', read_only=True)

    class Meta:
        model = Order
        fields = ['id', 'user_name', 'items', 'total', 'status', 'created_at']


class OrderWriteSerializer(serializers.ModelSerializer):
    items = OrderItemSerializer(many=True)

    class Meta:
        model = Order
        fields = ['items']

    def create(self, validated_data):
        items_data = validated_data.pop('items')
        with transaction.atomic():
            order = Order.objects.create(**validated_data)
            OrderItem.objects.bulk_create(
                OrderItem(order=order, **item_data) for item_data in items_data
            )
        return order
```

> **Why:** Use separate serializers for read (nested, rich) and write (flat, writable). Override `create()`/`update()` for writable nested serializers since DRF doesn't handle them automatically. Wrap multi-object writes in `transaction.atomic()` — otherwise a failure mid-loop leaves an order with half its items — and use `bulk_create` to insert all items in one query.

## Serializer Relations

### Choosing the right related field

**Wrong:**
```python
from rest_framework import serializers
from .models import Product

class ProductSerializer(serializers.ModelSerializer):
    # read_only relation used where the client needs to WRITE the category
    category = serializers.StringRelatedField()

    class Meta:
        model = Product
        fields = ['id', 'name', 'category']
    # POST/PUT with a category now fails — StringRelatedField is read-only
```

**Correct:**
```python
from rest_framework import serializers
from .models import Category, Product


class ProductSerializer(serializers.ModelSerializer):
    # Writable by pk — queryset= is REQUIRED for writable relations
    category = serializers.PrimaryKeyRelatedField(
        queryset=Category.objects.all(),
        html_cutoff=100,  # Cap browsable-API dropdown size
    )
    # Writable by natural key instead of pk
    # category = serializers.SlugRelatedField(
    #     slug_field='slug', queryset=Category.objects.all())
    # Read-only human-readable label (uses the model's __str__)
    category_label = serializers.StringRelatedField(source='category', read_only=True)

    class Meta:
        model = Product
        fields = ['id', 'name', 'category', 'category_label']
```

> **Why:** `PrimaryKeyRelatedField` is the default and fastest; `SlugRelatedField` accepts natural keys; `StringRelatedField` is always read-only. Writable relations require `queryset=` — it's how DRF validates the incoming value. Watch the browsable API: relation fields render as a `<select>` of the *entire* queryset, which can fetch thousands of rows per page load — cap it with `html_cutoff`.

## Views (APIView and Generics)

### Prefer generic views over raw APIView

**Wrong:**
```python
from rest_framework.views import APIView
from rest_framework.response import Response
from .models import Product
from .serializers import ProductSerializer

class ProductList(APIView):
    def get(self, request):
        products = Product.objects.all()
        serializer = ProductSerializer(products, many=True)
        return Response(serializer.data)

    def post(self, request):
        serializer = ProductSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=201)
        return Response(serializer.errors, status=400)
    # Re-implementing what generics already provide
```

**Correct:**
```python
from rest_framework import generics
from .models import Product
from .serializers import ProductSerializer


class ProductListCreateView(generics.ListCreateAPIView):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer


class ProductDetailView(generics.RetrieveUpdateDestroyAPIView):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer

    def get_queryset(self):
        return super().get_queryset().filter(is_active=True)
```

> **Why:** Generic views handle serialization, pagination, status codes, and error responses. Use `APIView` only when generics don't fit your use case.

## Error Handling & Exceptions

### raise_exception and a consistent error envelope

**Wrong:**
```python
from rest_framework.views import APIView
from rest_framework.response import Response

class ProductCreateView(APIView):
    def post(self, request):
        serializer = ProductSerializer(data=request.data)
        # Manual branching — every view reinvents error formatting
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=201)
        return Response({'oops': serializer.errors}, status=400)
        # Inconsistent error shape across endpoints
```

**Correct:**
```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status

class ProductCreateView(APIView):
    def post(self, request):
        serializer = ProductSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)  # 400 handled by DRF
        serializer.save()
        return Response(serializer.data, status=status.HTTP_201_CREATED)


# For a consistent error envelope API-wide, wrap the default handler:
# exceptions.py
from rest_framework.views import exception_handler

def custom_exception_handler(exc, context):
    response = exception_handler(exc, context)
    if response is not None:
        response.data = {
            'error': {
                'status_code': response.status_code,
                'detail': response.data,
            }
        }
    return response

# settings.py
REST_FRAMEWORK = {
    'EXCEPTION_HANDLER': 'apps.core.exceptions.custom_exception_handler',
}
```

> **Why:** `is_valid(raise_exception=True)` lets DRF's exception handler produce the 400 response — no manual branching, and every endpoint fails with the same shape. A custom `EXCEPTION_HANDLER` gives one place to define your error envelope. Caveat: it only catches DRF's `APIException` subclasses plus Django's `Http404` and `PermissionDenied` — any other unhandled exception is still a plain 500.

## Status Codes

### Named constants over magic numbers

**Wrong:**
```python
from rest_framework.response import Response

def post(self, request):
    ...
    return Response(serializer.data, status=201)  # Magic number

def destroy(self, request, pk=None):
    ...
    return Response(status=204)  # What was 204 again?
```

**Correct:**
```python
from rest_framework import status
from rest_framework.response import Response

def post(self, request):
    ...
    return Response(serializer.data, status=status.HTTP_201_CREATED)

def destroy(self, request, pk=None):
    ...
    return Response(status=status.HTTP_204_NO_CONTENT)
```

> **Why:** `status.HTTP_201_CREATED` reads as intent; `201` requires the reader to know the HTTP spec by heart. The constants also prevent typos (`status=200` vs `status=201` on create) from slipping through review, and `status.is_success()` / `is_client_error()` helpers make test assertions clearer.

## The Request Object

### request.data over request.POST

**Wrong:**
```python
from rest_framework.views import APIView

class ProductView(APIView):
    def put(self, request):
        # request.POST only contains form data from POST requests —
        # empty for JSON payloads and for PUT/PATCH entirely
        name = request.POST.get('name')

    def get(self, request):
        category = request.GET.get('category')  # Works, but not DRF-idiomatic
```

**Correct:**
```python
from rest_framework.views import APIView

class ProductView(APIView):
    def put(self, request):
        # request.data parses JSON, form, and multipart —
        # and works for POST, PUT, and PATCH
        name = request.data.get('name')

    def get(self, request):
        # query_params is the DRF-idiomatic (and better named) alias
        category = request.query_params.get('category')
        # request.user  → authenticated user (or AnonymousUser)
        # request.auth  → token/credential the user authenticated with
```

> **Why:** `request.POST` only handles form-encoded POST bodies; `request.data` handles any parser (JSON, multipart) and any method (POST/PUT/PATCH). Use `request.query_params` instead of `request.GET` — the name is accurate (query params arrive on any method). `request.user` and `request.auth` are populated by the authentication classes.

## ViewSets

### Permission-aware ViewSets with custom actions

**Wrong:**
```python
from rest_framework import viewsets
from .models import Product
from .serializers import ProductSerializer

class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    # Exposes create, update, partial_update, destroy without any permission checks
```

**Correct:**
```python
from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.permissions import IsAuthenticated, IsAdminUser
from rest_framework.response import Response
from .models import Product
from .serializers import ProductSerializer


class ProductViewSet(viewsets.ModelViewSet):
    serializer_class = ProductSerializer

    def get_queryset(self):
        return Product.objects.select_related('category').filter(is_active=True)

    def get_permissions(self):
        if self.action in ['create', 'update', 'partial_update', 'destroy']:
            return [IsAdminUser()]
        return [IsAuthenticated()]

    @action(detail=True, methods=['post'])
    def archive(self, request, pk=None):
        product = self.get_object()
        product.is_active = False
        product.save(update_fields=['is_active'])
        return Response({'status': 'archived'})

    @action(detail=False, methods=['get'])
    def featured(self, request):
        featured = self.get_queryset().filter(is_featured=True)[:10]
        serializer = self.get_serializer(featured, many=True)
        return Response(serializer.data)
```

> **Why:** ViewSets combine list/create/retrieve/update/destroy into one class. Use `get_permissions()` to vary permissions per action. `@action` adds custom endpoints.

### Exposing only the actions you need

**Wrong:**
```python
from rest_framework import viewsets

class CategoryViewSet(viewsets.ModelViewSet):
    queryset = Category.objects.all()
    serializer_class = CategorySerializer
    # Categories are managed in the admin — yet this exposes
    # POST/PUT/PATCH/DELETE endpoints nobody asked for
```

**Correct:**
```python
from rest_framework import viewsets, mixins


# Read-only resource: list + retrieve only
class CategoryViewSet(viewsets.ReadOnlyModelViewSet):
    queryset = Category.objects.all()
    serializer_class = CategorySerializer


# Custom combination: create + list, no update/delete
class FeedbackViewSet(mixins.CreateModelMixin,
                      mixins.ListModelMixin,
                      viewsets.GenericViewSet):
    queryset = Feedback.objects.all()
    serializer_class = FeedbackSerializer


# Per-action serializers
class OrderViewSet(viewsets.ModelViewSet):
    queryset = Order.objects.all()

    def get_serializer_class(self):
        if self.action in ['create', 'update', 'partial_update']:
            return OrderWriteSerializer
        return OrderReadSerializer
```

> **Why:** `ModelViewSet` exposes all six CRUD actions — don't ship write endpoints you don't need. Use `ReadOnlyModelViewSet` for read-only resources, or compose `GenericViewSet` with mixins for exact control. `get_serializer_class()` keyed on `self.action` pairs naturally with the read/write serializer split.

## Binding the Request User

### Never accept the user from the client payload

**Wrong:**
```python
class OrderSerializer(serializers.ModelSerializer):
    class Meta:
        model = Order
        fields = ['id', 'user', 'items', 'total']
        # 'user' is writable — a client can POST {"user": 42, ...}
        # and create orders on behalf of ANY user (mass assignment / IDOR)
```

**Correct:**
```python
class OrderSerializer(serializers.ModelSerializer):
    class Meta:
        model = Order
        fields = ['id', 'user', 'items', 'total']
        read_only_fields = ['user']


class OrderViewSet(viewsets.ModelViewSet):
    serializer_class = OrderSerializer
    permission_classes = [IsAuthenticated]

    def get_queryset(self):
        return Order.objects.filter(user=self.request.user)

    def perform_create(self, serializer):
        # Bind the authenticated user server-side
        serializer.save(user=self.request.user)


# Alternative — bind inside the serializer itself:
class OrderSerializer(serializers.ModelSerializer):
    user = serializers.HiddenField(default=serializers.CurrentUserDefault())
```

> **Why:** A writable `user` field lets any client create or reassign records to arbitrary users — a classic mass-assignment/IDOR vulnerability. Bind the user server-side via `perform_create(serializer.save(user=self.request.user))`, or use `HiddenField(default=CurrentUserDefault())` if you want the serializer self-contained (it also makes the value available to validators).

## Routers

### Auto-generating URL patterns for ViewSets

**Wrong:**
```python
from django.urls import path
from .views import ProductViewSet

# Manually mapping ViewSet methods to URLs
urlpatterns = [
    path('products/', ProductViewSet.as_view({'get': 'list', 'post': 'create'})),
    path('products/<int:pk>/', ProductViewSet.as_view({'get': 'retrieve', 'put': 'update', 'delete': 'destroy'})),
]
```

**Correct:**
```python
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import ProductViewSet, OrderViewSet

router = DefaultRouter()
router.register('products', ProductViewSet, basename='product')
router.register('orders', OrderViewSet, basename='order')

urlpatterns = [
    path('api/v1/', include(router.urls)),
]
# Generates: /api/v1/products/, /api/v1/products/{pk}/, /api/v1/products/{pk}/archive/
```

> **Why:** Routers auto-generate URL patterns for ViewSets, including custom `@action` endpoints. `DefaultRouter` adds an API root view listing all endpoints.

## Authentication

### Choosing authentication per client type

**Wrong:**
```python
# Reflexively reaching for JWT because "sessions are old"
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
}
# ...then storing the JWT in localStorage on the frontend,
# where any XSS payload can read and exfiltrate it
```

**Correct:**
```python
# Choose auth per client type:
# - Same-origin SPA or server-rendered pages → SessionAuthentication
#   (HttpOnly cookie, not readable by JS — often SAFER than JWT)
# - Simple first-party mobile/script clients → rest_framework.authentication.TokenAuthentication
# - Cross-origin SPAs, mobile, microservices → JWT (simplejwt)

# settings.py
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
        'rest_framework.authentication.SessionAuthentication',  # Browsable API + same-origin
    ],
}

from datetime import timedelta
SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=30),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=7),
    'ROTATE_REFRESH_TOKENS': True,
}

# urls.py
from rest_framework_simplejwt.views import TokenObtainPairView, TokenRefreshView

urlpatterns = [
    path('api/token/', TokenObtainPairView.as_view(), name='token_obtain'),
    path('api/token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
]
```

> **Why:** There is no one-size-fits-all auth. `SessionAuthentication` is fine — often safer — for same-origin SPAs because the session cookie is HttpOnly; a JWT in localStorage is readable by any injected script (XSS liability). DRF's built-in `TokenAuthentication` covers simple first-party clients without extra dependencies. Gotcha: `SessionAuthentication` enforces CSRF — unsafe methods (POST/PUT/DELETE) return 403 unless the client sends the CSRF token. If you do use JWT, keep access tokens short-lived and rotate refresh tokens.

## Permissions

### Global defaults and custom object-level permissions

**Wrong:**
```python
from rest_framework.views import APIView
from rest_framework.response import Response

class OrderView(APIView):
    # No permissions — any anonymous user can access
    def get(self, request):
        return Response(Order.objects.values())
```

**Correct:**
```python
from rest_framework import permissions


class IsOwnerOrReadOnly(permissions.BasePermission):
    def has_object_permission(self, request, view, obj):
        if request.method in permissions.SAFE_METHODS:
            return True
        return obj.user == request.user


# settings.py — global default
REST_FRAMEWORK = {
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
}

# Per-view override
from rest_framework import generics

class OrderDetailView(generics.RetrieveUpdateAPIView):
    permission_classes = [permissions.IsAuthenticated, IsOwnerOrReadOnly]
    queryset = Order.objects.all()
    serializer_class = OrderSerializer
```

> **Why:** Set `IsAuthenticated` as the global default. Create custom permissions for object-level checks. When listed in `permission_classes`, every class must return True (implicit AND) — but since DRF 3.9 you can also compose them with `|`, `&`, and `~` for OR/AND/NOT logic.

### Composing permissions and the list-view gotcha

**Wrong:**
```python
class ProductViewSet(viewsets.ModelViewSet):
    # Wants "read for anyone, write for admins" — but a list of classes
    # is AND logic, so this requires BOTH, blocking reads for non-admins
    permission_classes = [IsAuthenticatedOrReadOnly, IsAdminUser]


class DocumentViewSet(viewsets.ModelViewSet):
    queryset = Document.objects.all()  # ALL documents
    permission_classes = [IsOwner]     # has_object_permission checks obj.owner
    # List view still returns everyone's documents —
    # has_object_permission NEVER runs for list
```

**Correct:**
```python
from rest_framework.permissions import IsAuthenticatedOrReadOnly, IsAdminUser


class ProductViewSet(viewsets.ModelViewSet):
    # OR composition: anonymous reads pass, admin writes pass
    permission_classes = [IsAuthenticatedOrReadOnly | IsAdminUser]


class DocumentViewSet(viewsets.ModelViewSet):
    serializer_class = DocumentSerializer
    permission_classes = [IsAuthenticated, IsOwner]

    def get_queryset(self):
        # Scope the queryset — this is what protects list views
        return Document.objects.filter(owner=self.request.user)
```

> **Why:** Since DRF 3.9, permissions compose with `|`, `&`, and `~` — use OR to express "either role suffices" instead of writing a custom class. Critical caveat: `has_object_permission` only runs for detail routes (via `get_object()`); list views never call it. Object-level permissions cannot filter a list — scope `get_queryset()` to the requesting user instead.

## Throttling

### Rate limiting to prevent abuse

**Wrong:**
```python
# No rate limiting — API vulnerable to abuse
REST_FRAMEWORK = {
    # No throttle classes configured
}
```

**Correct:**
```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle',
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/hour',
        'user': '1000/hour',
    },
}

# Custom throttle for sensitive endpoints
from rest_framework.throttling import UserRateThrottle

class LoginRateThrottle(UserRateThrottle):
    rate = '5/minute'

class LoginView(APIView):
    throttle_classes = [LoginRateThrottle]
```

> **Why:** Throttling prevents abuse and brute-force attacks. Set lower rates for anonymous users and sensitive endpoints (login, password reset). DRF stores throttle state in the cache.

### Throttling caveats in production

**Wrong:**
```python
# Relying on DRF throttling as the security layer, with default config
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': ['rest_framework.throttling.AnonRateThrottle'],
    'DEFAULT_THROTTLE_RATES': {'anon': '100/hour'},
}
# Behind a load balancer: AnonRateThrottle keys on REMOTE_ADDR,
# which is the PROXY's IP — all anonymous users share one bucket
# With LocMemCache and 4 gunicorn workers: each worker counts separately
```

**Correct:**
```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle',
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/hour',
        'user': '1000/hour',
        'password-reset': '5/hour',
    },
    'NUM_PROXIES': 1,  # Behind one load balancer — trust X-Forwarded-For one hop
}

# Throttle state MUST live in a shared cache across processes/hosts
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.redis.RedisCache',
        'LOCATION': 'redis://cache:6379/1',
    },
}

# ScopedRateThrottle: per-endpoint rates without custom classes
from rest_framework.throttling import ScopedRateThrottle

class PasswordResetView(APIView):
    throttle_classes = [ScopedRateThrottle]
    throttle_scope = 'password-reset'
```

> **Why:** DRF throttling is a soft application-level guard, not a security measure — a determined attacker needs to be stopped at the edge (CDN/WAF/nginx rate limits). Behind proxies, set `NUM_PROXIES` or every anonymous client is keyed on the proxy's IP and shares one bucket. Throttle counters live in the cache: with `LocMemCache`, each worker process keeps its own counts, so multi-process deployments need Redis/Memcached. `ScopedRateThrottle` gives per-endpoint rates via `throttle_scope`.

## Pagination

### Always paginate list endpoints

**Wrong:**
```python
# Returning all records at once
class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.all()  # Returns 100,000 products in one response
```

**Correct:**
```python
# settings.py — global pagination
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 20,
}

# Custom pagination
from rest_framework.pagination import PageNumberPagination, CursorPagination


class StandardPagination(PageNumberPagination):
    page_size = 20
    page_size_query_param = 'page_size'
    max_page_size = 100


class TimelinePagination(CursorPagination):
    page_size = 50
    ordering = '-created_at'
    # CursorPagination is most efficient for large datasets — no OFFSET


class ProductViewSet(viewsets.ModelViewSet):
    pagination_class = StandardPagination
```

> **Why:** Always paginate list endpoints. `CursorPagination` is best for large datasets (no SQL OFFSET). `PageNumberPagination` is most familiar to API consumers.

## Filtering

### Declarative filtering with django-filter

**Wrong:**
```python
class ProductViewSet(viewsets.ModelViewSet):
    def get_queryset(self):
        qs = Product.objects.all()
        # Manual filtering — tedious and error-prone
        if self.request.query_params.get('category'):
            qs = qs.filter(category=self.request.query_params['category'])
        if self.request.query_params.get('min_price'):
            qs = qs.filter(price__gte=self.request.query_params['min_price'])
        return qs
```

**Correct:**
```python
# pip install django-filter

# filters.py
import django_filters
from .models import Product


class ProductFilter(django_filters.FilterSet):
    min_price = django_filters.NumberFilter(field_name='price', lookup_expr='gte')
    max_price = django_filters.NumberFilter(field_name='price', lookup_expr='lte')
    category = django_filters.CharFilter(field_name='category__slug')

    class Meta:
        model = Product
        fields = ['category', 'is_active']


# views.py
from rest_framework import viewsets, filters
from django_filters.rest_framework import DjangoFilterBackend


class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    filter_backends = [DjangoFilterBackend, filters.SearchFilter, filters.OrderingFilter]
    filterset_class = ProductFilter
    search_fields = ['name', 'description']
    ordering_fields = ['price', 'created_at', 'name']
    ordering = ['-created_at']
```

> **Why:** django-filter provides declarative filtering. Combine with DRF's `SearchFilter` for full-text search and `OrderingFilter` for sortable columns.

## API Versioning

### URL-based versioning with per-version serializers

**Wrong:**
```python
# No versioning — breaking changes affect all clients immediately
# Or: path('api/products/', ProductListView.as_view())
```

**Correct:**
```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.URLPathVersioning',
    'DEFAULT_VERSION': 'v1',
    'ALLOWED_VERSIONS': ['v1', 'v2'],
}

# urls.py
from django.urls import path, include

urlpatterns = [
    path('api/<version>/', include('apps.api.urls')),
]

# views.py
class ProductViewSet(viewsets.ModelViewSet):
    def get_serializer_class(self):
        if self.request.version == 'v2':
            return ProductV2Serializer
        return ProductV1Serializer
```

> **Why:** URL-based versioning (`/api/v1/`, `/api/v2/`) is the most explicit and easiest for API consumers. Switch serializers per version to evolve the API without breaking clients.

## Renderers & Parsers

### JSON-only in production, browsable API in DEBUG

**Wrong:**
```python
# Default settings shipped to production — BrowsableAPIRenderer stays on,
# rendering a full HTML page (with forms and dropdowns) for browser requests
REST_FRAMEWORK = {}
```

**Correct:**
```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_RENDERER_CLASSES': [
        'rest_framework.renderers.JSONRenderer',
    ],
    'DEFAULT_PARSER_CLASSES': [
        'rest_framework.parsers.JSONParser',
    ],
}

if DEBUG:
    REST_FRAMEWORK['DEFAULT_RENDERER_CLASSES'].append(
        'rest_framework.renderers.BrowsableAPIRenderer',
    )

# File uploads need multipart parsing — enable per-view
from rest_framework.parsers import MultiPartParser, FormParser

class AvatarUploadView(APIView):
    parser_classes = [MultiPartParser, FormParser]

    def put(self, request):
        file = request.data['avatar']
        ...
```

> **Why:** The browsable API is a great development tool but a liability in production: it leaks endpoint structure, renders relation dropdowns (extra queries), and adds template-rendering overhead. Serve JSON only in production and enable `BrowsableAPIRenderer` under `DEBUG`. For file uploads, add `MultiPartParser` on the specific view instead of globally.

## OpenAPI Schemas

### drf-spectacular over the deprecated built-in generation

**Wrong:**
```python
# DRF's built-in schema generation (CoreAPI/AutoSchema) is deprecated
from rest_framework.schemas import get_schema_view

urlpatterns = [
    path('openapi/', get_schema_view(title='My API', version='1.0.0')),
]
```

**Correct:**
```python
# pip install drf-spectacular

# settings.py
INSTALLED_APPS = [..., 'drf_spectacular']
REST_FRAMEWORK = {
    'DEFAULT_SCHEMA_CLASS': 'drf_spectacular.openapi.AutoSchema',
}
SPECTACULAR_SETTINGS = {
    'TITLE': 'My API',
    'VERSION': '1.0.0',
}

# urls.py
from drf_spectacular.views import SpectacularAPIView, SpectacularSwaggerView

urlpatterns = [
    path('api/schema/', SpectacularAPIView.as_view(), name='schema'),
    path('api/docs/', SpectacularSwaggerView.as_view(url_name='schema')),
]

# Refine per-view documentation where introspection isn't enough
from drf_spectacular.utils import extend_schema

class ProductViewSet(viewsets.ModelViewSet):
    @extend_schema(responses=ProductSerializer, summary='Archive a product')
    @action(detail=True, methods=['post'])
    def archive(self, request, pk=None):
        ...
```

> **Why:** DRF's built-in schema generation is deprecated and produces incomplete OpenAPI output. drf-spectacular is the community-standard replacement: it introspects serializers, pagination, and filters automatically, ships Swagger/Redoc UIs, and `@extend_schema` fills the gaps (custom actions, non-standard responses).

## SerializerMethodField Security

### Conditionally exposing sensitive data

**Wrong:**
```python
from rest_framework import serializers

class UserSerializer(serializers.ModelSerializer):
    role = serializers.SerializerMethodField()

    class Meta:
        model = User
        fields = ['id', 'username', 'role']

    def get_role(self, obj):
        # Exposes internal role to everyone — no permission check
        return obj.role
```

**Correct:**
```python
from rest_framework import serializers


class UserSerializer(serializers.ModelSerializer):
    role = serializers.SerializerMethodField()
    email = serializers.SerializerMethodField()

    class Meta:
        model = User
        fields = ['id', 'username', 'role', 'email']

    def get_role(self, obj):
        request = self.context.get('request')
        if request and (request.user == obj or request.user.is_staff):
            return obj.role
        return None

    def get_email(self, obj):
        request = self.context.get('request')
        if request and (request.user == obj or request.user.is_staff):
            return obj.email
        return None  # Hide email from other users
```

> **Why:** SerializerMethodField can access the request via `self.context['request']`. Use it to conditionally expose sensitive data based on the requesting user's identity or permissions.

## DRF Performance

### Optimizing querysets with select_related and prefetch_related

**Wrong:**
```python
from rest_framework import viewsets
from .models import Order
from .serializers import OrderSerializer

class OrderViewSet(viewsets.ModelViewSet):
    queryset = Order.objects.all()
    serializer_class = OrderSerializer
    # Serializer accesses order.customer.name and order.items.all()
    # N+1 queries on every list request
```

**Correct:**
```python
from rest_framework import viewsets
from .models import Order
from .serializers import OrderSerializer


class OrderViewSet(viewsets.ModelViewSet):
    serializer_class = OrderSerializer

    def get_queryset(self):
        return (
            Order.objects
            .select_related('customer')
            .prefetch_related('items__product')
            # WARNING: .only() must cover every field the serializer touches.
            # Accessing a deferred field triggers an extra query PER ROW —
            # worse than not using .only() at all.
            .only('id', 'total', 'status', 'created_at',
                  'customer__id', 'customer__name')
        )
```

> **Why:** Optimize `get_queryset()` with `select_related` for FK/O2O, `prefetch_related` for M2M/reverse FK, and `only()` to limit fetched columns. Caveat: `.only()` must stay in lockstep with the serializer's `fields` — if the serializer reads a deferred field, Django silently fetches it with one extra query per row, turning an optimization into an N+1. Profile with Django Debug Toolbar.

## Caching

### Caching API responses without leaking data across users

**Wrong:**
```python
from django.views.decorators.cache import cache_page
from django.utils.decorators import method_decorator

class OrderViewSet(viewsets.ModelViewSet):
    @method_decorator(cache_page(60 * 5))
    def list(self, request, *args, **kwargs):
        # Response is keyed by URL only — user A's orders
        # get served from cache to user B
        return super().list(request, *args, **kwargs)
```

**Correct:**
```python
from django.views.decorators.cache import cache_page
from django.views.decorators.vary import vary_on_headers
from django.utils.decorators import method_decorator


class ProductViewSet(viewsets.ReadOnlyModelViewSet):
    # Public, identical for everyone — safe to cache by URL
    @method_decorator(cache_page(60 * 15))
    def list(self, request, *args, **kwargs):
        return super().list(request, *args, **kwargs)


class OrderViewSet(viewsets.ModelViewSet):
    # Per-user data — vary the cache key on the auth credential
    @method_decorator(cache_page(60 * 2))
    @method_decorator(vary_on_headers('Authorization'))
    def list(self, request, *args, **kwargs):
        return super().list(request, *args, **kwargs)
```

> **Why:** DRF views work with Django's `cache_page`, applied via `method_decorator` on the handler method. `cache_page` keys on the URL — for authenticated endpoints you must add `vary_on_headers('Authorization')` (or `'Cookie'` for session auth) so each credential gets its own cache entry, otherwise one user's cached response is served to everyone. Keep TTLs short for data that changes.

## Testing APIs

### APITestCase with force_authenticate and named routes

**Wrong:**
```python
from django.test import TestCase

class ProductAPITest(TestCase):
    def test_create_product(self):
        # Hand-rolling token plumbing just to test a view
        response = self.client.post('/api/token/', {'username': 'u', 'password': 'p'})
        token = response.json()['access']
        response = self.client.post(
            '/api/v1/products/',  # Hardcoded URL breaks when routes change
            data='{"name": "Widget"}',
            content_type='application/json',
            HTTP_AUTHORIZATION=f'Bearer {token}',
        )
        self.assertEqual(response.status_code, 201)  # Magic number
```

**Correct:**
```python
from django.urls import reverse
from rest_framework import status
from rest_framework.test import APITestCase


class ProductAPITest(APITestCase):
    def setUp(self):
        self.user = User.objects.create_user('staff', password='x', is_staff=True)

    def test_create_product(self):
        # Skip token plumbing entirely — auth is not what's under test
        self.client.force_authenticate(user=self.user)
        url = reverse('product-list')  # Router basename, not a hardcoded path
        response = self.client.post(url, {'name': 'Widget', 'price': '9.99'})
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)
        self.assertEqual(response.data['name'], 'Widget')

    def test_anonymous_cannot_create(self):
        response = self.client.post(reverse('product-list'), {'name': 'Widget'})
        self.assertEqual(response.status_code, status.HTTP_401_UNAUTHORIZED)
```

> **Why:** `APITestCase` provides an `APIClient` that speaks JSON natively and offers `force_authenticate()` — testing your endpoint's behavior shouldn't require exercising the token flow too. `reverse('product-list')` / `reverse('product-detail', args=[pk])` use the router's basename, so URL changes don't break tests. Assert against `response.data` (parsed) and `status.HTTP_*` constants. For testing a view in full isolation (no URLconf, no middleware), use `APIRequestFactory` and call the view directly.
