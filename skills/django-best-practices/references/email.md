# Django Email Best Practices

Sending email best practices: `send_mail` basics, HTML email with plain-text fallbacks, per-environment backends, bulk sending with connection reuse, header injection, background delivery, and `fail_silently`.

## send_mail Basics

**Wrong:**
```python
from django.core.mail import send_mail

# Positional optional args — breaks on Django 6.0 (keyword-only)
send_mail('Welcome', 'Thanks for signing up.', 'noreply@example.com',
          ['user@example.com'], False, None, None)

# Hardcoded from address duplicated across the codebase
send_mail('Welcome', 'Thanks!', 'noreply@example.com', [user.email])
```

**Correct:**
```python
from django.core.mail import send_mail

# from_email=None falls back to settings.DEFAULT_FROM_EMAIL
send_mail(
    subject='Welcome',
    message='Thanks for signing up.',
    from_email=None,
    recipient_list=[user.email],
)

# settings.py
DEFAULT_FROM_EMAIL = 'MyApp <noreply@example.com>'
SERVER_EMAIL = 'errors@example.com'  # Sender for error mails to ADMINS
```

> **Why:** As of Django 6.0 the optional arguments to `send_mail` are keyword-only, so positional calls break. Passing `from_email=None` centralizes the sender in `DEFAULT_FROM_EMAIL` instead of scattering addresses through the code.

## HTML Email With a Plain-Text Part

**Wrong:**
```python
# HTML-only email — no text alternative
send_mail(
    subject='Your receipt',
    message='',  # Empty text part
    from_email=None,
    recipient_list=[user.email],
    html_message='<h1>Receipt</h1><p>Total: $42</p>',
)
```

**Correct:**
```python
from django.core.mail import EmailMultiAlternatives
from django.template.loader import render_to_string


def send_receipt(user, order):
    context = {'user': user, 'order': order}
    text_body = render_to_string('emails/receipt.txt', context)
    html_body = render_to_string('emails/receipt.html', context)

    msg = EmailMultiAlternatives(
        subject=f'Receipt for order #{order.pk}',
        body=text_body,  # Plain text is the primary part
        to=[user.email],
    )
    msg.attach_alternative(html_body, 'text/html')
    msg.send()
```

> **Why:** Multipart messages need a real plain-text part — spam filters penalize HTML-only mail and text-mode clients render nothing otherwise. `EmailMultiAlternatives` sends `multipart/alternative` with text as the fallback and HTML as the alternative.

## Backend Per Environment

**Wrong:**
```python
# settings.py — one config for all environments
EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'
EMAIL_HOST = 'smtp.sendgrid.net'
EMAIL_HOST_USER = 'apikey'
EMAIL_HOST_PASSWORD = 'SG.abc123secret'  # Hardcoded credential in VCS
# Dev and test runs send real email to real users
```

**Correct:**
```python
# settings/local.py — print email to the console
EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'

# settings/test.py — capture in memory (Django's test runner does this
# automatically; mail lands in django.core.mail.outbox)
EMAIL_BACKEND = 'django.core.mail.backends.locmem.EmailBackend'

# settings/production.py — real SMTP, credentials from the environment
EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'
EMAIL_HOST = os.environ['EMAIL_HOST']
EMAIL_PORT = 587
EMAIL_HOST_USER = os.environ['EMAIL_HOST_USER']
EMAIL_HOST_PASSWORD = os.environ['EMAIL_HOST_PASSWORD']
EMAIL_USE_TLS = True
EMAIL_TIMEOUT = 10  # Don't let a slow SMTP server hang requests
```

```python
# tests.py
from django.core import mail

def test_signup_sends_welcome_email(client):
    client.post('/signup/', {'email': 'new@example.com', ...})
    assert len(mail.outbox) == 1
    assert mail.outbox[0].to == ['new@example.com']
```

> **Why:** The console backend makes dev email visible without sending anything; locmem lets tests assert on `mail.outbox`. Production SMTP settings belong in environment variables, and `EMAIL_TIMEOUT` prevents a slow provider from blocking workers indefinitely.

## Bulk Sending: Reuse the Connection

**Wrong:**
```python
# Opens and closes an SMTP connection for every recipient
for user in User.objects.filter(newsletter=True):
    send_mail(
        subject='Monthly update',
        message=body,
        from_email=None,
        recipient_list=[user.email],
    )
```

