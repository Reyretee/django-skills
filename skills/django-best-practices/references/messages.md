# Django Messages Framework Best Practices

Messages framework best practices: setup, the flash-message-then-redirect flow, message levels, `SuccessMessageMixin`, template rendering, storage backends, and `fail_silently` in reusable apps.

## Setup

**Wrong:**
```python
# MessageMiddleware placed before SessionMiddleware — messages need the session
MIDDLEWARE = [
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    # ...
]

# Or: context processor removed, so {{ messages }} is empty in templates
```

**Correct:**
```python
# All of this is present in default startproject settings — don't remove it
INSTALLED_APPS = [
    # ...
    'django.contrib.sessions',
    'django.contrib.messages',
]

MIDDLEWARE = [
    # ...
    'django.contrib.sessions.middleware.SessionMiddleware',  # Before messages
    'django.middleware.common.CommonMiddleware',
    # ...
    'django.contrib.messages.middleware.MessageMiddleware',
]

TEMPLATES = [{
    # ...
    'OPTIONS': {
        'context_processors': [
            # ...
            'django.contrib.auth.context_processors.auth',
            'django.contrib.messages.context_processors.messages',  # Enables {{ messages }}
        ],
    },
}]
```

> **Why:** The default fallback storage writes to the session, so `MessageMiddleware` must come after `SessionMiddleware`. The `messages` context processor is what puts `{{ messages }}` into every template rendered with a `RequestContext` — all three pieces ship enabled in `startproject`.

## Flash Message Flow: Message, Then Redirect

**Wrong:**
```python
def update_profile(request):
    form = ProfileForm(request.POST or None, instance=request.user.profile)
    if request.method == 'POST' and form.is_valid():
        form.save()
        # Rendering a success page directly on POST: refreshing resubmits
        # the form, and the URL still points at the POST endpoint
        return render(request, 'profile.html', {'form': form, 'saved': True})
    return render(request, 'profile.html', {'form': form})
```

**Correct:**
```python
from django.contrib import messages
from django.shortcuts import redirect, render


def update_profile(request):
    form = ProfileForm(request.POST or None, instance=request.user.profile)
    if request.method == 'POST' and form.is_valid():
        form.save()
        messages.success(request, 'Profile updated.')
        return redirect('profile')  # POST -> Redirect -> GET
    return render(request, 'profile.html', {'form': form})
```

> **Why:** Messages exist precisely to survive a redirect — they're stored, then consumed on the next request. The Post/Redirect/Get pattern prevents duplicate submissions on refresh; rendering a "success" template on POST leaves the browser primed to resubmit.

## Message Levels

**Wrong:**
```python
# Everything is success — including failures
messages.success(request, 'Payment failed. Try again.')
messages.success(request, 'Your account will be deleted in 7 days.')
```

**Correct:**
```python
from django.contrib import messages

messages.debug(request, 'SQL took 0.3s')            # Dev diagnostics (hidden by default)
messages.info(request, 'You have 3 pending invites.')
messages.success(request, 'Profile updated.')
messages.warning(request, 'Your subscription expires in 3 days.')
messages.error(request, 'Payment failed. Try again.')

# Raise the minimum recorded level in production (drops debug/info)
# settings.py
from django.contrib.messages import constants as message_constants
MESSAGE_LEVEL = message_constants.INFO
```

> **Why:** Levels drive both filtering (`MESSAGE_LEVEL` discards anything below it) and presentation — the level's tag becomes a CSS class in templates. Misusing `success` for errors means users see failures styled green.

## SuccessMessageMixin on Generic Views

**Wrong:**
```python
class ProductCreateView(CreateView):
    model = Product
    fields = ['name', 'price']

    # Manually duplicating what the mixin does
    def form_valid(self, form):
        response = super().form_valid(form)
        messages.success(self.request, f'{form.instance.name} was created.')
        return response
```

