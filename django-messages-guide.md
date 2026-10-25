# Django Messages (`django.contrib.messages`): Step-by-Step Guide

You've seen it on almost every website: you submit a form, the page reloads, and a green bar at the top says **"Your post was saved."** Refresh the page and the bar is gone. That is a **one-time notification**, sometimes called a **flash message**. `django.contrib.messages` is Django's built-in app for exactly this: **store a short message during one request and show it once, on the next page the user sees**.

It gives you:

| Part                                         | What it does                                                                                           |
|----------------------------------------------|--------------------------------------------------------------------------------------------------------|
| **`messages.success()` / `info()` / …**      | One-line functions to add a message: `messages.success(request, 'Saved!')`                             |
| **`messages.add_message()`**                 | The general function behind the shortcuts (any level, including your own custom levels)                |
| **Message levels**                           | `DEBUG` (10), `INFO` (20), `SUCCESS` (25), `WARNING` (30), `ERROR` (40), plus any you invent           |
| **Message tags**                             | CSS-friendly names for each level (`'success'`, `'error'`…), customisable with `MESSAGE_TAGS`          |
| **`extra_tags`**                             | Extra CSS classes / markers you attach to one message (`'dismissible'`, `'toast'`…)                    |
| **`MessageMiddleware`**                      | Attaches the message storage to every request and saves unread messages on every response              |
| **3 storage backends**                       | Where messages wait between requests: a signed **cookie**, the **session**, or both (**fallback**)     |
| **`messages` context processor**             | Puts `messages` and `DEFAULT_MESSAGE_LEVELS` into every template                                       |
| **`get_messages()`**                         | Read the messages in Python code (views, JSON APIs, tests)                                             |
| **`get_level()` / `set_level()`**            | Change the minimum level that gets recorded, per request                                               |
| **`SuccessMessageMixin`**                    | Adds a success message to class-based form views (`CreateView`, `UpdateView`, `DeleteView`, `FormView`) |
| **`MessagesTestMixin.assertMessages()`**     | Test helper to check which messages a response produced (Django 5.0+)                                  |
| **`ModelAdmin.message_user()`**              | The admin's own way of sending messages (built on this app)                                            |

Four words you will see a lot:

- **Message** = one notification: a **text** + a **level** (+ optional **extra tags**). In Python it's a `Message` object.
- **Level** = an integer that says how important the message is (`25` = success, `40` = error). Higher = more important.
- **Tags** = the text version of the level plus any extra tags, e.g. `'success'` or `'dismissible error'`. You use them as CSS classes.
- **Storage** = the place messages wait between "added" and "shown": a cookie called `messages`, the session key `_messages`, or both.

This guide continues the `blog` project from [django-setup-guide.md](django-setup-guide.md), [django-admin-guide.md](django-admin-guide.md), [django-auth-guide.md](django-auth-guide.md), [django-contenttypes-guide.md](django-contenttypes-guide.md) and [django-sessions-guide.md](django-sessions-guide.md) (project package `config`, apps `blog`, `accounts`, `social` and `usersessions`, custom user `accounts.User`, `Post` with `title`, `slug`, `content`, `author` and `status`). Every step builds on the previous one. Commands are for **Windows** (PowerShell); they are the same on macOS/Linux except where noted.

---

## Prerequisites

- The project from the earlier guides, with the virtual environment **activated** (`(venv)` in your prompt).
- **Django 5.0+** recommended (needed for `assertMessages()` in Step 33). Everything else works on any supported Django version. Check with:
  ```bash
  python -m django --version
  ```
- Migrations applied (`python manage.py migrate`) and a superuser created.
- A browser with **Developer Tools** (Chrome, Edge or Firefox — press `F12`). You'll use the **Application** (Chrome/Edge) or **Storage** (Firefox) tab to look at cookies.
- If you enabled `LoginRequiredMiddleware` in the auth guide (Step 21), every new **public** view in this guide needs the `@login_not_required` decorator, otherwise anonymous visitors are redirected to the login page. The demo views below show it.

---

## Step 0: How Messages Work (Big Picture)

### 0.1 The problem — the Post/Redirect/Get pattern

When a user submits a form, the correct thing to do is to **redirect** after a successful POST. This is called **Post/Redirect/Get (PRG)**:

```
Browser                         Django
───────                         ──────
POST /post/new/  ─────────────► view saves the post
                 ◄───────────── 302 Redirect → /post/7/
GET /post/7/     ─────────────► view renders the post page
                 ◄───────────── 200 OK (HTML)
```

Why redirect? If you rendered the page directly after the POST, pressing **F5** would re-send the POST and create the post **twice**. After a redirect, F5 just reloads `/post/7/`.

But now there's a problem: the view that **knows** something happened ("the post was created") is the POST view. The view that **renders the page** is a different view, in a **different request**. HTTP is stateless, so the second request knows nothing about the first.

You need somewhere to put "Post created!" during request 1 so request 2 can display it **once**. That's what `django.contrib.messages` does.

### 0.2 The solution — store, redirect, display, forget

```
  BROWSER                                              DJANGO SERVER
  ───────                                              ─────────────

  ① POST /post/new/  ─────────────────────────────────► MessageMiddleware (request):
                                                          request._messages = FallbackStorage(request)
                                                        view:
                                                          post.save()
                                                          messages.success(request, 'Post created.')
                                                            → queued in memory (not stored yet)
                                                          return redirect('/post/7/')
                                                        MessageMiddleware (response):
                                                          storage.update(response)
                                                          → there is 1 unread message → write it to a cookie
                     ◄───────────────────────────────── 302 Found
                                                        Location: /post/7/
                                                        Set-Cookie: messages=W1siX19qc29u…:1xKscL:9JSY…; HttpOnly; Path=/; SameSite=Lax

  ② GET /post/7/
     Cookie: messages=W1siX19qc29u…  ─────────────────► MessageMiddleware (request):
                                                          request._messages = FallbackStorage(request)
                                                        view renders post_detail.html
                                                        template: {% for message in messages %}
                                                          → storage reads + verifies the cookie
                                                          → yields Message(level=25, 'Post created.')
                                                          → storage marks itself "used"
                                                        MessageMiddleware (response):
                                                          storage.update(response)
                                                          → messages were used, nothing new → delete the cookie
                     ◄───────────────────────────────── 200 OK  (HTML with the green bar)
                                                        Set-Cookie: messages=""; expires=Thu, 01 Jan 1970 …; Max-Age=0

  ③ F5 / GET /post/7/  (no messages cookie) ──────────► nothing to show
                     ◄───────────────────────────────── 200 OK  (no green bar)
```

### 0.3 The four rules that explain everything

1. **Adding a message only queues it in memory.** Nothing is written anywhere until the response goes out.
2. **On the response, `MessageMiddleware` saves every message that hasn't been displayed yet** (in a cookie, the session, or both).
3. **A message is "displayed" when the storage is iterated** (`{% for message in messages %}` or `for m in get_messages(request)`). After that request, the iterated messages are deleted.
4. **If nobody iterates the messages, they stay** and wait for the next request — they are not lost just because one page didn't show them.

Keep these four rules in mind; every "why did my message show up there / disappear?" question in the Troubleshooting section comes back to them.

### 0.4 Messages vs. other ways of telling the user something

| Need                                                              | Use                                                         |
|-------------------------------------------------------------------|-------------------------------------------------------------|
| "Saved!", "Deleted.", "You're logged in." — **once**, after an action | **`django.contrib.messages`** (this guide)               |
| "This field is required." — next to a form field                  | **Form errors** (`form.errors`, `{{ form.field.errors }}`)  |
| "You have 3 unread notifications" — stays until the user reads it | A **database model** (`Notification` with `is_read`)        |
| Data you need across many requests (cart, theme)                  | **Sessions** (see [django-sessions-guide.md](django-sessions-guide.md)) |
| Something for developers                                          | **Logging** (`logging.getLogger(__name__)`)                 |

Messages are **temporary, one-time and per-browser**. They are not a notification inbox, and they are not a replacement for form validation errors.

---

## Step 1: Check That Messages Are Enabled

`django-admin startproject` turns messages on by default. You need **three** things in `config/settings.py`. Open it and check each one.

### 1.1 The app in `INSTALLED_APPS`

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',      # 👈 this one
    'django.contrib.staticfiles',
    'blog',
    'accounts',
    'social',
    'usersessions',
]
```

The app itself has **no models and no migrations**. Being in `INSTALLED_APPS` does two things: the admin's system check sees it (the admin requires it), and its `AppConfig.ready()` connects a signal that refreshes the level tags when `MESSAGE_TAGS` is changed in tests (Step 9.4).

### 1.2 The middleware in `MIDDLEWARE` — order matters

```python
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',       # ① must be ABOVE messages
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',       # ② this one
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
    # 'django.contrib.auth.middleware.LoginRequiredMiddleware',   # if you added it in the auth guide
    # 'accounts.middleware.IdleTimeoutMiddleware',                # if you added it in the sessions guide
]
```

Why `SessionMiddleware` must come first: the default storage (`FallbackStorage`) uses the **session** when messages don't fit in the cookie. Middleware runs **top → bottom** on the way in, so `request.session` must exist before `MessageMiddleware` builds the storage. On the way **out**, middleware runs **bottom → top**, so `MessageMiddleware` writes `_messages` into the session **before** `SessionMiddleware` saves the session. Both directions only work with this order.

Any middleware that adds messages (like `IdleTimeoutMiddleware` in the sessions guide) must come **below** `MessageMiddleware`, because `request._messages` doesn't exist until `MessageMiddleware` has run.

### 1.3 The context processor in `TEMPLATES`

```python
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [BASE_DIR / 'templates'],
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',   # 👈 this one
                # 'blog.context_processors.session_extras',              # from the sessions guide
            ],
        },
    },
]
```

This makes two variables available in **every** template rendered with a request (`render(request, …)`, class-based views):

| Variable                   | Value                                                                                       |
|----------------------------|---------------------------------------------------------------------------------------------|
| `messages`                 | The message storage for this request (iterate it to get `Message` objects)                  |
| `DEFAULT_MESSAGE_LEVELS`   | `{'DEBUG': 10, 'INFO': 20, 'SUCCESS': 25, 'WARNING': 30, 'ERROR': 40}` — for comparisons    |

### 1.4 What happens if something is missing

| Missing                                                         | What you get                                                                                                                                       |
|-----------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------|
| `'django.contrib.messages'` in `INSTALLED_APPS` (with admin)    | `admin.E406: 'django.contrib.messages' must be in INSTALLED_APPS in order to use the admin application.`                                           |
| `MessageMiddleware` (with admin)                                | `admin.E409: 'django.contrib.messages.middleware.MessageMiddleware' must be in MIDDLEWARE in order to use the admin application.`                  |
| `MessageMiddleware` (when you add a message)                    | `MessageFailure: You cannot add messages without installing django.contrib.messages.middleware.MessageMiddleware`                                  |
| The context processor (with admin)                              | `admin.E404: 'django.contrib.messages.context_processors.messages' must be enabled in DjangoTemplates (TEMPLATES) in order to use the admin application.` |
| The context processor (your templates)                          | No error — `{% for message in messages %}` silently shows nothing, and messages pile up because nobody reads them                                   |
| `SessionMiddleware`, or it's **below** `MessageMiddleware`      | `ImproperlyConfigured: The session-based temporary message storage requires session middleware to be installed, and come before the message middleware in the MIDDLEWARE list.` |

### 1.5 Verify

```bash
python manage.py check
```

Expected:

```
System check identified no issues (0 silenced).
```

You can also check from the shell:

```bash
python manage.py shell
```

```python
>>> from django.conf import settings
>>> 'django.contrib.messages' in settings.INSTALLED_APPS
True
>>> settings.MIDDLEWARE.index('django.contrib.sessions.middleware.SessionMiddleware') < \
...     settings.MIDDLEWARE.index('django.contrib.messages.middleware.MessageMiddleware')
True
>>> settings.MESSAGE_STORAGE
'django.contrib.messages.storage.fallback.FallbackStorage'
>>> exit()
```

---

## Step 2: Know What the App Contains

Everything lives in the `django/contrib/messages/` folder of your Django install. You import almost everything from **one** place: `from django.contrib import messages`.

| Module                                        | Contains                                                                                                  |
|-----------------------------------------------|-----------------------------------------------------------------------------------------------------------|
| `django.contrib.messages` (the package)       | Re-exports everything below: `messages.success`, `messages.SUCCESS`, `messages.Message`, …                 |
| `messages.api`                                | `add_message()`, `debug()`, `info()`, `success()`, `warning()`, `error()`, `get_messages()`, `get_level()`, `set_level()`, `MessageFailure` |
| `messages.constants`                          | `DEBUG`, `INFO`, `SUCCESS`, `WARNING`, `ERROR`, `DEFAULT_TAGS`, `DEFAULT_LEVELS`                           |
| `messages.storage`                            | `default_storage(request)` — builds the class named in `MESSAGE_STORAGE`                                  |
| `messages.storage.base`                       | `Message`, `BaseStorage`, `LEVEL_TAGS`                                                                    |
| `messages.storage.cookie`                     | `CookieStorage`, `MessageEncoder`, `MessageDecoder`                                                       |
| `messages.storage.session`                    | `SessionStorage`                                                                                          |
| `messages.storage.fallback`                   | `FallbackStorage` (the default)                                                                           |
| `messages.middleware`                         | `MessageMiddleware`                                                                                       |
| `messages.context_processors`                 | `messages()` — the context processor                                                                      |
| `messages.views`                              | `SuccessMessageMixin`                                                                                     |
| `messages.test`                               | `MessagesTestMixin` (Django 5.0+)                                                                         |
| `messages.utils`                              | `get_level_tags()` — merges `DEFAULT_TAGS` with your `MESSAGE_TAGS`                                       |
| `messages.apps`                               | `MessagesConfig` (`verbose_name = 'Messages'`)                                                            |

What it does **not** contain:

- **No database tables.** `python manage.py migrate` creates nothing for messages. With `SessionStorage`, messages live inside the existing `django_session` table.
- **No admin pages.** You can't browse messages in the admin — they are temporary by design.
- **No JavaScript or CSS.** You style messages yourself (Steps 23–24).

Take a look at the code — it's short and readable:

```bash
python -c "import django.contrib.messages, os; print(os.path.dirname(django.contrib.messages.__file__))"
```

Open that folder in VS Code. `api.py` and `storage/base.py` are worth reading once; the whole app is a few hundred lines.

---

## Step 3: Your First Message

Let's add a message in one view, redirect, and display it in another.

### 3.1 The views

Add to `blog/views.py` (keep the existing imports and views; `messages`, `redirect` and `render` are probably already imported from the auth guide):

```python
from django.contrib import messages
from django.contrib.auth.decorators import login_not_required   # Django 5.1+, only if you use LoginRequiredMiddleware
from django.shortcuts import redirect, render


@login_not_required
def messages_demo(request):
    """Shows the page. Any waiting messages are displayed by base.html."""
    return render(request, 'blog/messages_demo.html')


@login_not_required
def messages_add(request):
    """Adds a message, then redirects (Post/Redirect/Get)."""
    if request.method == 'POST':
        messages.success(request, 'Hello! This is your first message.')
    return redirect('blog:messages_demo')