**Correct:**
```python
from django.core import mail
from django.core.mail import EmailMessage


def send_newsletter(body):
    recipients = User.objects.filter(newsletter=True).values_list('email', flat=True)
    messages = [
        EmailMessage(subject='Monthly update', body=body, to=[email])
        for email in recipients
    ]
    # One SMTP connection for the whole batch
    with mail.get_connection() as connection:
        connection.send_messages(messages)
```

> **Why:** Each `send_mail` call opens a fresh SMTP connection — TLS handshake and auth per message. `get_connection()` as a context manager opens one connection, `send_messages()` reuses it for the batch, and it closes cleanly on exit. Build one `EmailMessage` per recipient rather than one message with everyone in `to` (which leaks addresses).

## Header Injection

**Wrong:**
```python
def contact(request):
    subject = request.POST['subject']
    # User submits "Hi\nBcc: victim@example.com" — Django raises an
    # uncaught ValueError and the view 500s
    send_mail(subject=subject, message=request.POST['body'],
              from_email=request.POST['email'],
              recipient_list=['support@example.com'])
```

**Correct:**
```python
from django import forms
from django.core.mail import BadHeaderError, send_mail


class ContactForm(forms.Form):
    subject = forms.CharField(max_length=100)  # CharField rejects newlines
    email = forms.EmailField()
    body = forms.CharField(widget=forms.Textarea)


def contact(request):
    form = ContactForm(request.POST or None)
    if form.is_valid():
        try:
            send_mail(
                subject=form.cleaned_data['subject'],
                message=form.cleaned_data['body'],
                from_email=None,  # Never use user input as the envelope sender
                recipient_list=['support@example.com'],
            )
        except (BadHeaderError, ValueError):
            form.add_error(None, 'Invalid header found.')
    return render(request, 'contact.html', {'form': form})
```

> **Why:** Newlines in subject, from, or recipient values let attackers inject extra headers (`Bcc:`, forged `From:`) to turn your form into a spam relay. Django refuses to send such messages by raising an error — validate user input through forms, keep user data out of header fields, and catch the error instead of 500ing.

## Send From Background Tasks, Not the Request Cycle

**Wrong:**
```python
def signup(request):
    form = SignupForm(request.POST)
    if form.is_valid():
        user = form.save()
        # SMTP round-trip (or provider outage) blocks the HTTP response
        send_welcome_email(user)
        return redirect('dashboard')
```

**Correct:**
```python
# tasks.py — Celery (see references/celery.md), or django.tasks in Django 6.0
from django.tasks import task


@task
def send_welcome_email(user_id):
    user = User.objects.get(pk=user_id)
    send_mail(
        subject='Welcome',
        message=render_to_string('emails/welcome.txt', {'user': user}),
        from_email=None,
        recipient_list=[user.email],
    )


def signup(request):
    form = SignupForm(request.POST)
    if form.is_valid():
        user = form.save()
        send_welcome_email.enqueue(user.pk)  # Returns immediately
        return redirect('dashboard')
```

> **Why:** SMTP is slow and flaky — sending inline adds seconds to responses and turns a mail-provider outage into a signup outage. Queue the send in a background task (Celery, or Django 6.0's built-in `django.tasks`) and pass IDs, not model instances.

## Never Default to fail_silently=True

**Wrong:**
```python
# Order confirmations vanish without a trace when SMTP is down
send_mail(
    subject='Order confirmed',
    message=body,
    from_email=None,
    recipient_list=[user.email],
    fail_silently=True,  # Errors swallowed — nobody notices for weeks
)
```

**Correct:**
```python
# Default: let it raise so the task queue retries and monitoring alerts
send_mail(
    subject='Order confirmed',
    message=body,
    from_email=None,
    recipient_list=[user.email],
)

# fail_silently only for truly optional, best-effort notifications
send_mail(
    subject='Someone liked your post',
    message=body,
    from_email=None,
    recipient_list=[user.email],
    fail_silently=True,  # Losing this mail is acceptable
)
```

> **Why:** `fail_silently=True` swallows SMTP exceptions, so critical mail (receipts, password resets) can silently stop for weeks. Let errors propagate so retries and error monitoring catch them; reserve `fail_silently` for notifications nobody will miss.
