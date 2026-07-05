Django authentication system, sessions, and cookie security best practices.

## 10. Authentication System

### Default User Model

**Wrong:**
```python
# Starting a new project and immediately using auth.User directly everywhere
from django.contrib.auth.models import User

class Profile(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE)
    # Hardcoded to auth.User — can't switch to custom user later
```

**Correct:**
```python
from django.conf import settings
from django.db import models


class Profile(models.Model):
    user = models.OneToOneField(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
        related_name='profile',
    )
```

> **Why:** Always reference `settings.AUTH_USER_MODEL` instead of importing User directly. The default User is fine for simple projects, but referencing via settings lets you swap later without rewriting ForeignKeys.

### Custom User Model

**Wrong:**
```python
# Deciding to customize the user model mid-project
# This requires complex migration surgery and often a database rebuild
from django.contrib.auth.models import AbstractUser

class User(AbstractUser):
    phone = models.CharField(max_length=20)
    # Adding this after tables exist requires recreating the auth tables
```

**Correct:**
```python
# Do this BEFORE the first migrate — at the very start of the project
# users/models.py
from django.contrib.auth.models import AbstractUser
from django.db import models


class User(AbstractUser):
    phone = models.CharField(max_length=20, blank=True)
    date_of_birth = models.DateField(null=True, blank=True)

    class Meta:
        verbose_name = 'user'
        verbose_name_plural = 'users'


# settings.py
AUTH_USER_MODEL = 'users.User'
```

```python
# For maximum control — AbstractBaseUser
from django.contrib.auth.models import AbstractBaseUser, PermissionsMixin, BaseUserManager


class UserManager(BaseUserManager):
    def create_user(self, email, password=None, **extra_fields):
        if not email:
            raise ValueError('Email is required')
        email = self.normalize_email(email)
        user = self.model(email=email, **extra_fields)
        user.set_password(password)
        user.save(using=self._db)
        return user

    def create_superuser(self, email, password=None, **extra_fields):
        extra_fields.setdefault('is_staff', True)
        extra_fields.setdefault('is_superuser', True)
        return self.create_user(email, password, **extra_fields)


class User(AbstractBaseUser, PermissionsMixin):
    email = models.EmailField(unique=True)
    name = models.CharField(max_length=255)
    is_active = models.BooleanField(default=True)
    is_staff = models.BooleanField(default=False)

    objects = UserManager()
    USERNAME_FIELD = 'email'
    REQUIRED_FIELDS = ['name']
```

> **Why:** Define a custom user model at the start of every project — even if you don't need it yet. Switching later requires recreating the database. Use `AbstractUser` to extend, `AbstractBaseUser` to fully control.

### Authentication Backend

**Wrong:**
```python
# Manually checking passwords in views
from django.contrib.auth.hashers import check_password
from django.contrib.auth.models import User

def login_view(request):
    user = User.objects.get(email=request.POST['email'])
    if check_password(request.POST['password'], user.password):
        # Manual session management...
        pass
```

**Correct:**
```python
# Custom backend for email-based login
# backends.py
from django.contrib.auth import get_user_model
from django.contrib.auth.backends import ModelBackend

User = get_user_model()


class EmailBackend(ModelBackend):
    def authenticate(self, request, username=None, password=None, **kwargs):
        email = kwargs.get('email', username)
        try:
            user = User.objects.get(email=email)
        except User.DoesNotExist:
            return None
        if user.check_password(password) and self.user_can_authenticate(user):
            return user
        return None


# settings.py
AUTHENTICATION_BACKENDS = [
    'apps.users.backends.EmailBackend',
    'django.contrib.auth.backends.ModelBackend',
]
```

> **Why:** Custom backends let you authenticate by email, LDAP, OAuth, etc. Django tries each backend in order. `user_can_authenticate` checks `is_active`.

### Login/Logout Views

**Wrong:**
```python
from django.contrib.auth import authenticate, login

def login_view(request):
    if request.method == 'POST':
        user = authenticate(username=request.POST['username'],
                          password=request.POST['password'])
        if user:
            login(request, user)
            return redirect('/')
    return render(request, 'login.html')
    # No CSRF, no form validation, no error messages, no rate limiting
```