```

Line by line:

- `from django.contrib import messages` — the one import you need. By convention the module is called `messages`, so **don't** name a local variable `messages` in the same function (Step 17.4).
- `messages.success(request, '…')` — the **first argument is always the request**. The message is queued on `request._messages`.
- `return redirect(...)` — the message is saved on **this** redirect response and displayed on the **next** page.

> If you don't use `LoginRequiredMiddleware`, leave out the `login_not_required` import and decorators.

### 3.2 The URLs

Add to `blog/urls.py` inside `urlpatterns`:

```python
path('messages-demo/', views.messages_demo, name='messages_demo'),
path('messages-demo/add/', views.messages_add, name='messages_add'),
```

### 3.3 The template

Create `blog/templates/blog/messages_demo.html`:

```html
{% extends 'blog/base.html' %}

{% block title %}Messages demo{% endblock %}

{% block content %}
    <h2>Messages demo</h2>

    <form method="post" action="{% url 'blog:messages_add' %}">
        {% csrf_token %}
        <button type="submit">Add a message</button>
    </form>

    <p><a href="{% url 'blog:messages_demo' %}">Reload this page</a></p>
{% endblock %}
```

Your `blog/templates/blog/base.html` from the auth guide (Step 14) already contains the display loop:

```html
{% if messages %}
    <ul class="messages">
        {% for message in messages %}
            <li class="{{ message.tags }}">{{ message }}</li>
        {% endfor %}
    </ul>
{% endif %}
```

If your `base.html` doesn't have it, add it right above `{% block content %}{% endblock %}`. Step 4 explains every line.

### 3.4 Try it

```bash
python manage.py runserver
```

1. Open http://127.0.0.1:8000/messages-demo/ — no message.
2. Click **Add a message** → you're redirected back and see **"Hello! This is your first message."**
3. Click **Reload this page** (or press F5) → the message is **gone**. It was shown once.
4. Click **Add a message** several times. You see **one** message each time: every click adds one message, redirects, and the page displays it straight away. To see several messages at once, add them in the same request (Step 3.5).

### 3.5 Several messages at once

Change `messages_add` temporarily:

```python
@login_not_required
def messages_add(request):
    if request.method == 'POST':
        messages.debug(request, 'A debug message (hidden by default — see Step 8).')
        messages.info(request, 'An info message.')
        messages.success(request, 'A success message.')
        messages.warning(request, 'A warning message.')
        messages.error(request, 'An error message.')
    return redirect('blog:messages_demo')
```

Click the button. You see **four** messages, **in the order they were added**. The debug one is missing — `DEBUG` (10) is below the default minimum level `INFO` (20). Step 8 explains how to change that.

### 3.6 Look at the cookie

1. Open DevTools (`F12`) → **Application** → **Cookies** → `http://127.0.0.1:8000`.
2. In the **Network** tab, click **Add a message** and select the `add/` request (status **302**). Under **Response Headers** you'll see something like this (your value is different — it depends on the text and your `SECRET_KEY`):

   ```
   Set-Cookie: messages=W1siX19qc29uX21lc3NhZ2UiLDAsMjUsIlNhdmVkISIsIiJdXQ:1xKscL:9JSYD8LW…; HttpOnly; Path=/; SameSite=Lax
   ```

3. Select the next request (`messages-demo/`, status **200**). Its **Request Headers** contain `Cookie: messages=…`, and its **Response Headers** contain a `Set-Cookie: messages=""; expires=Thu, 01 Jan 1970 00:00:00 GMT; Max-Age=0; Path=/` — that **deletes** the cookie because the messages were displayed.

Step 13 decodes that cookie value piece by piece.

### 3.7 Without a redirect — show it on the same page

Messages also work **without** a redirect. If you add a message and then render a template in the **same** request, it's displayed immediately:

```python
def messages_same_request(request):
    messages.info(request, 'Shown right now, on this very page.')
    return render(request, 'blog/messages_demo.html')
```

Iterating the storage first yields the messages loaded from the cookie/session, then the ones added during this request. Nothing is stored, because everything was displayed. This is useful when you re-render a form after an error (Step 19), but for **successful** POSTs always redirect.

---

## Step 4: Display Messages in Templates — Every Detail

### 4.1 The basic loop, line by line

```html
{% if messages %}
    <ul class="messages">
        {% for message in messages %}
            <li class="{{ message.tags }}">{{ message }}</li>
        {% endfor %}
    </ul>
{% endif %}
```

| Code                               | What happens                                                                                                                                 |
|------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------|
| `{% if messages %}`                | Calls `len(storage)`. This **loads** the messages (reads the cookie/session) but does **not** mark them as displayed                           |
| `{% for message in messages %}`    | **Iterates** the storage → sets `storage.used = True` → these messages will be **deleted** at the end of the request                          |
| `{{ message }}`                    | `Message.__str__()` returns the text. **Auto-escaped** like any variable (Step 25)                                                           |
| `{{ message.tags }}`               | `'success'`, `'error'`, `'dismissible warning'`… — the level tag plus extra tags, separated by a space                                        |

### 4.2 Everything you can use on a message in a template

| Attribute               | Example value          | Notes                                                                       |
|-------------------------|------------------------|-----------------------------------------------------------------------------|
| `{{ message }}`         | `Post created.`        | Same as `{{ message.message }}`                                             |
| `{{ message.message }}` | `Post created.`        | The raw text (a `str`, or a safe string if you used `mark_safe`)            |
| `{{ message.level }}`   | `25`                   | The integer level                                                           |
| `{{ message.level_tag }}` | `success`            | Only the level's tag (from `DEFAULT_TAGS` + `MESSAGE_TAGS`); `''` for unknown levels |
| `{{ message.extra_tags }}` | `dismissible`       | Only the extra tags you passed; `''` by default                             |
| `{{ message.tags }}`    | `dismissible success`  | `extra_tags` + `level_tag`, joined with a space, empty parts skipped        |

### 4.3 Compare levels with `DEFAULT_MESSAGE_LEVELS`

```html
{% for message in messages %}
    <li class="{{ message.tags }}">
        {% if message.level == DEFAULT_MESSAGE_LEVELS.ERROR %}⛔{% endif %}
        {% if message.level == DEFAULT_MESSAGE_LEVELS.WARNING %}⚠️{% endif %}
        {% if message.level == DEFAULT_MESSAGE_LEVELS.SUCCESS %}✅{% endif %}
        {{ message }}
    </li>
{% endfor %}
```

`DEFAULT_MESSAGE_LEVELS` comes from the context processor. Compare **levels**, not tags — tags can be renamed with `MESSAGE_TAGS` (Step 9), levels can't.

You can also compare with `>=` — "anything at warning level or higher":

```html
{% if message.level >= DEFAULT_MESSAGE_LEVELS.WARNING %}<strong>{{ message }}</strong>{% else %}{{ message }}{% endif %}
```

### 4.4 Put the loop in **one** place — an include

Every page should show messages, and the loop should exist only once. Create `blog/templates/blog/includes/messages.html`:

```html
{% if messages %}
    <ul class="messages" role="status" aria-live="polite">
        {% for message in messages %}
            <li{% if message.tags %} class="{{ message.tags }}"{% endif %}>{{ message }}</li>
        {% endfor %}
    </ul>
{% endif %}
```

And replace the loop in `blog/templates/blog/base.html` with:

```html
    <hr>

    {% include 'blog/includes/messages.html' %}

    {% block content %}{% endblock %}
```

- `role="status"` + `aria-live="polite"` makes screen readers announce the message (Step 23.5).
- `{% if message.tags %} class="…"{% endif %}` avoids an empty `class=""` attribute for custom levels without a tag. This is exactly how the admin does it.

### 4.5 Where to put the include

- **Inside `base.html`, above the content** — the normal choice. Every page that extends `base.html` displays messages.
- **Not inside a `{% block %}` that child templates override** — if a child overrides that block without `{{ block.super }}`, its pages silently stop showing messages (and messages pile up until a page does show them).
- **Only once per page.** Within one request the storage yields the same list every time you iterate it, so a second loop shows the messages a second time. Keep one loop. (Step 28.4 shows the one exception: two loops that each show a different kind of message.)

### 4.6 Templates that **don't** extend `base.html`

Login pages, error pages, emails and standalone pages don't get the include automatically. Either add `{% include 'blog/includes/messages.html' %}` to them, or accept that messages wait until the next page that does display them (rule 4 in Step 0.3).

Templates rendered **without a request** (`render_to_string('x.html', context)` without `request=`, email templates, `Template(...).render(Context(...))`) **don't run context processors**, so `messages` doesn't exist there. Pass `request=request` if you need it.

### 4.7 Use messages in Jinja2 templates (optional)

Jinja2 doesn't run Django context processors. Add `get_messages` to your Jinja2 environment:

```python
# config/jinja2.py
from django.contrib.messages import get_messages
from django.templatetags.static import static
from django.urls import reverse
from jinja2 import Environment


def environment(**options):
    env = Environment(**options)
    env.globals.update({'static': static, 'url': reverse, 'get_messages': get_messages})
    return env
```

```html
{% for message in get_messages(request) %}
    <li class="{{ message.tags }}">{{ message }}</li>
{% endfor %}
```

---

## Step 5: Message Levels and the Functions That Add Them

### 5.1 The five built-in levels

| Constant              | Value | Default tag   | Shortcut function           | Use it for                                              |
|-----------------------|-------|---------------|-----------------------------|---------------------------------------------------------|
| `messages.DEBUG`      | 10    | `'debug'`     | `messages.debug()`          | Developer info — **ignored in production** by default   |
| `messages.INFO`       | 20    | `'info'`      | `messages.info()`           | Neutral information: "You have 2 drafts."               |
| `messages.SUCCESS`    | 25    | `'success'`   | `messages.success()`        | An action worked: "Post published."                     |
| `messages.WARNING`    | 30    | `'warning'`   | `messages.warning()`        | Worked, but watch out: "Your password expires soon."    |
| `messages.ERROR`      | 40    | `'error'`     | `messages.error()`          | An action failed: "You can't edit this post."           |

Why the gaps (25 between 20 and 30)? So you can insert your own levels (Step 10). The numbers match Python's `logging` levels (`DEBUG=10`, `INFO=20`, `WARNING=30`, `ERROR=40`), with `SUCCESS` squeezed in between.

### 5.2 The shortcut functions

All five have the same signature:

```python
messages.success(request, message, extra_tags='', fail_silently=False)
```

| Argument         | Meaning                                                                                                   |
|------------------|-----------------------------------------------------------------------------------------------------------|
| `request`        | The current `HttpRequest`. **Not** `self` in a class-based view — use `self.request`                      |
| `message`        | The text. A `str`, a lazy translation (`gettext_lazy`), or a safe string (`mark_safe`/`format_html`)       |
| `extra_tags`     | A string of extra tags (space-separated) added to `message.tags` (Step 7)                                 |
| `fail_silently`  | `True` → don't raise `MessageFailure` if `MessageMiddleware` isn't installed (Step 29)                    |

```python
messages.debug(request, f'{Post.objects.count()} posts in the database.')
messages.info(request, 'Three new comments since your last visit.')
messages.success(request, 'Profile details updated.')
messages.warning(request, 'Your account expires in three days.')
messages.error(request, 'Document deleted.')
```

They all return `None`.

### 5.3 The general function — `add_message()`

```python
messages.add_message(request, level, message, extra_tags='', fail_silently=False)
```

The shortcuts are just `add_message()` with a fixed level. These two lines are identical:

```python
messages.add_message(request, messages.INFO, 'Hello world.')
messages.info(request, 'Hello world.')
```

Use `add_message()` when the level is in a variable or is a **custom** level:

```python
level = messages.SUCCESS if post.status == 'published' else messages.INFO
messages.add_message(request, level, f'"{post.title}" saved ({post.get_status_display()}).')
```

### 5.4 What `add_message()` does, exactly

From `django/contrib/messages/api.py` and `storage/base.py`:

1. Looks for `request._messages` (created by `MessageMiddleware`).
2. If it doesn't exist:
   - and `request` isn't a request at all (no `META` attribute — e.g. you passed `self`) → `TypeError: add_message() argument must be an HttpRequest object, not 'PostCreateView'.`
   - and `fail_silently=False` → `MessageFailure`.
   - and `fail_silently=True` → does nothing.
3. Calls `storage.add(level, message, extra_tags)`, which:
   - **ignores empty messages** (`''`, `None`) — nothing is added, no error.
   - converts `level` to `int` (`'25'` works, `'success'` raises `ValueError`).
   - **ignores the message if `level < storage.level`** (the minimum level, Step 8) — no error.
   - otherwise creates `Message(level, message, extra_tags=extra_tags)` and appends it to the in-memory queue.

So `messages.success(request, '')` and `messages.debug(...)` in production both **silently do nothing**. If a message "doesn't appear", check these two first.

### 5.5 Messages are per browser, not per user

Messages are stored in **this browser's** cookie/session. You **cannot** send a message to another user ("notify the author that someone commented") — the author's browser isn't part of this request. For that, use a database model (Step 32).

---

## Step 6: The `Message` Object

When you iterate the storage you get `Message` objects (`django.contrib.messages.storage.base.Message`, also importable as `django.contrib.messages.Message`).

### 6.1 Attributes and properties

```python
class Message:
    def __init__(self, level, message, extra_tags=None):
        self.level = int(level)
        self.message = message
        self.extra_tags = extra_tags
```

| Name          | Type      | What it is                                                                                  |
|---------------|-----------|---------------------------------------------------------------------------------------------|
| `level`       | `int`     | 10, 20, 25, 30, 40 or a custom number                                                       |
| `message`     | `str`     | The text (lazy translations are turned into `str` before storing)                           |
| `extra_tags`  | `str`     | Extra tags, `''` by default when added via the API                                          |
| `level_tag`   | property  | The tag for `level` from `DEFAULT_TAGS` + `MESSAGE_TAGS`, or `''` if the level has no tag    |
| `tags`        | property  | `' '.join([extra_tags, level_tag])`, skipping empty parts                                   |
| `str(msg)`    | method    | Returns `msg.message`                                                                       |
| `repr(msg)`   | method    | `Message(level=25, message='Saved!')` (+ `extra_tags=…` when set)                            |
| `msg1 == msg2`| method    | `True` when **level and message text** are equal (extra tags are **not** compared)           |

### 6.2 Play with it in the shell

```bash
python manage.py shell
```

```python
>>> from django.contrib import messages
>>> from django.contrib.messages import Message
>>> m = Message(messages.SUCCESS, 'Saved!')
>>> m
Message(level=25, message='Saved!')
>>> str(m), m.level, m.level_tag, m.tags, m.extra_tags
('Saved!', 25, 'success', 'success', None)
>>> m2 = Message(messages.ERROR, 'Nope', extra_tags='dismissible')
>>> m2.tags
'dismissible error'
>>> Message(50, 'Custom level').tags          # no tag defined for 50
''
>>> Message(25, 'Saved!') == Message(25, 'Saved!', extra_tags='x')
True
>>> exit()
```

You will rarely create `Message` objects yourself — except in **tests** (Step 33) and **custom storage** (Step 31).

---

## Step 7: Extra Tags

`extra_tags` lets you attach your own markers to a single message. They are added **in front of** the level tag in `message.tags`.

### 7.1 Add extra tags

```python
messages.success(request, 'Post published.', extra_tags='dismissible')
# message.tags == 'dismissible success'

messages.warning(request, 'Saved as draft.', extra_tags='dismissible sticky')
# message.tags == 'dismissible sticky warning'

messages.add_message(request, messages.INFO, 'Welcome!', extra_tags='toast')
# message.tags == 'toast info'
```

Separate several tags with **spaces** — they end up in an HTML `class` attribute.

### 7.2 Use them in the template