**Correct:**
```python
from django.contrib.messages.views import SuccessMessageMixin
from django.views.generic import CreateView, UpdateView


class ProductCreateView(SuccessMessageMixin, CreateView):
    model = Product
    fields = ['name', 'price']
    success_message = '%(name)s was created.'  # Interpolated from cleaned_data


class ProductUpdateView(SuccessMessageMixin, UpdateView):
    model = Product
    fields = ['name', 'price']
    success_message = '%(name)s was updated.'

    def get_success_message(self, cleaned_data):
        # For values not in cleaned_data, compute from the saved object
        return f'{self.object.name} (SKU {self.object.sku}) was updated.'
```

> **Why:** `SuccessMessageMixin` adds the message in `form_valid()` for you and interpolates `success_message` against `cleaned_data`. Override `get_success_message(cleaned_data)` when the message needs computed or non-form values. Hand-rolling it in `form_valid` is duplicate code that drifts.

## Rendering Messages in Templates

**Wrong:**
```django
{# Hardcoded single style — errors and successes look identical #}
{% if messages %}
  {% for message in messages %}
    <div class="alert">{{ message }}</div>
  {% endfor %}
{% endif %}
```

**Correct:**
```django
{# base.html — render once, near the top of <body> #}
{% if messages %}
  <ul class="messages">
    {% for message in messages %}
      <li class="alert {% if message.tags %}alert-{{ message.tags }}{% endif %}"
          {% if message.level == DEFAULT_MESSAGE_LEVELS.ERROR %}role="alert"{% endif %}>
        {{ message }}
      </li>
    {% endfor %}
  </ul>
{% endif %}
```

```python
# Map Django tags to your CSS framework's class names if needed
from django.contrib.messages import constants as message_constants

MESSAGE_TAGS = {
    message_constants.ERROR: 'danger',  # Bootstrap uses 'alert-danger'
}
```

> **Why:** `{{ message.tags }}` yields the level tag (`success`, `error`, ...) plus any extra tags, giving you per-level styling for free. Put the loop in the base template — iterating `messages` marks them as read, so render them exactly once per request.

## Storage Backends

**Wrong:**
```python
# Cookie-only storage with big messages — silently dropped past ~2KB
MESSAGE_STORAGE = 'django.contrib.messages.storage.cookie.CookieStorage'

def import_view(request):
    # A multi-KB report crammed into messages
    messages.info(request, '\n'.join(f'Imported row {r}' for r in report_rows))
```

**Correct:**
```python
# Default — usually leave it alone: tries the cookie first (no session
# hit), falls back to the session when messages exceed the ~2KB cookie limit
MESSAGE_STORAGE = 'django.contrib.messages.storage.fallback.FallbackStorage'

# Session-only if you must guarantee capacity (requires sessions everywhere)
MESSAGE_STORAGE = 'django.contrib.messages.storage.session.SessionStorage'

def import_view(request):
    # Keep messages short; store bulky results properly
    report = ImportReport.objects.create(rows=report_rows, user=request.user)
    messages.success(request, f'Imported {len(report_rows)} rows.')
    return redirect('import-report', pk=report.pk)
```

> **Why:** `FallbackStorage` (the default) avoids a session write when messages fit in the cookie and degrades to the session when they don't. Messages are flash notices, not a data channel — anything larger than a sentence belongs in the database or session, not a cookie.

## fail_silently in Reusable Apps

**Wrong:**
```python
# Project code hiding a misconfiguration
def checkout(request):
    messages.success(request, 'Order placed.', fail_silently=True)
    # If the messages framework is broken, you never find out
```

**Correct:**
```python
# Project code: let it raise — MessageFailure means your setup is broken
def checkout(request):
    messages.success(request, 'Order placed.')
    return redirect('order-confirmation')


# Reusable/third-party app: must work even if the host project
# didn't install the messages framework
def notify(request, text):
    messages.info(request, text, fail_silently=True)
```

> **Why:** Without the middleware installed, `messages.add_message()` raises `MessageFailure` — in your own project that's a configuration bug you want to hear about immediately. `fail_silently=True` is only appropriate in reusable apps that can't assume the framework is installed.