**Correct:**
```python
# urls.py — use Django's built-in auth views
from django.contrib.auth import views as auth_views
from django.urls import path

urlpatterns = [
    path('login/', auth_views.LoginView.as_view(template_name='users/login.html'), name='login'),
    path('logout/', auth_views.LogoutView.as_view(), name='logout'),
]

# settings.py
LOGIN_URL = '/users/login/'
LOGIN_REDIRECT_URL = '/'
LOGOUT_REDIRECT_URL = '/'
```

```html
<!-- users/login.html -->
{% extends "base.html" %}
{% block content %}
<form method="post">
  {% csrf_token %}
  {{ form.as_p }}
  <button type="submit">Login</button>
</form>
{% endblock %}
```

> **Why:** Django's auth views handle CSRF, form validation, error messages, and the `next` parameter for post-login redirect. Don't rewrite what's already built.

### Password Reset Flow

**Wrong:**
```python
# Building password reset from scratch
def reset_password(request):
    email = request.POST['email']
    user = User.objects.get(email=email)
    new_password = 'temporary123'  # Sending plaintext password!
    user.set_password(new_password)
    user.save()
    send_mail('Your new password', f'Password: {new_password}', ...)
```

**Correct:**
```python
# urls.py — use Django's built-in password reset views
from django.contrib.auth import views as auth_views
from django.urls import path

urlpatterns = [
    path('password-reset/',
         auth_views.PasswordResetView.as_view(template_name='users/password_reset.html'),
         name='password_reset'),
    path('password-reset/done/',
         auth_views.PasswordResetDoneView.as_view(template_name='users/password_reset_done.html'),
         name='password_reset_done'),
    path('password-reset/<uidb64>/<token>/',
         auth_views.PasswordResetConfirmView.as_view(template_name='users/password_reset_confirm.html'),
         name='password_reset_confirm'),
    path('password-reset/complete/',
         auth_views.PasswordResetCompleteView.as_view(template_name='users/password_reset_complete.html'),
         name='password_reset_complete'),
]
```

> **Why:** Django's reset flow uses cryptographically signed tokens that expire. Never send plaintext passwords. The built-in views handle the full flow: request, email, confirmation, and completion.

### Permissions

**Wrong:**
```python
# Checking permissions with hardcoded role strings
def delete_article(request, pk):
    if request.user.role != 'admin':  # Custom role field — doesn't use Django's permission system
        return HttpResponseForbidden()
    Article.objects.get(pk=pk).delete()
```

**Correct:**
```python
from django.contrib.auth.decorators import permission_required
from django.shortcuts import get_object_or_404
from .models import Article


@permission_required('articles.delete_article', raise_exception=True)
def delete_article(request, pk):
    article = get_object_or_404(Article, pk=pk)
    article.delete()
    return redirect('articles:list')


# Object-level permissions with django-guardian
from guardian.shortcuts import assign_perm

def create_article(request):
    article = Article.objects.create(author=request.user, ...)
    assign_perm('articles.change_article', request.user, article)
    assign_perm('articles.delete_article', request.user, article)
```

> **Why:** Django auto-creates add/change/delete/view permissions per model. Use them with `@permission_required` or `has_perm()`. For object-level permissions, use django-guardian.

### Groups Usage

**Wrong:**
```python
# Assigning permissions to users individually
from django.contrib.auth.models import Permission

user = User.objects.get(username='editor')
perms = Permission.objects.filter(codename__in=[
    'add_article', 'change_article', 'view_article'
])
user.user_permissions.set(perms)
# Repeat for every new editor — tedious and error-prone
```

**Correct:**
```python
from django.contrib.auth.models import Group, Permission

# Create groups once (in a migration or management command)
editors_group, _ = Group.objects.get_or_create(name='Editors')
editor_perms = Permission.objects.filter(
    content_type__app_label='articles',
    codename__in=['add_article', 'change_article', 'view_article'],
)
editors_group.permissions.set(editor_perms)

# Assign users to groups
user.groups.add(editors_group)

# Check in templates: {% if perms.articles.change_article %}
# Check in code: user.has_perm('articles.change_article')
```

> **Why:** Groups let you manage permissions as roles. Change the group's permissions once, and all members are updated. Much easier than managing per-user permissions.

### @login_required and @permission_required Decorators