```html
{% for message in messages %}
    <li class="{{ message.tags }}">
        {{ message }}
        {% if 'dismissible' in message.extra_tags %}
            <button type="button" class="close" aria-label="Close">&times;</button>
        {% endif %}
    </li>
{% endfor %}
```

- Check `message.extra_tags` (only your tags), not `message.tags`, when looking for **your** markers.
- `'x' in message.extra_tags` is a **substring** check on a string: `'toast' in 'toaster'` is `True`. Use distinct names, or split them in a template filter if that matters.

### 7.3 Typical uses

| Extra tag        | Meaning in your templates / JS                                     |
|------------------|--------------------------------------------------------------------|
| `dismissible`    | Show a × close button                                              |
| `sticky`         | Don't auto-hide this one (Step 23.4)                               |
| `toast`          | Show as a floating toast instead of an inline bar                  |
| `safe`           | Don't use it — use `mark_safe`/`format_html` instead (Step 25)     |
| `form-top`       | Display near the form instead of at the top of the page            |

---

## Step 8: The Minimum Level — `MESSAGE_LEVEL`, `get_level()`, `set_level()`

### 8.1 Default: `INFO`

Messages below the **minimum level** are thrown away when added. The default is `INFO` (20), so `messages.debug()` does nothing out of the box.

### 8.2 Change it for the whole site — `MESSAGE_LEVEL`

Show debug messages while developing. In `config/settings.py`:

```python
from django.contrib.messages import constants as message_constants

MESSAGE_LEVEL = message_constants.DEBUG if DEBUG else message_constants.INFO
```

> **Import `constants`, not `django.contrib.messages`, in `settings.py`.** `from django.contrib import messages` loads `messages.api` → `messages.storage`, which reads settings while settings are still being loaded. Importing `django.contrib.messages.constants` is always safe. You can also just write the number: `MESSAGE_LEVEL = 10`.

Restart the server, click **Add a message** in the demo (with the five-message version from Step 3.5) — now you see all five.

### 8.3 Change it for one request — `set_level()`

```python
messages.set_level(request, messages.DEBUG)     # record everything from now on, in this request
messages.debug(request, 'Recorded now.')

messages.set_level(request, messages.WARNING)   # only warnings and errors from now on
messages.info(request, 'Ignored.')

messages.set_level(request, None)               # back to MESSAGE_LEVEL
```

- `set_level()` returns `True` if it worked, `False` if there's no message storage on the request (no middleware).
- The level is stored **on the storage object for this request only**. It is not saved anywhere; the next request starts again with `MESSAGE_LEVEL`.
- It only affects messages **added afterwards** in this request. Messages already queued stay.

Real example — show debug messages to superusers only, even in production:

```python
def post_list(request):
    if request.user.is_superuser:
        messages.set_level(request, messages.DEBUG)
    posts = ...
    messages.debug(request, f'{len(posts)} posts, query took …')
    ...
```

Or for every request, with a tiny middleware in `blog/middleware.py`:

```python
from django.contrib import messages


class SuperuserDebugMessagesMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        if request.user.is_authenticated and request.user.is_superuser:
            messages.set_level(request, messages.DEBUG)
        return self.get_response(request)
```

Add `'blog.middleware.SuperuserDebugMessagesMiddleware'` to `MIDDLEWARE` **below** `AuthenticationMiddleware` and `MessageMiddleware` (it needs `request.user` and `request._messages`).

### 8.4 Read it — `get_level()`

```python
current = messages.get_level(request)     # e.g. 20
if current <= messages.DEBUG:
    messages.debug(request, expensive_debug_summary())   # only build the text if it will be kept
```

---

## Step 9: Rename the Tags — `MESSAGE_TAGS`

### 9.1 Why

The tag is what ends up in `class="…"`. Your CSS framework might use different names. **Bootstrap** calls the red alert `alert-danger`, not `error`.

### 9.2 How

In `config/settings.py`:

```python
from django.contrib.messages import constants as message_constants

MESSAGE_TAGS = {
    message_constants.ERROR: 'danger',
}
```

Now `messages.error(request, '…')` produces `message.tags == 'danger'` and `message.level_tag == 'danger'`.

`MESSAGE_TAGS` is **merged** with the defaults — you only list the levels you change. The result is:

```python
{10: 'debug', 20: 'info', 25: 'success', 30: 'warning', 40: 'danger'}
```

### 9.3 Full Bootstrap mapping (with a prefix)

If you'd rather put the whole class in the tag:

```python
MESSAGE_TAGS = {
    message_constants.DEBUG: 'alert-secondary',
    message_constants.INFO: 'alert-info',
    message_constants.SUCCESS: 'alert-success',
    message_constants.WARNING: 'alert-warning',
    message_constants.ERROR: 'alert-danger',
}
```

Then the template is just `<div class="alert {{ message.tags }}">`. Step 24 uses the simpler `ERROR: 'danger'` version with `alert-{{ message.level_tag }}`.

### 9.4 Tags are computed when **displayed**, not when added

`message.level_tag` looks the level up in a table built from `MESSAGE_TAGS` each time you read it. Changing `MESSAGE_TAGS` changes the tags of messages already waiting in cookies. In tests, `@override_settings(MESSAGE_TAGS={...})` works because the messages app listens to the `setting_changed` signal and rebuilds that table — one more reason to keep `django.contrib.messages` in `INSTALLED_APPS`.

---

## Step 10: Custom Levels

You can invent your own levels — they're just integers.

### 10.1 Define them in one place

Create `blog/message_levels.py`:

```python
# Custom message levels. Built-ins: DEBUG=10, INFO=20, SUCCESS=25, WARNING=30, ERROR=40.
NOTICE = 22      # between INFO and SUCCESS
CRITICAL = 50    # above ERROR
```

### 10.2 Give them tags

In `config/settings.py` (use the numbers — importing `blog` modules from settings is fragile):

```python
MESSAGE_TAGS = {
    message_constants.ERROR: 'danger',
    22: 'notice',
    50: 'critical',
}
```

Without a tag, a custom level's `level_tag` is `''` and `tags` contains only the extra tags.

### 10.3 Add messages with them

```python
from django.contrib import messages

from .message_levels import CRITICAL

messages.add_message(request, CRITICAL, 'Your account has been suspended.')
```

Optional — your own shortcut, like the built-in ones, in `blog/message_levels.py`:

```python
from django.contrib import messages


def critical(request, message, extra_tags='', fail_silently=False):
    messages.add_message(request, CRITICAL, message, extra_tags=extra_tags, fail_silently=fail_silently)
```

### 10.4 Things to know

- `DEFAULT_MESSAGE_LEVELS` in templates only contains the five built-ins. Compare custom levels with numbers (`{% if message.level == 50 %}`) or, better, with their tag (`{% if message.level_tag == 'critical' %}`).
- Custom levels respect `MESSAGE_LEVEL` like any other: `NOTICE = 22` is recorded with the default `INFO` (20); a level of `15` would be dropped.
- **Reusable apps** shouldn't define custom levels — the project may not have tags for them, and numbers may collide with the project's own levels.

---

## Step 11: The Message Lifecycle — When Is a Message Deleted?

This is the most important step for understanding the app. It's all in `BaseStorage` (`django/contrib/messages/storage/base.py`).

### 11.1 The storage object's state

On every request `MessageMiddleware` creates a **new** storage object: `request._messages = default_storage(request)`. It has:

| Attribute            | Meaning                                                                                              |
|----------------------|------------------------------------------------------------------------------------------------------|
| `_loaded_messages`   | Messages that came **in** with this request (from the cookie/session). Loaded **lazily** on first use |
| `_queued_messages`   | Messages **added** during this request                                                               |
| `used`               | `True` once the storage has been **iterated**                                                        |
| `added_new`          | `True` once a message was added during this request                                                  |
| `level`              | The minimum level (Step 8)                                                                           |

### 11.2 What each operation does

| Operation                                           | Effect                                                                                                          |
|-----------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| `messages.success(request, …)`                      | Appends to `_queued_messages`, sets `added_new = True`                                                          |
| `len(storage)` / `{% if messages %}`                | Loads `_loaded_messages`, returns `len(loaded) + len(queued)`. Does **not** set `used`                           |
| `msg in storage`                                    | Checks both lists. Does **not** set `used`                                                                      |
| `for m in storage` / `{% for … in messages %}` / `list(storage)` | Sets `used = True`, moves queued messages into `_loaded_messages`, yields them all                  |
| Response passes through `MessageMiddleware`         | Calls `storage.update(response)` — see 11.3                                                                     |

### 11.3 What happens at the end of the request — `update()`

```python
def update(self, response):
    self._prepare_messages(self._queued_messages)          # turn lazy strings into str
    if self.used:
        return self._store(self._queued_messages, response)  # keep only messages added AFTER the last iteration
    elif self.added_new:
        messages = self._loaded_messages + self._queued_messages
        return self._store(messages, response)              # keep old (unread) + new
    # else: nothing iterated, nothing added → do nothing, the cookie/session stays as it is
```

In plain words:

| During this request…                                          | At the end…                                                        |
|---------------------------------------------------------------|--------------------------------------------------------------------|
| Nothing happened with messages                                | Nothing changes — waiting messages keep waiting                     |
| A message was added, nobody iterated                          | Old unread messages + the new one are stored                        |
| Messages were iterated (displayed)                            | All of them are deleted                                             |
| Messages were iterated, then another one was added            | Only the one added **after** iteration is stored (for the next page) |

### 11.4 Prove it with the demo

Add to `blog/views.py`:

```python
from django.http import HttpResponse


@login_not_required
def messages_peek(request):
    """Counts messages without displaying them — they stay for the next page."""
    storage = messages.get_messages(request)
    return HttpResponse(f'{len(storage)} message(s) waiting. Nothing was consumed.')


@login_not_required
def messages_consume(request):
    """Iterates messages — they are deleted after this request."""
    texts = [str(m) for m in messages.get_messages(request)]
    return HttpResponse(f'Consumed: {texts}')
```

```python
path('messages-demo/peek/', views.messages_peek, name='messages_peek'),
path('messages-demo/consume/', views.messages_consume, name='messages_consume'),
```

Then, with the five-message version of `messages_add`:

1. Change `messages_add` to `return redirect('blog:messages_peek')` and click the button → `4 message(s) waiting.`
2. Reload → still `4 message(s) waiting.` (`len()` doesn't consume).
3. Visit `/messages-demo/consume/` → `Consumed: ['An info message.', …]`.
4. Visit `/messages-demo/peek/` → `0 message(s) waiting.`

Change `messages_add` back to `redirect('blog:messages_demo')` afterwards.

### 11.5 Keep messages after iterating — `storage.used = False`

Sometimes you want to **look** at messages in Python without consuming them:

```python
storage = messages.get_messages(request)
has_errors = any(m.level >= messages.ERROR for m in storage)
storage.used = False          # "I didn't display them" → they're stored again for the next page
```

### 11.6 Messages don't expire by time

There's no "valid for 5 minutes". A message waits until a page **iterates** it, or until its storage disappears:

- **Cookie** storage: the `messages` cookie has no `Max-Age`, so it's a **browser-session cookie** — closing the browser normally deletes it.
- **Session** storage: it lives as long as the session (`SESSION_COOKIE_AGE`, 2 weeks by default) or until the session is flushed.

---

## Step 12: Storage Backends — Overview

The `MESSAGE_STORAGE` setting picks the class that stores messages between requests.

| Class (`MESSAGE_STORAGE` value)                                   | Where messages live                                | Needs sessions? | Size limit     |
|-------------------------------------------------------------------|----------------------------------------------------|-----------------|----------------|
| `'django.contrib.messages.storage.fallback.FallbackStorage'` **(default)** | Cookie first, overflow in the session       | Yes             | None in practice |
| `'django.contrib.messages.storage.cookie.CookieStorage'`          | A signed cookie named `messages`                   | No              | ~2 KB          |
| `'django.contrib.messages.storage.session.SessionStorage'`        | The session, key `_messages`                       | Yes             | The session backend's |

You'll almost never change it. Steps 13–15 show what each one does so you can debug them; Step 16 helps you choose.

All three share the same interface (from `BaseStorage`). Your views and templates don't change when you switch.

---

## Step 13: `CookieStorage` in Detail

### 13.1 The cookie

| Property            | Value                                                                                      |
|---------------------|--------------------------------------------------------------------------------------------|
| Name                | `messages` (`CookieStorage.cookie_name`)                                                   |
| Value               | Signed, base64-encoded JSON (optionally zlib-compressed)                                   |
| Max size            | 2048 bytes (`CookieStorage.max_cookie_size`) — half of the 4 KB header limit of some servers |
| Expiry              | None → deleted when the browser closes                                                     |
| `Domain`            | `SESSION_COOKIE_DOMAIN`                                                                    |
| `Secure`            | `SESSION_COOKIE_SECURE`                                                                    |
| `HttpOnly`          | `SESSION_COOKIE_HTTPONLY` (default `True`)                                                 |
| `SameSite`          | `SESSION_COOKIE_SAMESITE` (default `'Lax'`)                                                |
| `Path`              | `/`                                                                                        |

There are **no** `MESSAGE_COOKIE_*` settings — the cookie reuses your **session cookie** settings. If you made your session cookie secure in production, the messages cookie is secure too.

### 13.2 Anatomy of the value

```
W1siX19qc29uX21lc3NhZ2UiLDAsMjUsIlNhdmVkISIsIiJdXQ : 1xKscL : 9JSYD8LWzakl-iQ0PclCIqqwKj0Jt7wnc5rFv0UJNqI
└────────────── payload (base64 JSON) ───────────┘   └time┘   └───────────── HMAC signature ─────────────┘
```

Decode the payload yourself (`python`, not the Django shell):

```python
>>> import base64
>>> base64.urlsafe_b64decode('W1siX19qc29uX21lc3NhZ2UiLDAsMjUsIlNhdmVkISIsIiJdXQ' + '==')
b'[["__json_message",0,25,"Saved!",""]]'
```

Each message is a JSON list:

| Position | Example            | Meaning                                                     |
|----------|--------------------|-------------------------------------------------------------|
| 0        | `"__json_message"` | Marker: "this list is a `Message`"                          |
| 1        | `0`                | `1` if the text was marked safe (`mark_safe`), else `0`     |
| 2        | `25`               | Level                                                       |
| 3        | `"Saved!"`         | Text                                                        |
| 4        | `""`               | Extra tags                                                  |

When compression makes the value shorter, Django compresses it and the payload starts with a **dot** (`.eJyLjo4…`).

### 13.3 Signed — not encrypted

- **Signed**: the signature is an HMAC made with your `SECRET_KEY` (salt `'django.contrib.messages'`). If anyone changes the value, the signature doesn't match, Django ignores the cookie and deletes it. A user **can't forge** a message like "Payment received".
- **Not encrypted**: as you just saw, **anyone can read** the text. Never put secrets in a message (passwords, tokens, reset links, personal data of other people).
- If you **rotate `SECRET_KEY`**, waiting messages become invalid and are silently dropped (or add the old key to `SECRET_KEY_FALLBACKS` while rotating).

### 13.4 Decode it with Django

```bash
python manage.py shell
```

```python
>>> from django.contrib.messages.storage.cookie import CookieStorage
>>> from django.test import RequestFactory
>>> request = RequestFactory().get('/')
>>> request.COOKIES['messages'] = 'W1siX19qc29uX21lc3NhZ2UiLDAsMjUsIlNhdmVkISIsIiJdXQ:1xKscL:9JSYD8LWzakl-iQ0PclCIqqwKj0Jt7wnc5rFv0UJNqI'
>>> list(CookieStorage(request))
[Message(level=25, message='Saved!')]
```

(Paste **your** cookie value from DevTools — the example only works with the `SECRET_KEY` it was signed with.)

### 13.5 What happens when messages don't fit

`CookieStorage` alone, with too many messages:

- In the **default** `CookieStorage._store(remove_oldest=True)`, the **oldest** messages are removed until the rest fits.
- The removed messages are returned as "unstored". With `DEBUG = True`, `MessageMiddleware` then raises `ValueError: Not all temporary messages could be stored.` In production they are silently lost.
- When some messages had to be left out, a marker `"__messagesnotfinished__"` is added at the end of the cookie list. `FallbackStorage` uses that marker to know "there's more in the session" (Step 15).

### 13.6 Use cookie-only storage

```python
MESSAGE_STORAGE = 'django.contrib.messages.storage.cookie.CookieStorage'
```

Good for: sites **without sessions** at all (no `SessionMiddleware`). Messages must stay short and few.

---

## Step 14: `SessionStorage` in Detail

### 14.1 Where the data goes

```python
MESSAGE_STORAGE = 'django.contrib.messages.storage.session.SessionStorage'
```

Messages are saved in `request.session['_messages']` as a **JSON string** in the same format as the cookie (without the signature — the session is already on the server):

```python
>>> request.session['_messages']
'[["__json_message", 0, 25, "Saved!", ""]]'
```

When there are no more messages, the key is removed (`session.pop('_messages', None)`).

### 14.2 See it in the database

With the database session backend, add a message (don't display it), copy your `sessionid` cookie value from DevTools, then:

```bash
python manage.py shell
```

```python
>>> from django.contrib.sessions.models import Session
>>> Session.objects.get(session_key='PASTE-YOUR-SESSIONID-HERE').get_decoded()
{'_auth_user_id': '1', ..., '_messages': '[["__json_message", 0, 25, "Saved!", ""]]'}
```

### 14.3 Things to know

- **Requires sessions** — `SessionMiddleware` above `MessageMiddleware` (Step 1.2).
- **Adding a message writes the session** (a database `UPDATE` with the db backend) on that response and again when it's displayed (to remove it). With `FallbackStorage`, small messages avoid these writes by using the cookie.
- **Every new visitor gets a session**: adding a message for an anonymous visitor creates a session row and sets a `sessionid` cookie.
- **No size limit** beyond the session backend's — except with `signed_cookies` sessions (~4 KB in total).
- **Logout**: `logout()` calls `session.flush()`. Messages are written to the session at the **end** of the request, so messages added before and after `logout()` in the same request **both survive** — they land in the new, empty session. Messages added in an **earlier** request and not yet displayed are lost with the old session (with cookie storage, they'd survive).

---

## Step 15: `FallbackStorage` in Detail (the Default)

```python
class FallbackStorage(BaseStorage):
    storage_classes = (CookieStorage, SessionStorage)
```

### 15.1 Storing

1. Try to put **all** messages in the cookie (`remove_oldest=False` — the **first** messages are kept in the cookie).
2. Messages that don't fit are passed to `SessionStorage`, and the cookie ends with the `"__messagesnotfinished__"` marker.
3. A storage that held messages before but has none now is **cleared** (the cookie deleted / `_messages` removed).

### 15.2 Reading

1. Read the cookie. No cookie → stop (the session isn't even touched — **no session query** for the common case).
2. If the cookie ends with the "not finished" marker → also read `_messages` from the session and append.

### 15.3 See it overflow

Add a temporary view:

```python
import secrets


@login_not_required
def messages_flood(request):
    for i in range(40):
        messages.info(request, f'Message {i}: {secrets.token_hex(40)}')   # random text doesn't compress
    return redirect('blog:messages_demo')
```

```python
path('messages-demo/flood/', views.messages_flood, name='messages_flood'),
```

Visit `/messages-demo/flood/`:

- In DevTools, the `messages` cookie is just under 2 KB (about the first 28 messages).
- The remaining ~12 are in the session's `_messages` key (check with Step 14.2).
- The page shows all 40, **in order**.

Remove the view afterwards.

### 15.4 Why it's the default

| Situation                         | What happens                                                               |
|-----------------------------------|----------------------------------------------------------------------------|
| Normal (a message or two)         | Cookie only — **no extra database writes**, works for anonymous visitors without creating a session |
| Lots of / long messages           | The overflow goes to the session — **nothing is lost**                     |

---

## Step 16: Choose a Storage Backend

| Your situation                                                                 | Choose                                                      |
|--------------------------------------------------------------------------------|-------------------------------------------------------------|
| Normal project                                                                 | **`FallbackStorage`** (default — change nothing)            |
| No sessions in the project                                                     | `CookieStorage`                                             |
| Messages must not be readable in the browser                                   | `SessionStorage` (data stays on the server)                 |
| Messages must not appear in cookies at all (strict header size, privacy policy) | `SessionStorage`                                            |
| Messages must survive across devices / be stored per user                      | Not this app — use a database model (Step 32)               |
| Something special (cache, logging, Redis…)                                     | A custom storage (Step 31)                                  |

Change it in `config/settings.py`:

```python
MESSAGE_STORAGE = 'django.contrib.messages.storage.session.SessionStorage'
```

No migration and no code changes needed. Messages waiting in the old storage at deploy time are simply not read.

---

## Step 17: Messages in Function-Based Views — the Blog

Now use messages properly in the real blog views from the auth guide (Step 28).

### 17.1 The pattern

```python
if request.method == 'POST':
    form = SomeForm(request.POST)
    if form.is_valid():
        form.save()
        messages.success(request, 'Saved.')        # ① add the message
        return redirect('somewhere')               # ② redirect (PRG)
else:
    form = SomeForm()
return render(request, 'template.html', {'form': form})   # ③ invalid form → re-render, errors on the form
```

### 17.2 Publish a post

`post_publish` already does it:

```python
@login_required
@permission_required('blog.publish_post', raise_exception=True)
@require_POST
def post_publish(request, pk):
    post = get_object_or_404(Post, pk=pk)
    if post.status == 'published':
        messages.info(request, f'"{post.title}" was already published.')
        return redirect(post)
    post.status = 'published'
    post.save(update_fields=['status', 'updated_at'])
    messages.success(request, f'"{post.title}" is now published.')
    return redirect(post)
```

- Use the **right level**: nothing changed → `info`; it worked → `success`.
- Mention **what** happened to **which** object. "Saved." is fine for one form; "'My first post' is now published." is better on a list page.

### 17.3 Unpublish with a warning

Add to `blog/views.py`:

```python
@login_required
@permission_required('blog.publish_post', raise_exception=True)
@require_POST
def post_unpublish(request, pk):
    post = get_object_or_404(Post, pk=pk)
    post.status = 'draft'
    post.save(update_fields=['status', 'updated_at'])
    messages.warning(request, f'"{post.title}" is now a draft and hidden from readers.')
    return redirect(post)
```

```python
path('post/<int:pk>/unpublish/', views.post_unpublish, name='post_unpublish'),
```

### 17.4 Don't shadow the `messages` module

```python
# ❌ WRONG
def inbox(request):
    messages = Message.objects.filter(to=request.user)      # your own model called Message
    messages.success(request, 'Loaded.')                     # AttributeError: 'QuerySet' object has no attribute 'success'
```

```python
# ✅ RIGHT
def inbox(request):
    user_messages = Message.objects.filter(to=request.user)
    messages.success(request, 'Loaded.')
```

The same goes for templates: if you pass `{'messages': ...}` in a view's context, it **replaces** the context processor's `messages`, and the flash messages disappear from that page. Call your variable `inbox_messages`, `chat_messages`, etc.

If you really need another name for the module:

```python
from django.contrib import messages as flash
flash.success(request, 'Saved.')
```

### 17.5 Message before the redirect, not after

```python
return redirect('blog:post_list')
messages.success(request, 'Deleted.')       # ❌ never runs
```

Obvious when written like this, but easy to miss inside `try/except` blocks or helper functions. Add the message, **then** return.

### 17.6 Messages after `bulk` operations

```python
@login_required
@permission_required('blog.publish_post', raise_exception=True)
@require_POST
def publish_all_my_drafts(request):
    count = Post.objects.filter(author=request.user, status='draft').update(status='published')
    if count == 0:
        messages.info(request, 'You have no drafts to publish.')
    elif count == 1:
        messages.success(request, '1 draft was published.')
    else:
        messages.success(request, f'{count} drafts were published.')
    return redirect('blog:post_list')
```

Step 26 shows how to do the singular/plural properly with `ngettext` when the site is translated.

### 17.7 Use messages with `HttpResponseRedirect` and `reverse`

Any redirect works — `redirect()`, `HttpResponseRedirect(reverse(...))`, `redirect(post)` (uses `get_absolute_url()`), or a `next` URL:

```python
from django.utils.http import url_has_allowed_host_and_scheme

next_url = request.POST.get('next', '')
if not url_has_allowed_host_and_scheme(next_url, allowed_hosts={request.get_host()}):
    next_url = reverse('blog:post_list')
messages.success(request, 'Added to your reading list.')
return redirect(next_url)
```

The message travels via a cookie, so it works for any redirect target **on the same site** (same cookie domain). It does **not** appear on another domain.

---

## Step 18: Messages in Class-Based Views — `SuccessMessageMixin`

### 18.1 The mixin

`django.contrib.messages.views.SuccessMessageMixin` is tiny:

```python
class SuccessMessageMixin:
    success_message = ''

    def form_valid(self, form):
        response = super().form_valid(form)               # saves the object and builds the redirect
        success_message = self.get_success_message(form.cleaned_data)
        if success_message:
            messages.success(self.request, success_message)
        return response

    def get_success_message(self, cleaned_data):
        return self.success_message % cleaned_data
```

It works with any view that has `form_valid()`: `FormView`, `CreateView`, `UpdateView`, and `DeleteView` (Django 4.0+).

### 18.2 `CreateView` — replace the manual message

In `blog/views.py`, change `PostCreateView`:

```python
from django.contrib.messages.views import SuccessMessageMixin


class PostCreateView(LoginRequiredMixin, PermissionRequiredMixin, SuccessMessageMixin, CreateView):
    permission_required = 'blog.add_post'
    model = Post
    fields = ['title', 'slug', 'content']
    template_name = 'blog/post_form.html'
    success_message = '"%(title)s" was created as a draft.'

    def form_valid(self, form):
        form.instance.author = self.request.user
        return super().form_valid(form)
```

- **`%(title)s`** is filled in from `form.cleaned_data['title']` — the mixin uses Python's old `%` formatting with the cleaned data dict.
- You can only use **form fields** (`title`, `slug`, `content`) this way — not `pk` or `author`, which aren't in the form. See 18.4 for those.
- To write a literal `%` in the message, double it: `'Upload 100%% complete'`.
- If a placeholder isn't in `cleaned_data`, you get `KeyError` **after** the object was saved — test your success messages.

### 18.3 Mixin order (MRO)

Put `SuccessMessageMixin` **before** the generic view (`CreateView`, `UpdateView`…). Access mixins (`LoginRequiredMixin`, `PermissionRequiredMixin`) stay **first** because they must run before anything else:

```python
class PostCreateView(LoginRequiredMixin, PermissionRequiredMixin, SuccessMessageMixin, CreateView):
```

If you put it after `CreateView`, `CreateView.form_valid()` is found first and the mixin's `form_valid()` never runs — no message, no error.

Your own `form_valid()` must call `super().form_valid(form)` — that's what reaches the mixin.

### 18.4 Use object data — override `get_success_message()`

After `super().form_valid()`, `self.object` is the saved object, so you can use anything on it:

```python
class PostUpdateView(LoginRequiredMixin, OwnerOrPermissionMixin, SuccessMessageMixin, UpdateView):
    owner_bypass_permission = 'blog.change_post'
    model = Post
    fields = ['title', 'slug', 'content']
    template_name = 'blog/post_form.html'
    success_message = '"%(title)s" was updated (status: %(status)s).'

    def get_success_message(self, cleaned_data):
        return self.success_message % dict(
            cleaned_data,
            status=self.object.get_status_display(),
        )
```

Or write the text directly, ignoring `success_message`:

```python
    def get_success_message(self, cleaned_data):
        return f'"{self.object.title}" was updated by {self.request.user.get_username()}.'
```

Return an **empty string** to skip the message conditionally — for example when the user saved the form without changing anything:

```python
    def form_valid(self, form):
        self.nothing_changed = not form.has_changed()     # remember it for get_success_message()
        return super().form_valid(form)

    def get_success_message(self, cleaned_data):
        if self.nothing_changed:
            return ''                                     # empty → the mixin adds no message
        return super().get_success_message(cleaned_data)
```

### 18.5 `DeleteView`

Since Django 4.0, `DeleteView` processes a (empty) form, so `SuccessMessageMixin` works. But the form has **no fields**, so `cleaned_data` is `{}` — use `self.object`:

```python
class PostDeleteView(LoginRequiredMixin, OwnerOrPermissionMixin, SuccessMessageMixin, DeleteView):
    owner_bypass_permission = 'blog.delete_post'
    model = Post
    template_name = 'blog/post_confirm_delete.html'
    success_url = reverse_lazy('blog:post_list')
    success_message = 'The post was deleted.'

    def get_success_message(self, cleaned_data):
        return f'"{self.object.title}" was deleted.'
```

`self.object` is still available (in memory) after the row has been deleted.

> Before Django 4.0, `DeleteView` ignored `SuccessMessageMixin`; people overrode `delete()` instead. If you see that in old tutorials, it's no longer needed.

### 18.6 `FormView` (no model)

```python
from django.views.generic import FormView

from .forms import ContactForm


class ContactView(SuccessMessageMixin, FormView):
    template_name = 'blog/contact.html'
    form_class = ContactForm
    success_url = reverse_lazy('blog:post_list')
    success_message = 'Thanks %(name)s, we got your message.'

    def form_valid(self, form):
        form.send_email()          # whatever your form does
        return super().form_valid(form)
```

### 18.7 Messages in other class-based view methods

`SuccessMessageMixin` only covers `form_valid()`. Anywhere else, call the functions with **`self.request`**:

```python
class OwnerOrPermissionMixin(UserPassesTestMixin):
    owner_bypass_permission = None

    def test_func(self):
        post = self.get_object()
        user = self.request.user
        return post.author == user or user.has_perm(self.owner_bypass_permission)

    def handle_no_permission(self):
        if self.request.user.is_authenticated:
            messages.error(self.request, "You can only edit your own posts.")
            return redirect('blog:post_list')
        return super().handle_no_permission()      # anonymous → login page
```

- `messages.error(self, …)` → `TypeError: add_message() argument must be an HttpRequest object, not 'PostUpdateView'.`
- Overriding `handle_no_permission()` this way replaces the 403 page with a friendly message + redirect.

---

## Step 19: Error and Warning Messages — When (and When Not)

### 19.1 Invalid forms — use form errors, optionally a summary message

Django already shows field errors next to the fields. A message is a **summary on top**, not a replacement:

```python
class PostCreateView(LoginRequiredMixin, PermissionRequiredMixin, SuccessMessageMixin, CreateView):
    ...

    def form_invalid(self, form):
        messages.error(self.request, 'Please correct the errors below.')
        return super().form_invalid(form)
```

Because `form_invalid()` **re-renders** the page (no redirect), the message is displayed in the **same** request (Step 3.7).

In a function view:

```python
if form.is_valid():
    ...
else:
    messages.error(request, 'Please correct the errors below.')
return render(request, 'blog/post_form.html', {'form': form})
```

> Don't put the actual validation errors into messages (`messages.error(request, form.errors)`). They'd be shown far from the fields, and they're lost if the user navigates away.

### 19.2 Not found / not allowed — message + redirect, or a real error page?

| Situation                                                   | Better choice                                                 |
|-------------------------------------------------------------|---------------------------------------------------------------|
| Someone follows a broken or old link                        | `404` (`get_object_or_404`) — search engines need the status  |
| A logged-in user clicks a button they shouldn't see         | `messages.error` + redirect (friendly) — or `403`             |
| An API / AJAX request                                       | A proper status code + JSON (Step 27)                         |
| A draft the user isn't allowed to see                       | An `info` message ("That post isn't published yet.") + redirect |

Update `post_detail` from the auth guide:

```python
def post_detail(request, pk):
    post = get_object_or_404(Post, pk=pk)
    is_owner = request.user.is_authenticated and post.author == request.user
    if post.status == 'draft' and not (is_owner or request.user.has_perm('blog.change_post')):
        messages.info(request, "That post isn't published yet.")
        return redirect('blog:post_list')
    return render(request, 'blog/post_detail.html', {'post': post, 'is_owner': is_owner})
```

### 19.3 Catch exceptions — tell the user, log the details

```python
import logging

logger = logging.getLogger(__name__)


@login_required
@require_POST
def post_share_by_email(request, pk):
    post = get_object_or_404(Post, pk=pk, status='published')
    try:
        send_share_email(post, to=request.POST['email'])
    except Exception:
        logger.exception('Sharing post %s failed', post.pk)               # full traceback for you
        messages.error(request, "Sorry, we couldn't send the email. Please try again later.")   # short text for the user
    else:
        messages.success(request, 'Email sent.')
    return redirect(post)
```

Never show `str(exception)` to users — it can leak file paths, SQL, or internal hostnames.

---

## Step 20: Messages in the Admin

The admin uses this same app for its green "The post “Hello” was changed successfully." bars.

### 20.1 `message_user()`

Inside a `ModelAdmin`, use `self.message_user()` instead of `messages.success()`:

```python
ModelAdmin.message_user(request, message, level=messages.INFO, extra_tags='', fail_silently=False)
```

Note the **argument order** differs from `add_message()` (message before level), and `level` can be a **string**: `'success'`, `'warning'`, `'error'`, `'info'`, `'debug'` (any case). A bad string raises `ValueError: Bad message level string: …`.

### 20.2 Improve the admin actions from the admin guide

Update `blog/admin.py`:

```python
from django.contrib import admin, messages
from django.utils.translation import ngettext

from .models import Post


@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    # ... list_display, list_filter etc. from the admin guide ...
    actions = ['make_published', 'make_draft']

    @admin.action(description='Mark selected posts as published')
    def make_published(self, request, queryset):
        already = queryset.filter(status='published').count()
        updated = queryset.exclude(status='published').update(status='published')
        if updated:
            self.message_user(
                request,
                ngettext('%d post was published.', '%d posts were published.', updated) % updated,
                messages.SUCCESS,
            )
        if already:
            self.message_user(
                request,
                ngettext('%d post was already published.', '%d posts were already published.', already) % already,
                messages.WARNING,
            )

    @admin.action(description='Mark selected posts as draft')
    def make_draft(self, request, queryset):
        updated = queryset.update(status='draft')
        self.message_user(request, f'{updated} post(s) marked as draft.', 'warning')   # string level works too
```

Select some posts, run the action, and you'll see a green and/or yellow bar.

### 20.3 Messages in other admin hooks

```python
    def save_model(self, request, obj, form, change):
        super().save_model(request, obj, form, change)
        if not obj.content.strip():
            self.message_user(request, f'"{obj.title}" has no content yet.', messages.WARNING)

    def delete_model(self, request, obj):
        if obj.status == 'published':
            self.message_user(request, f'You deleted a published post: "{obj.title}".', messages.WARNING)
        super().delete_model(request, obj)
```

Your messages appear **next to** the admin's own "…was changed successfully." message.

### 20.4 Admin styling

The admin template (`admin/base.html`) renders:

```html
<ul class="messagelist">{% for message in messages %}
  <li{% if message.tags %} class="{{ message.tags }}"{% endif %}>{{ message|capfirst }}</li>
{% endfor %}</ul>
```

and has CSS (in `admin/css/base.css`) for the classes `debug`, `info`, `success`, `warning` and `error`. If you renamed a tag with `MESSAGE_TAGS` (e.g. `ERROR: 'danger'` from Step 9), **admin error messages lose their red style and icon** — the admin has no CSS for `danger`.

Fix it in `templates/admin/base_site.html` from the admin guide (Step 16) by adding a rule to the `extrastyle` block:

```html
{% block extrastyle %}
    {{ block.super }}
    <style>
        #header { background: #1e3a5f; }
        ul.messagelist li.danger {            /* MESSAGE_TAGS renamed 'error' to 'danger' */
            background-color: var(--message-error-bg);
            background-image: var(--message-error-icon);
        }
    </style>
{% endblock %}
```

The CSS variables come from the admin's own stylesheet, so this also works in the admin's dark mode.

### 20.5 Admin actions that return a response

If an action **returns** an `HttpResponse` (e.g. a CSV download), the page doesn't reload, so a message added in the action is shown on the **next** admin page instead. Don't add a message in download actions, or the user will see it later out of context.

---

## Step 21: Messages in Signals, Middleware, Decorators and Helpers

Anything that has the **request** can add messages.

### 21.1 Signals — welcome message on login

The auth guide (Step 30) has this in `accounts/signals.py`. Add **`fail_silently=True`**:

```python
@receiver(user_logged_in)
def on_login(sender, request, user, **kwargs):
    messages.success(request, f'Welcome back, {user.get_username()}!', fail_silently=True)
```

Why: `self.client.login()` and `self.client.force_login()` in **tests** call `login()` with a bare `HttpRequest` that never went through `MessageMiddleware`. Without `fail_silently=True`, **every test that logs a user in crashes** with `MessageFailure: You cannot add messages without installing django.contrib.messages.middleware.MessageMiddleware`. The same applies to any other code that logs users in outside a normal request. With `fail_silently=True`, the welcome message simply isn't added there, and real browser logins still get it.

- `user_logged_in` gets `request` — so messages work.
- `post_save` and other **model** signals do **not** get a request. Don't try to add messages from them (no global "current request" in Django). Add the message in the view that saves the model instead.
- `user_logged_in` also fires for `client.login()` / `force_login()` in tests and for logins via the admin. The admin login page then shows "Welcome back".

### 21.2 Signals — goodbye message on logout

```python
@receiver(user_logged_out)
def on_logout(sender, request, user, **kwargs):
    if user is not None:
        logger.info('Logout: %s', user.get_username())
    messages.info(request, 'You have been logged out. See you soon!', fail_silently=True)
```

(`self.client.logout()` in tests also sends this signal with a bare request — hence `fail_silently=True` again.)

This **works** with both `FallbackStorage` and `SessionStorage`: the signal is sent before the session is flushed, but the message is only kept **in memory** (on `request._messages`) and written to the cookie/session at the end of the request — after the flush. Add `{% include 'blog/includes/messages.html' %}` to the page `LogoutView` redirects to (or to `registration/logged_out.html` if it renders that template).

### 21.3 Middleware

The sessions guide's `IdleTimeoutMiddleware` adds "You were logged out after a period of inactivity." Rules for middleware:

- It must be **below** `MessageMiddleware` in `MIDDLEWARE`.
- Add the message **before** calling `get_response()` if it should appear on the page being requested, or **after** it if the page was already rendered (then it shows on the next page).

### 21.4 Decorators

A decorator that blocks users and tells them why:

```python
# blog/decorators.py
from functools import wraps

from django.contrib import messages
from django.shortcuts import redirect


def verified_email_required(view_func):
    @wraps(view_func)
    def wrapper(request, *args, **kwargs):
        if not getattr(request.user, 'email_verified', False):
            messages.warning(request, 'Please verify your email address first.')
            return redirect('accounts:profile')
        return view_func(request, *args, **kwargs)
    return wrapper
```

### 21.5 Helper functions and services

Pass the request, or — better — keep business logic message-free and let the view decide:

```python
# ✅ the service returns a result, the view talks to the user
def publish_post(post):
    if post.status == 'published':
        return False
    post.status = 'published'
    post.save(update_fields=['status', 'updated_at'])
    return True


def post_publish(request, pk):
    post = get_object_or_404(Post, pk=pk)
    if publish_post(post):
        messages.success(request, f'"{post.title}" is now published.')
    else:
        messages.info(request, f'"{post.title}" was already published.')
    return redirect(post)
```

That way `publish_post()` also works in management commands, Celery tasks and tests, where there's no request.

### 21.6 Management commands, Celery tasks, cron jobs

There's **no request** and no browser → no messages. Use `self.stdout.write()` in commands, logging in tasks, and a `Notification` model (Step 32) to tell users about background work.

---

## Step 22: Messages and Login / Logout / Sessions

| Event                                           | What happens to messages (default `FallbackStorage`)                                             |
|-------------------------------------------------|--------------------------------------------------------------------------------------------------|
| `login()` (calls `cycle_key()`)                 | Session data is kept, cookie untouched → messages **survive**                                    |
| `logout()` (calls `flush()`)                    | Messages in the **cookie** survive; messages added in this request survive; old overflow in the session is lost |
| Session expires                                 | Only overflow messages (in the session) are lost                                                 |
| Browser closed                                  | The `messages` cookie is deleted (browser-session cookie)                                        |
| `request.session.flush()` in your own code      | Same as logout                                                                                   |
| "Log out other devices" (sessions guide)        | Other browsers' messages aren't affected — they're in those browsers' cookies                    |

> **Correction to the auth guide (Step 30):** its note says messages added during logout are lost. They aren't. With Django's built-in storages, messages added in the same request as `logout()` (including in `user_logged_out` receivers) are saved *after* the flush and do appear on the next page. Only messages from **earlier** requests that were waiting in the session are lost.

### 22.1 Login with `next` — where does the message appear?

`LoginView` redirects to `next`. The `user_logged_in` signal's "Welcome back" message is shown on **that** page, as long as it includes the messages template.

### 22.2 Messages on `LoginRequiredMixin` redirects

Tell users **why** they were sent to the login page:

```python
from django.contrib import messages
from django.contrib.auth.mixins import LoginRequiredMixin


class LoginRequiredMessageMixin(LoginRequiredMixin):
    login_message = 'Please log in to continue.'

    def handle_no_permission(self):
        if not self.request.user.is_authenticated:
            messages.info(self.request, self.login_message)
        return super().handle_no_permission()
```

```python
class PostCreateView(LoginRequiredMessageMixin, PermissionRequiredMixin, SuccessMessageMixin, CreateView):
    login_message = 'Log in to write a post.'
    ...
```

Make sure `registration/login.html` includes the messages template.

---

## Step 23: Style the Messages (Plain CSS + a Little JavaScript)

### 23.1 CSS for the default tags

Add to `blog/static/blog/style.css` (from the setup guide):

```css
/* ---------- messages ---------- */
.messages {
    list-style: none;
    padding: 0;
    margin: 1rem 0;
}
.messages li {
    position: relative;
    padding: 0.75rem 2.5rem 0.75rem 1rem;
    margin-bottom: 0.5rem;
    border-radius: 6px;
    border-left: 5px solid;
    background: #f5f5f5;
    border-color: #999;
}
.messages .debug   { background: #f3f3f3; border-color: #777;    color: #333; font-family: monospace; }
.messages .info    { background: #e7f1fb; border-color: #2b7bd6; color: #123c69; }
.messages .success { background: #e6f6ea; border-color: #2e9e4f; color: #134d25; }
.messages .warning { background: #fff6dd; border-color: #e0a800; color: #5c4400; }
.messages .error,
.messages .danger  { background: #fde8e8; border-color: #d33a3a; color: #6b1212; }
.messages .critical{ background: #6b1212; border-color: #3d0000; color: #fff; font-weight: bold; }

.messages .close {
    position: absolute;
    top: 0.4rem;
    right: 0.6rem;
    border: 0;
    background: transparent;
    font-size: 1.3rem;
    line-height: 1;
    cursor: pointer;
    color: inherit;
}
.messages li.hiding {
    opacity: 0;
    transition: opacity 0.4s ease;
}
```

Make sure `base.html` loads it (setup guide): `{% load static %}` at the top and `<link rel="stylesheet" href="{% static 'blog/style.css' %}">` in `<head>`.

### 23.2 Icons per level

Update `blog/templates/blog/includes/messages.html`:

```html
{% if messages %}
    <ul class="messages" role="status" aria-live="polite">
        {% for message in messages %}
            <li{% if message.tags %} class="{{ message.tags }}"{% endif %}>
                {% if message.level == DEFAULT_MESSAGE_LEVELS.SUCCESS %}<span aria-hidden="true">✅</span>
                {% elif message.level == DEFAULT_MESSAGE_LEVELS.WARNING %}<span aria-hidden="true">⚠️</span>
                {% elif message.level >= DEFAULT_MESSAGE_LEVELS.ERROR %}<span aria-hidden="true">⛔</span>
                {% else %}<span aria-hidden="true">ℹ️</span>{% endif %}
                {{ message }}
                <button type="button" class="close" aria-label="Dismiss message">&times;</button>
            </li>
        {% endfor %}
    </ul>
{% endif %}
```

`aria-hidden="true"` stops screen readers from reading "check mark button" before every message.

### 23.3 Close button (JavaScript)

Create `blog/static/blog/messages.js`:

```javascript
// Close button and auto-hide for Django messages.
document.addEventListener('DOMContentLoaded', () => {
    const hide = (item) => {
        item.classList.add('hiding');
        item.addEventListener('transitionend', () => item.remove(), { once: true });
    };

    document.querySelectorAll('.messages .close').forEach((button) => {
        button.addEventListener('click', () => hide(button.closest('li')));
    });

    // Auto-hide success and info messages after 5 seconds, unless tagged "sticky".
    document.querySelectorAll('.messages li.success, .messages li.info').forEach((item) => {
        if (!item.classList.contains('sticky')) {
            setTimeout(() => hide(item), 5000);
        }
    });
});
```

Load it at the end of `base.html`, just before `</body>`:

```html
    <script src="{% static 'blog/messages.js' %}"></script>
</body>
```

### 23.4 Which messages to auto-hide

| Level             | Auto-hide?                                                                      |
|-------------------|---------------------------------------------------------------------------------|
| success / info    | OK after ~5 s (but give users time to read; never less than ~4 s)               |
| warning / error   | **No** — the user must see and understand them; let them close manually         |
| anything `sticky` | No (the extra tag from Step 7)                                                  |

```python
messages.info(request, 'Read this carefully: the export will take a few minutes.', extra_tags='sticky')
```

### 23.5 Accessibility

- `role="status"` + `aria-live="polite"` → screen readers announce messages after the current sentence.
- For errors you can use `role="alert"` (announced immediately). Use it on the `<li>` of error messages only:
  ```html
  <li class="{{ message.tags }}"{% if message.level >= DEFAULT_MESSAGE_LEVELS.ERROR %} role="alert"{% endif %}>
  ```
- Don't rely on colour alone — the icons/words ("Error:", "Warning:") help colour-blind users.
- Put the messages **near the top** of the content, before the main heading or right after it, so keyboard and screen-reader users reach them first.
- Make the close button a real `<button>` with an `aria-label` (done above).

---

## Step 24: Bootstrap 5 Integration

### 24.1 Settings

```python
from django.contrib.messages import constants as message_constants

MESSAGE_TAGS = {
    message_constants.DEBUG: 'secondary',
    message_constants.ERROR: 'danger',
}
```

Bootstrap's alert colours are `primary`, `secondary`, `success`, `danger`, `warning`, `info`, `light`, `dark`. `info`, `success` and `warning` already match Django's names.

### 24.2 Alerts

Load Bootstrap in `base.html`'s `<head>`:

```html
<link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css" rel="stylesheet">
```

and before `</body>`:

```html
<script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/js/bootstrap.bundle.min.js"></script>
```

Create `blog/templates/blog/includes/messages_bootstrap.html`:

```html
{% for message in messages %}
    <div class="alert alert-{{ message.level_tag }} alert-dismissible fade show {{ message.extra_tags }}" role="alert">
        {{ message }}
        <button type="button" class="btn-close" data-bs-dismiss="alert" aria-label="Close"></button>
    </div>
{% endfor %}
```

Use `message.level_tag` (just `danger`/`success`…) for the `alert-…` class and `message.extra_tags` separately — `alert-{{ message.tags }}` would produce `alert-dismissible success` for a message with extra tags.

### 24.3 Toasts

```html
<div class="toast-container position-fixed top-0 end-0 p-3">
    {% for message in messages %}
        <div class="toast align-items-center text-bg-{{ message.level_tag }} border-0" role="status" aria-live="polite" aria-atomic="true"
             data-bs-autohide="{% if message.level >= DEFAULT_MESSAGE_LEVELS.WARNING %}false{% else %}true{% endif %}">
            <div class="d-flex">
                <div class="toast-body">{{ message }}</div>
                <button type="button" class="btn-close btn-close-white me-2 m-auto" data-bs-dismiss="toast" aria-label="Close"></button>
            </div>
        </div>
    {% endfor %}
</div>
<script>
    document.querySelectorAll('.toast').forEach((el) => new bootstrap.Toast(el).show());
</script>
```

### 24.4 Tailwind CSS

Tailwind needs full class names in the source, so map levels in the template:

```html
{% for message in messages %}
    <div class="mb-2 rounded border-l-4 p-3
        {% if message.level == DEFAULT_MESSAGE_LEVELS.SUCCESS %}border-green-600 bg-green-50 text-green-900
        {% elif message.level == DEFAULT_MESSAGE_LEVELS.WARNING %}border-yellow-500 bg-yellow-50 text-yellow-900
        {% elif message.level == DEFAULT_MESSAGE_LEVELS.ERROR %}border-red-600 bg-red-50 text-red-900
        {% else %}border-blue-600 bg-blue-50 text-blue-900{% endif %}">
        {{ message }}
    </div>
{% endfor %}
```

---

## Step 25: HTML in Messages — Escaping and `mark_safe`

### 25.1 Messages are escaped by default

```python
messages.success(request, 'Read <a href="/help/">the guide</a>.')
```

shows the literal text `Read <a href="/help/">the guide</a>.` — because `{{ message }}` is auto-escaped. **That's good**: messages often contain user input (post titles!), and escaping prevents XSS.

### 25.2 Safe HTML with `format_html()`

To include a link, build the HTML with `format_html()`. It escapes the **arguments** and marks the result safe:

```python
from django.urls import reverse
from django.utils.html import format_html

messages.success(
    request,
    format_html(
        '"{}" was created. <a href="{}">View it</a> or <a href="{}">write another</a>.',
        post.title,                                  # escaped → safe even if the title contains <script>
        post.get_absolute_url(),
        reverse('blog:post_create'),
    ),
)
```

The **"safe" flag survives the trip** through the cookie/session: `MessageEncoder` stores `1` in position 1 of the JSON list (Step 13.2), and `MessageDecoder` calls `mark_safe()` again when reading. So `{{ message }}` renders the link as HTML.

### 25.3 `mark_safe()` — only for text you wrote completely yourself

```python
from django.utils.safestring import mark_safe

messages.info(request, mark_safe('Read <a href="/help/">the guide</a>.'))       # ✅ fixed text, no user input
messages.info(request, mark_safe(f'Welcome, <b>{user.first_name}</b>!'))        # ❌ XSS — first_name could be <script>…
messages.info(request, format_html('Welcome, <b>{}</b>!', user.first_name))     # ✅
```

### 25.4 Don't use `|safe` in the messages template

```html
{{ message|safe }}      {# ❌ makes EVERY message HTML — any message containing user input becomes an XSS hole #}
{{ message }}           {# ✅ safe-marked messages are already rendered as HTML, all others are escaped #}
```

### 25.5 Plain-text line breaks

`{{ message|linebreaksbr }}` turns `\n` into `<br>` (it escapes first, so it's safe).

---

## Step 26: Translating Messages

### 26.1 `gettext` in views

```python
from django.utils.translation import gettext as _

messages.success(request, _('Profile details updated.'))
```

### 26.2 Placeholders — named, so translators can reorder them

```python
messages.success(request, _('"%(title)s" is now published.') % {'title': post.title})
```

Don't build sentences with f-strings around `_()` — translators need the whole sentence.

### 26.3 Singular / plural — `ngettext`

```python
from django.utils.translation import ngettext

messages.success(
    request,
    ngettext(
        '%(count)d draft was published.',
        '%(count)d drafts were published.',
        count,
    ) % {'count': count},
)
```

Languages have different plural rules (some have 3–6 forms) — `ngettext` handles it.

### 26.4 Lazy translations are fine

`gettext_lazy` strings can be passed directly (e.g. a class attribute):

```python
from django.utils.translation import gettext_lazy as _


class PostCreateView(LoginRequiredMixin, PermissionRequiredMixin, SuccessMessageMixin, CreateView):
    success_message = _('"%(title)s" was created as a draft.')
```

Before storing, `Message._prepare()` turns lazy strings into normal `str` (lazy objects can't be JSON-serialized). This happens at the end of the request, in the **active language of that request** — the language of the user who triggered the action.

---

## Step 27: Messages with AJAX, `fetch()` and HTMX

### 27.1 The problem

If a JavaScript `fetch()` request adds a message and nobody displays it, it waits and pops up on the **next full page load** — out of context. And if an AJAX request **renders a template that iterates messages**, it consumes messages meant for a normal page.

Rule: for AJAX endpoints, either **return** the messages in the response, or don't use messages at all.

### 27.2 Return messages in JSON

```python
from django.http import JsonResponse


def messages_as_json(request):
    """Return (and consume) all waiting messages."""
    return [
        {'level': m.level, 'tag': m.level_tag, 'tags': m.tags, 'message': str(m)}
        for m in messages.get_messages(request)
    ]


@login_required
@require_POST
def reading_list_add_ajax(request, pk):
    post = get_object_or_404(Post, pk=pk, status='published')
    reading_list.add(request, post.pk)                    # from the sessions guide (Step 13)
    messages.success(request, f'"{post.title}" was added to your reading list.')
    return JsonResponse({
        'ok': True,
        'count': reading_list.count(request),
        'messages': messages_as_json(request),
    })
```

```python
path('reading-list/add/<int:pk>/ajax/', views.reading_list_add_ajax, name='reading_list_add_ajax'),
```

Iterating in `messages_as_json()` marks them **used**, so they won't show up again on the next page.

### 27.3 Show them with JavaScript

Add to `blog/static/blog/messages.js`:

```javascript
// Render messages returned by a JSON endpoint into the .messages list.
function showMessages(list) {
    let container = document.querySelector('.messages');
    if (!container) {
        container = document.createElement('ul');
        container.className = 'messages';
        container.setAttribute('role', 'status');
        container.setAttribute('aria-live', 'polite');
        document.querySelector('main, body').prepend(container);
    }
    for (const msg of list) {
        const item = document.createElement('li');
        item.className = msg.tags;
        item.textContent = msg.message;      // textContent → no HTML injection
        container.append(item);
    }
}

async function postJSON(url) {
    const csrftoken = document.querySelector('[name=csrfmiddlewaretoken]').value;
    const response = await fetch(url, {
        method: 'POST',
        headers: { 'X-CSRFToken': csrftoken, 'X-Requested-With': 'fetch' },
    });
    const data = await response.json();
    showMessages(data.messages || []);
    return data;
}
```

Use it on the post detail page (`blog/post_detail.html`):

```html
{% csrf_token %}
<button type="button" onclick="postJSON('{% url 'blog:reading_list_add_ajax' post.pk %}')">Add to reading list</button>
```

`textContent` shows the text as plain text — safe-marked HTML messages are shown with their tags visible. If you need HTML in AJAX messages, send `message|escape`-d HTML from a template partial instead (27.4).

### 27.4 Return a rendered partial instead

```python
from django.template.loader import render_to_string


def messages_html(request):
    return render_to_string('blog/includes/messages.html', request=request)   # request= runs context processors
```

Return it in the JSON (`'messages_html': messages_html(request)`) and insert it with `container.outerHTML = data.messages_html`. Same escaping rules as normal templates.

### 27.5 HTMX — out-of-band swap

With HTMX, render the messages partial into every HTMX response with `hx-swap-oob`. Create `blog/templates/blog/includes/messages_oob.html`:

```html
<div id="messages" hx-swap-oob="true">
    {% include 'blog/includes/messages.html' %}
</div>
```

In `base.html`, wrap the include: `<div id="messages">{% include 'blog/includes/messages.html' %}</div>`.

Then a middleware that appends it to HTMX responses, in `blog/middleware.py`:

```python
from django.contrib.messages import get_messages
from django.template.loader import render_to_string


class HtmxMessagesMiddleware:
    """Append waiting messages to every HTMX partial response (out-of-band swap)."""

    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        response = self.get_response(request)
        is_htmx = request.headers.get('HX-Request') == 'true'
        is_redirect = 300 <= response.status_code < 400 or 'HX-Redirect' in response
        if (
            is_htmx
            and not is_redirect
            and not getattr(response, 'streaming', False)
            and 'text/html' in response.get('Content-Type', '')
            and len(get_messages(request))
        ):
            response.write(render_to_string('blog/includes/messages_oob.html', request=request))
        return response
```

Add `'blog.middleware.HtmxMessagesMiddleware'` to `MIDDLEWARE` **below** `MessageMiddleware`. On the way out it runs **before** `MessageMiddleware`'s `process_response`, so rendering here marks the messages used and they are then deleted properly.

### 27.6 Never consume messages in "background" requests

Polling endpoints, autocomplete, analytics pings, health checks: don't render templates that include the messages partial, and don't call `get_messages()` there. Otherwise a message added by a form POST may be consumed by an invisible request and the user never sees it.

---

## Step 28: Read Messages in Python Code

### 28.1 `get_messages(request)`

```python
storage = messages.get_messages(request)
```

- Returns the **storage** (iterable), or an empty list `[]` if the middleware isn't installed — it **never raises**.
- Iterating consumes (Step 11). `len()` and `in` don't.

### 28.2 Filter by level

```python
storage = messages.get_messages(request)
errors = [m for m in storage if m.level >= messages.ERROR]
others = [m for m in storage if m.level < messages.ERROR]      # second iteration: same list again, still fine
```

Within one request you can iterate as many times as you like — the same messages are yielded each time.

### 28.3 Check "is there an error waiting?" without consuming

```python
storage = messages.get_messages(request)
has_error = any(m.level >= messages.ERROR for m in storage)
storage.used = False      # put them back
```

### 28.4 Template: split errors from the rest

```html
{% for message in messages %}
    {% if message.level >= DEFAULT_MESSAGE_LEVELS.ERROR %}
        <div class="banner {{ message.tags }}">{{ message }}</div>
    {% endif %}
{% endfor %}

{# ...later on the page... #}
{% for message in messages %}
    {% if message.level < DEFAULT_MESSAGE_LEVELS.ERROR %}
        <p class="{{ message.tags }}">{{ message }}</p>
    {% endif %}
{% endfor %}
```

Both loops see all messages (same request); each one prints only its kind.

### 28.5 Remove duplicates

Adding the same message twice (e.g. two signal receivers) shows it twice. De-duplicate when displaying:

```python
# blog/templatetags/message_tags.py
from django import template

register = template.Library()


@register.filter
def unique_messages(storage):
    seen = set()
    result = []
    for message in storage:
        key = (message.level, str(message))
        if key not in seen:
            seen.add(key)
            result.append(message)
    return result
```

(Create `blog/templatetags/__init__.py` too.) Then in the include:

```html
{% load message_tags %}
{% for message in messages|unique_messages %}
```

Better still: avoid adding duplicates — check `Message(level, text) in messages.get_messages(request)` before adding (the `in` check doesn't consume):

```python
from django.contrib.messages import Message

def add_once(request, level, text):
    if Message(level, text) not in messages.get_messages(request):
        messages.add_message(request, level, text)
```

---

## Step 29: Reusable Apps — `fail_silently`

If you write an app that **other** projects install, you can't be sure they enabled `MessageMiddleware`. Use `fail_silently=True`:

```python
messages.success(request, 'Bookmark saved.', fail_silently=True)
```

- With the middleware → works normally.
- Without it → the message is dropped silently instead of raising `MessageFailure`.
- `fail_silently` does **not** hide the `TypeError` for passing a non-request object.

In your **own** project, leave the default `False` — you want to know when messages are broken.

---

## Step 30: Messages in Async Views

The messages API is plain in-memory work, so it's safe to call from `async def` views:

```python
import asyncio


async def async_messages_demo(request):
    await asyncio.sleep(0.1)                      # pretend to do async work
    messages.info(request, 'Added from an async view.')
    return redirect('blog:messages_demo')
```

- Adding messages doesn't touch the database (it only queues them in memory). Saving happens in `MessageMiddleware`, which supports both sync and async.
- **Iterating** messages in an async view is also fine with `CookieStorage`/`FallbackStorage` when everything fits in the cookie. If messages overflowed into the **session** with a database backend, reading them is a database query → `SynchronousOnlyOperation`. Load them in a thread:

  ```python
  from asgiref.sync import sync_to_async

  async def async_inbox(request):
      waiting = await sync_to_async(list)(messages.get_messages(request))
      ...
  ```

- The same caveat applies to templates: `render()` in an async view runs the template right there, so `{% for message in messages %}` is fine while messages fit in the cookie, but would hit the database if they overflowed into a DB-backed session. Keep messages short (Step 35.3), or iterate them with `sync_to_async` as above and pass the list to the template under another name.

---

## Step 31: Write Your Own Storage Backend

You rarely need this, but writing one shows exactly how the app works.

### 31.1 The interface

Subclass `BaseStorage` and implement **two** methods:

| Method                                         | Must                                                                                                              |
|------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| `_get(self, *args, **kwargs)`                  | Return a tuple `(messages, all_retrieved)`: a list of `Message` objects (or `None` if this storage was never used) and `True` if everything was retrieved |
| `_store(self, messages, response, *args, **kwargs)` | Save the list (an empty list means "delete everything") and return a list of messages it **couldn't** store (usually `[]`) |

Everything else — `add()`, iteration, `used`, levels, `update()` — comes from `BaseStorage`. `self.request` is the current request.

Use `MessageEncoder` / `MessageDecoder` from `django.contrib.messages.storage.cookie` to turn messages into JSON and back (they keep the "safe" flag).

### 31.2 Extend a built-in — log every message

Create `config/message_storage.py`:

```python
import json
import logging

from django.contrib.messages.storage.base import BaseStorage
from django.contrib.messages.storage.cookie import MessageDecoder, MessageEncoder
from django.contrib.messages.storage.fallback import FallbackStorage
from django.core.cache import cache

logger = logging.getLogger(__name__)


class LoggingFallbackStorage(FallbackStorage):
    """The default storage, plus a log line for every message that is added."""

    def add(self, level, message, extra_tags=''):
        if message and int(level) >= self.level:
            logger.info('Message for %s [%s]: %s', self.request.path, level, message)
        super().add(level, message, extra_tags)
```

```python
MESSAGE_STORAGE = 'config.message_storage.LoggingFallbackStorage'
```

### 31.3 From scratch — a cache-based storage

Messages in the cache, keyed by the visitor's session key. Add to the same file:

```python
class CacheStorage(BaseStorage):
    """Store messages in the default cache, under the visitor's session key."""

    timeout = 60 * 60          # unread messages are forgotten after one hour

    def _cache_key(self, create=False):
        session = self.request.session
        if session.session_key is None:
            if not create:
                return None
            session.save()             # a new visitor needs a session key
            session.modified = True    # make SessionMiddleware send the sessionid cookie
        return f'messages:{session.session_key}'

    def _get(self, *args, **kwargs):
        key = self._cache_key()
        data = cache.get(key) if key else None
        if data is None:
            return None, True          # None = "this storage was never used"
        return json.loads(data, cls=MessageDecoder), True

    def _store(self, messages, response, *args, **kwargs):
        if messages:
            cache.set(self._cache_key(create=True), json.dumps(messages, cls=MessageEncoder), self.timeout)
        else:
            key = self._cache_key()
            if key:
                cache.delete(key)
        return []                      # everything fits
```

```python
MESSAGE_STORAGE = 'config.message_storage.CacheStorage'
```

Limitations, so you know why the built-ins are designed as they are:

- It needs sessions (like `SessionStorage`), and `login()`/`logout()` change the session key, so unread messages from **earlier** requests are lost at login/logout (messages added in the login/logout request itself are fine — they are stored at the end, under the new key).
- `LocMemCache` is per process: with several worker processes, a message stored by one may be read by another that doesn't have it. Use Redis or Memcached.
- The cache can evict entries under memory pressure — messages may vanish.

Switch back to the default afterwards (remove `MESSAGE_STORAGE` or set it to the fallback class).

### 31.4 Custom storage in a `FallbackStorage` chain

`FallbackStorage.storage_classes` is just a tuple. For example, cookie first, then cache:

```python
from django.contrib.messages.storage.cookie import CookieStorage


class CookieThenCacheStorage(FallbackStorage):
    storage_classes = (CookieStorage, CacheStorage)
```

Every class in the chain except the last must support `remove_oldest=False` in `_store()` and report `all_retrieved=False` when it couldn't hold everything (as `CookieStorage` does with its "not finished" marker). The **last** one must accept everything.

---

## Step 32: When Not to Use Messages — Persistent Notifications

Messages are one-shot and live in one browser. For notifications that must **wait for a specific user** (on any device), stay until read, or be sent from background jobs, use a model.

### 32.1 A minimal `Notification` model

Add to `accounts/models.py`:

```python
from django.conf import settings
from django.db import models


class Notification(models.Model):
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name='notifications')
    text = models.CharField(max_length=255)
    level = models.PositiveSmallIntegerField(default=20)        # reuse the message levels
    is_read = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ['-created_at']

    def __str__(self):
        return self.text
```

```bash
python manage.py makemigrations accounts
python manage.py migrate
```

### 32.2 Bridge: turn unread notifications into flash messages on the next page

`accounts/middleware.py` — add:

```python
from django.contrib import messages


class NotificationsToMessagesMiddleware:
    """Show each unread Notification once as a flash message, then mark it read."""

    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        if request.user.is_authenticated and request.method == 'GET' \
                and request.headers.get('X-Requested-With') is None:
            unread = list(request.user.notifications.filter(is_read=False)[:5])
            for notification in unread:
                messages.add_message(request, notification.level, notification.text)
            if unread:
                request.user.notifications.filter(pk__in=[n.pk for n in unread]).update(is_read=True)
        return self.get_response(request)
```

Add `'accounts.middleware.NotificationsToMessagesMiddleware'` **below** `AuthenticationMiddleware` and `MessageMiddleware`.

Now anything — a comment signal, a Celery task, an admin action — can notify a user:

```python
Notification.objects.create(user=post.author, text=f'New comment on "{post.title}".', level=20)
```

…and the author sees it as a normal flash message on their next page load.

| Feature                          | `django.contrib.messages`   | `Notification` model            |
|----------------------------------|-----------------------------|---------------------------------|
| Stored in                        | Cookie / session            | Database                        |
| For                              | The current browser         | A specific user, any device     |
| Can be sent without a request    | No                          | Yes                             |
| Shown                            | Once                        | Until marked read               |
| History                          | No                          | Yes                             |

---

## Step 33: Test Messages

### 33.1 With the test client — `get_messages(response.wsgi_request)`

Add to `blog/tests.py`:

```python
from django.contrib import messages
from django.contrib.auth import get_user_model
from django.contrib.auth.models import Permission
from django.contrib.messages import Message, get_messages
from django.test import TestCase
from django.urls import reverse

from .models import Post

User = get_user_model()


class MessagesDemoTests(TestCase):
    def test_add_redirects_with_success_message(self):
        response = self.client.post(reverse('blog:messages_add'))

        self.assertRedirects(response, reverse('blog:messages_demo'))
        stored = [(m.level, str(m)) for m in get_messages(response.wsgi_request)]
        self.assertIn((messages.SUCCESS, 'Hello! This is your first message.'), stored)

    def test_message_is_shown_once(self):
        self.client.post(reverse('blog:messages_add'))

        first = self.client.get(reverse('blog:messages_demo'))
        self.assertContains(first, 'Hello! This is your first message.')

        second = self.client.get(reverse('blog:messages_demo'))
        self.assertNotContains(second, 'Hello! This is your first message.')
```

- `response.wsgi_request` is the request object the view used — its `_messages` storage still holds the messages added in that request.
- The test client keeps cookies between requests (like a browser), so `messages` travels from the POST to the next GET.
- Iterating `get_messages(response.wsgi_request)` after the response is fine — that request is already finished.

### 33.2 Follow the redirect — `follow=True` and `response.context['messages']`

```python
    def test_follow_redirect(self):
        response = self.client.post(reverse('blog:messages_add'), follow=True)

        self.assertEqual(response.status_code, 200)
        self.assertEqual(response.redirect_chain, [(reverse('blog:messages_demo'), 302)])
        texts = [str(m) for m in response.context['messages']]
        self.assertEqual(texts, ['Hello! This is your first message.'])
        self.assertContains(response, 'Hello! This is your first message.')
```

(Use the original one-message version of `messages_add` from Step 3.1 for these tests.)

### 33.3 `MessagesTestMixin.assertMessages()` (Django 5.0+)

```python
from django.contrib.messages.test import MessagesTestMixin


class PostMessagesTests(MessagesTestMixin, TestCase):
    @classmethod
    def setUpTestData(cls):
        cls.editor = User.objects.create_user('editor', password='pass12345')
        cls.editor.user_permissions.add(
            Permission.objects.get(codename='publish_post'),
            Permission.objects.get(codename='add_post'),
        )
        cls.post = Post.objects.create(title='Hello', slug='hello', content='…',
                                       author=cls.editor, status='draft')

    def setUp(self):
        self.client.force_login(self.editor)

    def test_publish_message(self):
        response = self.client.post(reverse('blog:post_publish', args=[self.post.pk]))

        self.assertMessages(response, [Message(messages.SUCCESS, '"Hello" is now published.')])

    def test_publish_twice_gives_info(self):
        self.post.status = 'published'
        self.post.save()

        response = self.client.post(reverse('blog:post_publish', args=[self.post.pk]))

        self.assertMessages(response, [Message(messages.INFO, '"Hello" was already published.')])

    def test_create_view_success_message(self):
        response = self.client.post(reverse('blog:post_create'), {
            'title': 'New one', 'slug': 'new-one', 'content': 'Text',
        })

        self.assertMessages(response, [Message(messages.SUCCESS, '"New one" was created as a draft.')])

    def test_only_check_one_message(self):
        response = self.client.post(reverse('blog:post_publish', args=[self.post.pk]))

        texts = [str(m) for m in get_messages(response.wsgi_request)]
        self.assertIn('"Hello" is now published.', texts)
```

- `assertMessages(response, expected, ordered=True)` compares `list(get_messages(response.wsgi_request))` with your list. `ordered=False` ignores the order (`assertCountEqual`).
- `Message.__eq__` compares **level and text only** — `extra_tags` aren't checked. Check `m.extra_tags` yourself if they matter.
- `assertMessages` checks the **complete** list, including messages from earlier requests that weren't displayed yet. When you only care about one message, use `assertIn` on the texts (`test_only_check_one_message`).
- `force_login()` doesn't add the "Welcome back" message: its internal request has no message storage, and the receiver uses `fail_silently=True` (Step 21.1). Without `fail_silently=True`, `force_login()` would raise `MessageFailure`.

### 33.4 Unit-test a view with `RequestFactory`

`RequestFactory` skips middleware, so `request._messages` doesn't exist and `messages.success()` raises `MessageFailure`. Add the storage yourself:

```python
from django.contrib.messages.storage.fallback import FallbackStorage
from django.contrib.sessions.backends.db import SessionStore
from django.test import RequestFactory

from . import views


class RequestFactoryMessagesTests(TestCase):
    def test_messages_add_view(self):
        request = RequestFactory().post('/messages-demo/add/')
        request.session = SessionStore()             # FallbackStorage → SessionStorage needs a session
        request._messages = FallbackStorage(request)  # what MessageMiddleware would do

        response = views.messages_add(request)

        self.assertEqual(response.status_code, 302)
        self.assertEqual([str(m) for m in request._messages], ['Hello! This is your first message.'])
```

Simpler alternative when you don't need sessions:

```python
from django.contrib.messages.storage.cookie import CookieStorage

request._messages = CookieStorage(request)
```

### 33.5 Override message settings in tests

```python
from django.test import override_settings
from django.contrib.messages import constants as message_constants


@override_settings(MESSAGE_LEVEL=message_constants.DEBUG)
def test_debug_messages_recorded(self):
    response = self.client.post(reverse('blog:messages_add'))
    levels = [m.level for m in get_messages(response.wsgi_request)]
    ...


@override_settings(MESSAGE_TAGS={message_constants.ERROR: 'danger'})
def test_error_tag_renamed(self):
    ...   # message.tags == 'danger' — works because the app refreshes LEVEL_TAGS on setting_changed


@override_settings(MESSAGE_STORAGE='django.contrib.messages.storage.session.SessionStorage')
def test_with_session_storage(self):
    response = self.client.post(reverse('blog:messages_add'))
    self.assertIn('_messages', self.client.session)
```

### 33.6 Test the admin action messages

```python
class PostAdminActionTests(MessagesTestMixin, TestCase):
    def setUp(self):
        self.admin = User.objects.create_superuser('boss', 'boss@example.com', 'pass12345')
        self.client.force_login(self.admin)
        self.post = Post.objects.create(title='A', slug='a', content='…', author=self.admin, status='draft')

    def test_make_published(self):
        response = self.client.post(reverse('admin:blog_post_changelist'), {
            'action': 'make_published',
            '_selected_action': [self.post.pk],
        })
        self.assertMessages(response, [Message(messages.SUCCESS, '1 post was published.')])
```

### 33.7 Run the tests

```bash
python manage.py test blog
```

---

## Step 34: Security

| Risk                                       | What to do                                                                                                     |
|--------------------------------------------|----------------------------------------------------------------------------------------------------------------|
| **XSS** through messages                   | Never `mark_safe()` text containing user input; use `format_html()`. Never use `{{ message|safe }}` (Step 25) |
| **Readable cookie**                        | Cookie storage is signed, **not encrypted**. No secrets, tokens, or private data in messages (Step 13.3)       |
| **Forged messages**                        | Prevented by the HMAC signature — keep `SECRET_KEY` secret; tampered cookies are ignored and deleted           |
| **Cookie stolen over HTTP**                | `SESSION_COOKIE_SECURE = True` in production → the `messages` cookie is `Secure` too                           |
| **Cookie read by JavaScript**              | `SESSION_COOKIE_HTTPONLY = True` (default) → the `messages` cookie is `HttpOnly` too                          |
| **Cross-site requests**                    | `SESSION_COOKIE_SAMESITE = 'Lax'` (default) applies to the `messages` cookie                                  |
| **Account enumeration**                    | Don't say "No account with this email" on password reset — use the same message for both cases               |
| **Leaking internals**                      | Don't show exception text in messages; log it instead (Step 19.3)                                              |
| **Open redirect** after adding a message   | Validate `next` URLs with `url_has_allowed_host_and_scheme()` (Step 17.7)                                     |
| **Rotating `SECRET_KEY`**                  | Waiting cookie messages become invalid (harmless); use `SECRET_KEY_FALLBACKS` for a smooth rotation           |

A good password-reset message, for example:

```python
messages.info(request, 'If an account exists for that email, we have sent a reset link.')
```

---

## Step 35: Performance and Caching

### 35.1 Cost of messages

| Operation                                     | Cost (default `FallbackStorage`)                                    |
|-----------------------------------------------|---------------------------------------------------------------------|
| A request with no messages, nothing displayed | Nothing — the storage is created but never read                     |
| Rendering a page with `{% if messages %}`     | Read + verify one cookie (no DB) if a `messages` cookie exists       |
| Adding a message                              | One `Set-Cookie` header on the response                             |
| Overflow                                      | Session read/write (a DB query with the db session backend)         |

The context processor returns the storage itself, which reads the cookie/session **only when a template uses `messages`**.

### 35.2 Don't cache pages that show messages

If you cache a page with `cache_page`, a CDN, or a reverse proxy **while a message is on it**, every visitor gets that message baked into the HTML. And the user who triggered it gets a cached page **without** it.

- Pages that display messages (practically every page with `base.html`) must not be cached as a whole for logged-in users or after POSTs.
- Django's cache middleware only refuses to cache in a few cookie situations (for example, a response that sets a cookie for a request that came without any cookies). A page that displays a message and then *deletes* the `messages` cookie can still end up cached, and CDNs and reverse proxies have their own rules. Don't rely on any of this to protect you.
- Safe approach: cache **fragments** (`{% cache 600 sidebar %}…{% endcache %}`) and never put `{% include 'blog/includes/messages.html' %}` inside a cached fragment.
- For full-page caching (CDN), load messages via a small uncached AJAX endpoint (Step 27.2) instead of rendering them in the HTML.

### 35.3 Keep messages short

Long messages fill the 2 KB cookie quickly and spill into the session (database writes). One sentence per message, at most a few messages per request.

---

## Step 36: Production Checklist

- [ ] `'django.contrib.messages'` in `INSTALLED_APPS`.
- [ ] `SessionMiddleware` **above** `MessageMiddleware`; any middleware that adds messages **below** it.
- [ ] `'django.contrib.messages.context_processors.messages'` in `TEMPLATES`.
- [ ] One messages include in `base.html`, outside overridable blocks; included in pages that don't extend `base.html` (login, logged out).
- [ ] `MESSAGE_LEVEL` is `INFO` in production (no debug messages for users).
- [ ] Every successful POST: message **and** redirect.
- [ ] No user input passed through `mark_safe()`; no `|safe` in the messages template.
- [ ] No secrets or personal data in messages.
- [ ] `SESSION_COOKIE_SECURE = True` (also secures the `messages` cookie).
- [ ] AJAX / polling endpoints don't consume messages unintentionally.
- [ ] Pages with messages aren't served from a full-page cache.
- [ ] Messages are translated (`gettext` / `ngettext`) if the site is multilingual.
- [ ] Tests cover the important messages (`assertMessages` / `get_messages(response.wsgi_request)`).
- [ ] `python manage.py check --deploy` passes.

---

## Reference: API (`from django.contrib import messages`)

| Function / class                                                         | Returns                    | Notes                                                              |
|--------------------------------------------------------------------------|----------------------------|--------------------------------------------------------------------|
| `messages.add_message(request, level, message, extra_tags='', fail_silently=False)` | `None`         | Ignores empty messages and levels below the minimum                |
| `messages.debug(request, message, extra_tags='', fail_silently=False)`   | `None`                     | Level 10                                                           |
| `messages.info(…)`                                                       | `None`                     | Level 20                                                           |
| `messages.success(…)`                                                    | `None`                     | Level 25                                                           |
| `messages.warning(…)`                                                    | `None`                     | Level 30                                                           |
| `messages.error(…)`                                                      | `None`                     | Level 40                                                           |
| `messages.get_messages(request)`                                         | Storage, or `[]`           | Iterating consumes                                                 |
| `messages.get_level(request)`                                            | `int`                      | Current minimum level                                              |
| `messages.set_level(request, level)`                                     | `bool`                     | `None` resets to `MESSAGE_LEVEL`; `False` if no middleware          |
| `messages.MessageFailure`                                                | exception                  | Raised when the middleware is missing and `fail_silently=False`    |
| `messages.Message(level, message, extra_tags=None)`                      | `Message`                  | Mostly for tests                                                   |
| `messages.views.SuccessMessageMixin`                                     | mixin                      | `success_message`, `get_success_message(cleaned_data)`             |
| `messages.test.MessagesTestMixin`                                        | mixin                      | `assertMessages(response, expected, *, ordered=True)` (5.0+)       |
| `ModelAdmin.message_user(request, message, level=INFO, extra_tags='', fail_silently=False)` | `None`  | Level may be a string (`'warning'`)                                |

## Reference: Constants (`django.contrib.messages.constants`)

| Name             | Value                                                                              |
|------------------|------------------------------------------------------------------------------------|
| `DEBUG`          | `10`                                                                               |
| `INFO`           | `20`                                                                               |
| `SUCCESS`        | `25`                                                                               |
| `WARNING`        | `30`                                                                               |
| `ERROR`          | `40`                                                                               |
| `DEFAULT_TAGS`   | `{10: 'debug', 20: 'info', 25: 'success', 30: 'warning', 40: 'error'}`             |
| `DEFAULT_LEVELS` | `{'DEBUG': 10, 'INFO': 20, 'SUCCESS': 25, 'WARNING': 30, 'ERROR': 40}` (= `DEFAULT_MESSAGE_LEVELS` in templates) |

## Reference: `Message`

| Member        | Description                                              |
|---------------|----------------------------------------------------------|
| `level`       | `int`                                                    |
| `message`     | The text                                                 |
| `extra_tags`  | `str` (`''` when added through the API)                  |
| `level_tag`   | Tag for the level (`DEFAULT_TAGS` + `MESSAGE_TAGS`) or `''` |
| `tags`        | `extra_tags` + `level_tag`, space-separated              |
| `__str__`     | The text                                                 |
| `__eq__`      | Same `level` and `message`                               |

## Reference: Storage (`BaseStorage` and subclasses)

| Member                         | Description                                                                         |
|--------------------------------|-------------------------------------------------------------------------------------|
| `request`                      | The request                                                                         |
| `used`                         | `True` after iteration; set to `False` to keep messages                             |
| `added_new`                    | `True` after a message was added in this request                                    |
| `level`                        | Minimum level (get / set / `del`)                                                   |
| `add(level, message, extra_tags='')` | Queue a message                                                               |
| `update(response)`             | Called by the middleware; stores what's left; returns unstored messages            |
| `__iter__`, `__len__`, `__contains__` | Iterate (consumes), count, membership                                        |
| `_get()`, `_store()`           | Implemented by each backend                                                         |
| `CookieStorage.cookie_name`    | `'messages'`                                                                        |
| `CookieStorage.max_cookie_size`| `2048`                                                                              |
| `CookieStorage.key_salt`       | `'django.contrib.messages'`                                                         |
| `SessionStorage.session_key`   | `'_messages'`                                                                       |
| `FallbackStorage.storage_classes` | `(CookieStorage, SessionStorage)`                                                |

## Reference: Settings

| Setting                     | Default                                                         | Purpose                                         |
|-----------------------------|-----------------------------------------------------------------|-------------------------------------------------|
| `MESSAGE_STORAGE`           | `'django.contrib.messages.storage.fallback.FallbackStorage'`    | Storage class (Steps 12–16)                     |
| `MESSAGE_LEVEL`             | `messages.INFO` (20)                                            | Minimum level recorded (Step 8)                 |
| `MESSAGE_TAGS`              | `{}` (merged with `DEFAULT_TAGS`)                               | Level → CSS tag (Step 9)                        |
| `SESSION_COOKIE_DOMAIN`     | `None`                                                          | Also used for the `messages` cookie             |
| `SESSION_COOKIE_SECURE`     | `False`                                                         | Also used for the `messages` cookie             |
| `SESSION_COOKIE_HTTPONLY`   | `True`                                                          | Also used for the `messages` cookie             |
| `SESSION_COOKIE_SAMESITE`   | `'Lax'`                                                         | Also used for the `messages` cookie             |
| `SECRET_KEY` / `SECRET_KEY_FALLBACKS` | —                                                     | Sign / verify the `messages` cookie             |

## Reference: Template Context (from the context processor)

| Variable                  | Value                                         |
|---------------------------|-----------------------------------------------|
| `messages`                | The storage (iterate to get `Message` objects) |
| `DEFAULT_MESSAGE_LEVELS`  | `DEFAULT_LEVELS` dict                          |

## Reference: Errors and System Checks

| Error                                                                                                     | Cause                                                       |
|-----------------------------------------------------------------------------------------------------------|-------------------------------------------------------------|
| `MessageFailure: You cannot add messages without installing django.contrib.messages.middleware.MessageMiddleware` | Middleware missing, or `RequestFactory` without `_messages` |
| `TypeError: add_message() argument must be an HttpRequest object, not 'X'.`                                | Passed `self` (or something else) instead of the request    |
| `ImproperlyConfigured: The session-based temporary message storage requires session middleware…`         | No `SessionMiddleware`, or it's below `MessageMiddleware`   |
| `ValueError: Not all temporary messages could be stored.`                                                 | `DEBUG=True` and messages didn't fit (`CookieStorage` only) |
| `ValueError: Bad message level string: …`                                                                  | `message_user(level='oops')`                                 |
| `admin.E404`                                                                                              | Context processor missing (admin installed)                 |
| `admin.E406`                                                                                              | App missing from `INSTALLED_APPS` (admin installed)         |
| `admin.E409`                                                                                              | Middleware missing (admin installed)                        |

---

## Troubleshooting

| Problem                                                                    | Cause / fix                                                                                                            |
|----------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| Message never appears                                                      | The template doesn't iterate `messages` (include missing, overridden block, context processor missing) — Steps 1.3, 4.5 |
| `messages.debug()` does nothing                                            | `MESSAGE_LEVEL` is `INFO`; set it to `DEBUG` (Step 8)                                                                  |
| Message with an empty string doesn't appear                                | Empty messages are ignored by design (Step 5.4)                                                                        |
| Message appears one page too late                                          | You added it **after** rendering (e.g. in middleware after `get_response`), or the page it was meant for doesn't display messages |
| Message appears on a random later page                                     | Nobody displayed it — it waited (rule 4). Make sure the redirect target shows messages; or an AJAX call added it (Step 27) |
| Message shows twice                                                        | Two loops on the page, or it was added twice (two signal receivers, `form_valid` + `SuccessMessageMixin`) — Step 28.5 |
| Message disappears before the user sees it                                 | An AJAX/polling request rendered a template that iterates messages (Step 27.6)                                         |
| Message shows again after refresh                                          | The view rendered instead of redirecting **and** the page didn't iterate messages, or `storage.used = False` left on   |
| `SuccessMessageMixin` adds nothing                                         | Mixin after `CreateView` in the bases, `success_message` empty, or your `form_valid` doesn't call `super()` (Step 18.3) |
| `KeyError` in `get_success_message`                                        | `%(field)s` isn't a form field; override `get_success_message` (Step 18.4)                                            |
| `TypeError: not enough arguments for format string` / `unsupported format character` | A literal `%` in `success_message`; write `%%`                                                               |
| `AttributeError: 'QuerySet' object has no attribute 'success'`             | A local variable named `messages` shadows the module (Step 17.4)                                                       |
| HTML shows as `&lt;a href…`                                                | Escaped as designed — use `format_html()` (Step 25)                                                                    |
| Admin error messages aren't red                                            | You renamed `ERROR` in `MESSAGE_TAGS`; add admin CSS (Step 20.4)                                                       |
| `ValueError: Not all temporary messages could be stored.`                  | Too many/long messages with `CookieStorage`; use the default `FallbackStorage` or shorter messages                     |
| Message shows on `localhost` but not on `127.0.0.1` after redirect         | Different hosts = different cookies; redirect to the same host                                                         |
| Messages missing behind a CDN / cache                                      | Cached HTML (Step 35.2)                                                                                               |
| Messages lost after deploy                                                 | `SECRET_KEY` changed (cookie signature invalid) — harmless, they're one-time anyway                                     |
| `SynchronousOnlyOperation` when showing messages in an async view          | Messages overflowed into a DB-backed session; read them with `sync_to_async` (Step 30)                                 |
| Message missing in a test with `RequestFactory`                            | Add `request.session` and `request._messages` (Step 33.4)                                                             |
| `MessageFailure` in tests on `self.client.login()` / `force_login()` / `logout()` | A `user_logged_in` / `user_logged_out` receiver adds a message; add `fail_silently=True` (Step 21.1)            |
| `assertMessages` fails because of an extra, older message                  | It checks the complete list; display/consume earlier messages first, or use `assertIn` (Step 33.3)                    |

---

## Final Project Structure

```
myproject/
├── manage.py
├── config/
│   ├── settings.py                  # MESSAGE_LEVEL, MESSAGE_TAGS, (MESSAGE_STORAGE), middleware order, context processor
│   └── message_storage.py           # (optional) LoggingFallbackStorage, CacheStorage — Step 31
├── templates/
│   └── admin/
│       └── base_site.html           # (optional) admin CSS for renamed tags — Step 20.4
├── accounts/
│   ├── middleware.py                # IdleTimeoutMiddleware, NotificationsToMessagesMiddleware — Step 32
│   ├── models.py                    # User, Notification — Step 32
│   └── signals.py                   # "Welcome back" / "logged out" messages — Step 21
├── blog/
│   ├── admin.py                     # message_user() in actions and hooks — Step 20
│   ├── decorators.py                # verified_email_required — Step 21.4
│   ├── message_levels.py            # NOTICE, CRITICAL, critical() — Step 10
│   ├── middleware.py                # SuperuserDebugMessagesMiddleware, HtmxMessagesMiddleware
│   ├── templatetags/
│   │   ├── __init__.py
│   │   └── message_tags.py          # unique_messages filter — Step 28.5
│   ├── tests.py                     # MessagesDemoTests, PostMessagesTests, RequestFactoryMessagesTests…
│   ├── urls.py                      # messages-demo/…, post_unpublish, reading_list_add_ajax
│   ├── views.py                     # messages_demo/add/peek/consume, SuccessMessageMixin views
│   ├── static/blog/
│   │   ├── style.css                # .messages styles — Step 23
│   │   └── messages.js              # close, auto-hide, showMessages(), postJSON() — Steps 23, 27
│   └── templates/blog/
│       ├── base.html                # {% include 'blog/includes/messages.html' %}
│       ├── messages_demo.html
│       └── includes/
│           ├── messages.html        # the one display loop
│           ├── messages_bootstrap.html
│           └── messages_oob.html    # HTMX out-of-band swap
├── social/
└── usersessions/
```

---

## Quick Reference

```python
# settings.py
INSTALLED_APPS = [..., 'django.contrib.sessions', 'django.contrib.messages', ...]
MIDDLEWARE = [
    ...,
    'django.contrib.sessions.middleware.SessionMiddleware',        # above
    ...,
    'django.contrib.messages.middleware.MessageMiddleware',        # below sessions
    ...,
]
TEMPLATES[0]['OPTIONS']['context_processors'] += ['django.contrib.messages.context_processors.messages']

from django.contrib.messages import constants as message_constants   # not "from django.contrib import messages"
MESSAGE_LEVEL = message_constants.DEBUG if DEBUG else message_constants.INFO
MESSAGE_TAGS = {message_constants.ERROR: 'danger'}
# MESSAGE_STORAGE = 'django.contrib.messages.storage.session.SessionStorage'

# views
from django.contrib import messages
messages.success(request, 'Saved.')
messages.error(request, 'Failed.', extra_tags='dismissible')
messages.add_message(request, messages.WARNING, 'Careful.')
return redirect('somewhere')                       # PRG

# class-based views
from django.contrib.messages.views import SuccessMessageMixin
class PostCreateView(LoginRequiredMixin, SuccessMessageMixin, CreateView):
    success_message = '"%(title)s" was created.'

# admin
self.message_user(request, '3 posts published.', messages.SUCCESS)

# reading
for m in messages.get_messages(request): m.level, m.message, m.tags     # consumes
len(messages.get_messages(request))                                       # doesn't consume
storage.used = False                                                      # keep after reading
messages.set_level(request, messages.DEBUG); messages.get_level(request)

# HTML
from django.utils.html import format_html
messages.success(request, format_html('<a href="{}">{}</a> saved.', url, title))

# tests
list(get_messages(response.wsgi_request))
response = self.client.post(url, follow=True); response.context['messages']
class T(MessagesTestMixin, TestCase): self.assertMessages(response, [Message(messages.SUCCESS, 'Saved.')])
request._messages = FallbackStorage(request)        # with RequestFactory (+ request.session)
```

```html
{% if messages %}
    <ul class="messages" role="status" aria-live="polite">
        {% for message in messages %}
            <li{% if message.tags %} class="{{ message.tags }}"{% endif %}>{{ message }}</li>
        {% endfor %}
    </ul>
{% endif %}

{% if message.level == DEFAULT_MESSAGE_LEVELS.ERROR %}…{% endif %}
```

---

## Next Steps

- **django-htmx** — helpers for HTMX requests (`request.htmx`), pairs well with Step 27.5.
- **django-bootstrap5** / **crispy-forms** — template tags that render messages and forms in Bootstrap style.
- **django-notifications-hq** — a ready-made persistent notification model (Step 32).
- **django-debug-toolbar** — shows the messages added during a request in its panel.
- Official docs:
  - The messages framework: https://docs.djangoproject.com/en/stable/ref/contrib/messages/
  - Messages settings: https://docs.djangoproject.com/en/stable/ref/settings/#messages
  - `SuccessMessageMixin`: https://docs.djangoproject.com/en/stable/ref/contrib/messages/#adding-messages-in-class-based-views
  - Testing tools (`MessagesTestMixin`): https://docs.djangoproject.com/en/stable/ref/contrib/messages/#testing
  - Cryptographic signing: https://docs.djangoproject.com/en/stable/topics/signing/