**Wrong:**
```python
def dashboard(request):
    if not request.user.is_authenticated:
        return redirect('/login/')
    if not request.user.has_perm('reports.view_dashboard'):
        return HttpResponseForbidden()
    # ...
```

**Correct:**
```python
from django.contrib.auth.decorators import login_required, permission_required


@login_required
def dashboard(request):
    return render(request, 'dashboard.html')


@permission_required('reports.view_dashboard', raise_exception=True)
def admin_dashboard(request):
    return render(request, 'admin_dashboard.html')


# Stacking decorators
@login_required
@permission_required('reports.export', raise_exception=True)
def export_report(request):
    ...
```

> **Why:** Decorators are declarative and consistent. `login_required` redirects to `LOGIN_URL`. `raise_exception=True` returns 403 instead of redirecting to login (for already-authenticated users without permission).

### LoginRequiredMixin and PermissionRequiredMixin

**Wrong:**
```python
from django.views.generic import ListView

class ArticleListView(ListView):
    model = Article

    def dispatch(self, request, *args, **kwargs):
        if not request.user.is_authenticated:
            return redirect('login')
        return super().dispatch(request, *args, **kwargs)
```

**Correct:**
```python
from django.contrib.auth.mixins import LoginRequiredMixin, PermissionRequiredMixin
from django.views.generic import ListView, UpdateView
from .models import Article


class ArticleListView(LoginRequiredMixin, ListView):
    model = Article
    login_url = '/accounts/login/'


class ArticleUpdateView(LoginRequiredMixin, PermissionRequiredMixin, UpdateView):
    model = Article
    fields = ['title', 'body']
    permission_required = 'articles.change_article'
    raise_exception = True
```

> **Why:** Mixins must come before the view class in the inheritance chain (MRO). They handle redirect-to-login and 403 responses automatically.

### LoginRequiredMiddleware (Django 5.1+)

**Wrong:**
```python
# Protecting views one by one — one forgotten view is an open door
class ArticleListView(LoginRequiredMixin, ListView):
    model = Article

class ArticleDetailView(LoginRequiredMixin, DetailView):
    model = Article

class ExportView(View):  # Oops — forgot the mixin, now it's public
    def get(self, request):
        return generate_sensitive_export()
```

**Correct:**
```python
# settings.py — Django 5.1+ makes authentication the default for ALL views
MIDDLEWARE = [
    # ...
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.auth.middleware.LoginRequiredMiddleware',  # After AuthenticationMiddleware
    # ...
]
```

```python
# Opt OUT the few public views instead of opting in every private one
from django.contrib.auth.decorators import login_not_required


@login_not_required
def landing_page(request):
    return render(request, 'landing.html')


@login_not_required
def health_check(request):
    return JsonResponse({'status': 'ok'})
```

> **Why:** For sites where most views require login, `LoginRequiredMiddleware` inverts the default — everything is protected, and public views are explicitly marked with `@login_not_required`. A forgotten decorator now fails closed (login redirect) instead of open (data leak). Note: the middleware respects `login_url` and `redirect_field_name` set via `@login_required`, but NOT the attributes set on `LoginRequiredMixin` — configure those views with the decorator if they need custom redirect behavior.

### Async Authentication (Django 5.1+)

**Wrong:**
```python
# Blocking the event loop in an async view
async def profile(request):
    user = request.user  # Sync lazy attribute — triggers a blocking DB query
    ...
```

**Correct:**
```python
from django.contrib.auth.decorators import login_required


@login_required  # Supports async views natively in Django 5.1+
async def profile(request):
    user = await request.auser()  # Async user access — no blocking query
    return render(request, 'profile.html', {'user': user})


# Async manager methods on the user model
user = await User.objects.acreate_user(email='a@example.com', password='...')
```

> **Why:** In async views, use `await request.auser()` instead of `request.user` to avoid blocking database access. Django 5.1+ adds async support to `@login_required` and async variants like `acreate_user`. See references/async.md for the full async patterns.

## 11. Sessions & Cookies

### Session Middleware Configuration

**Wrong:**
```python
# settings.py — forgetting session middleware or putting it in wrong order
MIDDLEWARE = [
    'django.middleware.common.CommonMiddleware',
    # SessionMiddleware missing — request.session won't work
    'django.contrib.auth.middleware.AuthenticationMiddleware',
]
```

**Correct:**
```python
# settings.py
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',  # Before AuthenticationMiddleware
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',  # Depends on SessionMiddleware
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]
```

> **Why:** `SessionMiddleware` must come before `AuthenticationMiddleware` because auth reads from the session. Middleware order matters — Django processes them top-to-bottom on requests.

### Session Backend Selection

**Wrong:**
```python
# Using database sessions on a high-traffic site
SESSION_ENGINE = 'django.contrib.sessions.backends.db'
# Every request hits the database for session lookup
```

**Correct:**
```python
# Cache-based sessions for high-traffic sites (with Redis)
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
SESSION_CACHE_ALIAS = 'sessions'

CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/0',
    },
    'sessions': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
    },
}

# Or cached_db for durability — cache first, DB fallback
# SESSION_ENGINE = 'django.contrib.sessions.backends.cached_db'
```

> **Why:** Database sessions add a query per request. Cache-based sessions (Redis/Memcached) are faster. Use `cached_db` if you need sessions to survive cache restarts.

### Cookie Security

**Wrong:**
```python
# settings.py
SESSION_COOKIE_SECURE = False  # Cookie sent over HTTP
SESSION_COOKIE_HTTPONLY = False  # JavaScript can read the session cookie
CSRF_COOKIE_HTTPONLY = False
```

**Correct:**
```python
# settings.py — production
SESSION_COOKIE_SECURE = True        # Only sent over HTTPS
SESSION_COOKIE_HTTPONLY = True       # Not accessible via JavaScript
SESSION_COOKIE_SAMESITE = 'Lax'     # CSRF protection for cross-site requests
CSRF_COOKIE_SECURE = True           # CSRF cookie also HTTPS-only
CSRF_COOKIE_HTTPONLY = True          # CSRF cookie not accessible via JS
```

> **Why:** `Secure` prevents cookie theft over HTTP. `HttpOnly` prevents JavaScript access (XSS protection). `SameSite=Lax` blocks most cross-site request forgery attacks.

### Session Expiry

**Wrong:**
```python
# Default session never expires until browser closes
# No explicit expiry configuration — sessions persist indefinitely on server
```

**Correct:**
```python
# settings.py
SESSION_COOKIE_AGE = 60 * 60 * 24 * 7  # 1 week in seconds
SESSION_EXPIRE_AT_BROWSER_CLOSE = False  # Persist across browser sessions
SESSION_SAVE_EVERY_REQUEST = True  # Reset expiry on every request (sliding window)

# Per-view session expiry
def login_view(request):
    # ...
    if form.cleaned_data.get('remember_me'):
        request.session.set_expiry(60 * 60 * 24 * 30)  # 30 days
    else:
        request.session.set_expiry(0)  # Expire when browser closes
```

> **Why:** Set explicit session lifetimes. `SESSION_SAVE_EVERY_REQUEST` creates a sliding window — active users stay logged in. Use `set_expiry()` per-session for remember-me features.

### Session Security

**Wrong:**
```python
# Not rotating session after login — session fixation vulnerability
from django.contrib.auth import login

def login_view(request):
    user = authenticate(request, ...)
    if user:
        # Reuses the same session ID from before login
        login(request, user)  # Django does rotate by default, but...
        request.session.cycle_key = lambda: None  # DON'T disable rotation
```

**Correct:**
```python
from django.contrib.auth import login, authenticate


def login_view(request):
    if request.method == 'POST':
        user = authenticate(request, username=request.POST['username'],
                          password=request.POST['password'])
        if user is not None:
            # Django's login() calls request.session.cycle_key() automatically
            # This rotates the session ID to prevent session fixation
            login(request, user)
            return redirect('dashboard')

# For sensitive operations, manually flush the session
def change_password(request):
    # After password change, invalidate all other sessions
    from django.contrib.auth import update_session_auth_hash
    form = PasswordChangeForm(request.user, request.POST)
    if form.is_valid():
        form.save()
        update_session_auth_hash(request, form.user)  # Keep current session valid
```

> **Why:** Django's `login()` rotates session keys automatically. `update_session_auth_hash()` prevents logout after password change. Never disable session rotation.
