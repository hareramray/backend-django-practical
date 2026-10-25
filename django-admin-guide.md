# Django Admin (`django.contrib.admin`): Step-by-Step Guide

`django.contrib.admin` is Django's built-in admin panel. It reads your models and automatically builds a web interface where staff users can **create, read, update and delete (CRUD)** records — no extra code for forms or pages needed.

This guide continues the `blog` app from [django-setup-guide.md](django-setup-guide.md). Every step builds on the previous one.

---

## Step 1: Check That the Admin Is Enabled

A new project from `startproject` already has the admin switched on. Confirm these settings in `config/settings.py`.

**`INSTALLED_APPS`** — the admin and the apps it depends on:

```python
INSTALLED_APPS = [
    'django.contrib.admin',         # the admin site
    'django.contrib.auth',          # users, groups, permissions
    'django.contrib.contenttypes',  # needed by auth/permissions
    'django.contrib.sessions',      # keeps admin users logged in
    'django.contrib.messages',      # "Saved successfully" messages
    'django.contrib.staticfiles',   # admin CSS/JS
    'blog',
]
```

**`MIDDLEWARE`** — must include:

```python
'django.contrib.sessions.middleware.SessionMiddleware',
'django.contrib.auth.middleware.AuthenticationMiddleware',
'django.contrib.messages.middleware.MessageMiddleware',
```

**`TEMPLATES` → `context_processors`** — must include:

```python
'django.template.context_processors.request',
'django.contrib.auth.context_processors.auth',
'django.contrib.messages.context_processors.messages',
```

**`config/urls.py`** — the admin URL:

```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('blog.urls')),
]
```

---

## Step 2: Create the Database Tables

The admin, users and permissions all need tables. Run:

```bash
python manage.py migrate
```

---

## Step 3: Create a Superuser

A **superuser** can log in to the admin and has every permission.

```bash
python manage.py createsuperuser
```

Enter a username, email (optional) and password.

> Forgot the password? Reset it with:
> `python manage.py changepassword <username>`

---

## Step 4: Log In to the Admin

```bash
python manage.py runserver
```

Open **http://127.0.0.1:8000/admin/** and log in.

You'll see **Authentication and Authorization** with **Users** and **Groups**. Your own models won't appear until you register them.

---

## Step 5: Prepare the Models

To show off more admin features, extend `blog/models.py` with extra fields and a `Comment` model:

```python
from django.conf import settings
from django.db import models


class Post(models.Model):
    STATUS_CHOICES = [
        ('draft', 'Draft'),
        ('published', 'Published'),
    ]

    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=200, unique=True)
    author = models.ForeignKey(
        settings.AUTH_USER_MODEL, on_delete=models.CASCADE, null=True, blank=True
    )
    content = models.TextField()
    status = models.CharField(max_length=10, choices=STATUS_CHOICES, default='draft')
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    def __str__(self):
        return self.title


class Comment(models.Model):
    post = models.ForeignKey(Post, on_delete=models.CASCADE, related_name='comments')
    name = models.CharField(max_length=100)
    body = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return f'Comment by {self.name}'
```

Apply the changes:

```bash
python manage.py makemigrations
python manage.py migrate
```

> If you already have posts in the database, `makemigrations` asks for a default value for `slug`. The easiest fix while learning is to delete `db.sqlite3` and the files in `blog/migrations/` (except `__init__.py`), then run both commands again and recreate the superuser.

> **Tip:** Always define `__str__` on your models. The admin uses it everywhere to show each record's name — without it you'll see "Post object (1)".

---

## Step 6: Register a Model (Basic)

The simplest registration, in `blog/admin.py`:

```python
from django.contrib import admin
from .models import Post

admin.site.register(Post)
```

Refresh the admin — a **Blog → Posts** section appears with full add, edit and delete pages.

---

## Step 7: Customize with `ModelAdmin`

A `ModelAdmin` class controls how a model looks and behaves in the admin. The `@admin.register` decorator is the cleaner way to register it.

Replace `blog/admin.py` with:

```python
from django.contrib import admin
from .models import Post


@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_display = ('title', 'author', 'status', 'created_at')
```

> Use **either** `admin.site.register(Post, PostAdmin)` **or** `@admin.register(Post)` — not both, or you'll get an `AlreadyRegistered` error.

---

## Step 8: Customize the List Page

These options control the page that lists all records:

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_display = ('title', 'author', 'status', 'created_at')  # columns
    list_display_links = ('title',)          # which columns link to the edit page
    list_filter = ('status', 'created_at')   # filter sidebar on the right
    search_fields = ('title', 'content')     # search box at the top
    ordering = ('-created_at',)              # default sort (newest first)
    date_hierarchy = 'created_at'            # date drill-down bar
    list_editable = ('status',)              # edit directly in the list
    list_per_page = 20                       # pagination
```

| Option              | What it does                                              |
|---------------------|-----------------------------------------------------------|
| `list_display`      | Columns shown in the list                                 |
| `list_display_links`| Columns that open the edit form                           |
| `list_filter`       | Sidebar filters                                           |
| `search_fields`     | Fields the search box looks in                            |
| `ordering`          | Default sort order (`-` means descending)                 |
| `date_hierarchy`    | Year → month → day navigation for a date field            |
| `list_editable`     | Fields editable inline in the list (can't also be a link) |
| `list_per_page`     | Rows per page                                             |

> To search a related field, use a double underscore: `search_fields = ('title', 'author__username')`

---

## Step 9: Add Custom Columns

You can show computed values as columns by writing a method and adding its name to `list_display`:

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_display = ('title', 'status', 'comment_count', 'is_published')

    @admin.display(description='Comments')
    def comment_count(self, obj):
        return obj.comments.count()

    @admin.display(boolean=True, description='Published?')
    def is_published(self, obj):
        return obj.status == 'published'
```

- `description` sets the column header.
- `boolean=True` shows a green tick or red cross instead of True/False.

---

## Step 10: Customize the Edit Form

These options control the add/change form:

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    prepopulated_fields = {'slug': ('title',)}   # auto-fill slug while typing the title
    readonly_fields = ('created_at', 'updated_at')
    raw_id_fields = ('author',)                  # ID input instead of a dropdown

    fieldsets = (
        (None, {
            'fields': ('title', 'slug', 'author', 'content'),
        }),
        ('Publishing', {
            'fields': ('status',),
        }),
        ('Dates', {
            'fields': ('created_at', 'updated_at'),
            'classes': ('collapse',),            # collapsed by default
        }),
    )
```

| Option                | What it does                                                  |
|-----------------------|---------------------------------------------------------------|
| `fields`              | Simple list of fields to show, in order                       |
| `fieldsets`           | Groups fields into titled sections (use instead of `fields`)  |
| `exclude`             | Fields to hide from the form                                  |
| `readonly_fields`     | Shown but not editable (needed for `auto_now` fields)         |
| `prepopulated_fields` | Auto-fill one field from others (great for slugs)             |
| `raw_id_fields`       | Text input for foreign keys (faster with many records)        |
| `autocomplete_fields` | Searchable dropdown for foreign keys (see note below)         |

> `autocomplete_fields = ('author',)` needs the related model's admin to have `search_fields`. The built-in `UserAdmin` already does, so it works for `author` out of the box.

---

## Step 11: Edit Related Records Inline

**Inlines** let you edit related objects (comments) on the parent's page (post).

```python
from django.contrib import admin
from .models import Post, Comment


class CommentInline(admin.TabularInline):   # or admin.StackedInline
    model = Comment
    extra = 1                                # number of empty forms shown


@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    inlines = [CommentInline]
```

- `TabularInline` — compact table, one row per comment.
- `StackedInline` — each comment as a full form, one below another.

You can also register `Comment` separately to manage all comments in one place:

```python
@admin.register(Comment)
class CommentAdmin(admin.ModelAdmin):
    list_display = ('name', 'post', 'created_at')
    list_filter = ('created_at',)
    search_fields = ('name', 'body')
```

---

## Step 12: Add Custom Actions

**Actions** run on several selected records at once from the list page (like the built-in "Delete selected").

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    actions = ['make_published', 'make_draft']

    @admin.action(description='Mark selected posts as published')
    def make_published(self, request, queryset):
        updated = queryset.update(status='published')
        self.message_user(request, f'{updated} post(s) marked as published.')

    @admin.action(description='Mark selected posts as draft')
    def make_draft(self, request, queryset):
        updated = queryset.update(status='draft')
        self.message_user(request, f'{updated} post(s) marked as draft.')
```

Select posts in the list → choose the action from the dropdown → click **Go**.

---

## Step 13: Override Admin Behavior

`ModelAdmin` has methods you can override for custom logic.

**Set the author automatically when saving:**

```python
    def save_model(self, request, obj, form, change):
        if not change:               # only when creating a new post
            obj.author = request.user
        super().save_model(request, obj, form, change)
```

**Show staff users only their own posts:**

```python
    def get_queryset(self, request):
        qs = super().get_queryset(request)
        if request.user.is_superuser:
            return qs
        return qs.filter(author=request.user)
```

**Block deleting published posts:**

```python
    def has_delete_permission(self, request, obj=None):
        if obj and obj.status == 'published':
            return False
        return super().has_delete_permission(request, obj)
```

---

## Step 14: Put It All Together

The complete `blog/admin.py`:

```python
from django.contrib import admin
from .models import Post, Comment


class CommentInline(admin.TabularInline):
    model = Comment
    extra = 1


@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    # List page
    list_display = ('title', 'author', 'status', 'comment_count', 'created_at')
    list_filter = ('status', 'created_at')
    search_fields = ('title', 'content', 'author__username')
    ordering = ('-created_at',)
    date_hierarchy = 'created_at'
    list_editable = ('status',)
    list_per_page = 20

    # Edit form
    prepopulated_fields = {'slug': ('title',)}
    readonly_fields = ('author', 'created_at', 'updated_at')
    fieldsets = (
        (None, {'fields': ('title', 'slug', 'author', 'content')}),
        ('Publishing', {'fields': ('status',)}),
        ('Dates', {'fields': ('created_at', 'updated_at'), 'classes': ('collapse',)}),
    )
    inlines = [CommentInline]
    actions = ['make_published', 'make_draft']

    @admin.display(description='Comments')
    def comment_count(self, obj):
        return obj.comments.count()

    @admin.action(description='Mark selected posts as published')
    def make_published(self, request, queryset):
        updated = queryset.update(status='published')
        self.message_user(request, f'{updated} post(s) marked as published.')

    @admin.action(description='Mark selected posts as draft')
    def make_draft(self, request, queryset):
        updated = queryset.update(status='draft')
        self.message_user(request, f'{updated} post(s) marked as draft.')

    def save_model(self, request, obj, form, change):
        if not change:
            obj.author = request.user
        super().save_model(request, obj, form, change)


@admin.register(Comment)
class CommentAdmin(admin.ModelAdmin):
    list_display = ('name', 'post', 'created_at')
    list_filter = ('created_at',)
    search_fields = ('name', 'body')
```

---

## Step 15: Change the Admin Site Titles

By default the header says "Django administration". Change it in `blog/admin.py` (or any `admin.py`):

```python
admin.site.site_header = 'My Blog Admin'      # top banner and login page
admin.site.site_title = 'My Blog Admin Portal' # browser tab title
admin.site.index_title = 'Welcome to the Blog Dashboard'  # admin home heading
```

---

## Step 16: Override Admin Templates (Optional)

For deeper visual changes, override the admin's own templates.

**1. Tell Django about a project-level templates folder** in `config/settings.py`:

```python
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [BASE_DIR / 'templates'],   # 👈 add this
        'APP_DIRS': True,
        # ...
    },
]
```

**2. Create `templates/admin/base_site.html`** in the project root:

```html
{% extends "admin/base_site.html" %}
{% load static %}

{% block extrastyle %}
    {{ block.super }}
    <style>
        #header { background: #1e3a5f; }
    </style>
{% endblock %}

{% block branding %}
    <div id="site-name"><a href="{% url 'admin:index' %}">📝 My Blog Admin</a></div>
{% endblock %}
```

Django loads your file first, and `{% extends %}` pulls in the original so you only change the blocks you need.

---

## Step 17: Manage Users, Groups and Permissions

The admin includes **Users** and **Groups** for access control.

**User flags:**

| Flag           | Meaning                                            |
|----------------|----------------------------------------------------|
| `is_active`    | Account can log in at all                          |
| `is_staff`     | Can log in to the admin                            |
| `is_superuser` | Has every permission automatically                 |

**Permissions:** for each model Django creates four permissions automatically — `add`, `change`, `delete` and `view` (e.g. `blog.add_post`).

**Example — create an "Editors" group:**

1. Go to **Admin → Groups → Add group**.
2. Name it `Editors`.
3. Give it `blog | post | Can add post`, `Can change post`, `Can view post`.
4. Save.
5. Go to **Users**, create or edit a user, tick **Staff status**, and add them to the `Editors` group.

That user can now log in to the admin and manage posts, but can't delete them or see Users and Groups.

---

## Step 18: Secure the Admin

The admin has full control over your data, so protect it before going live:

- **Change the URL** so bots can't find it easily:
  ```python
  path('my-secret-panel/', admin.site.urls),
  ```
- Use **strong passwords** for staff accounts.
- Always use **HTTPS** in production.
- Set `DEBUG = False` in production.
- Give staff users **only the permissions they need** (use Groups).
- Consider packages like `django-axes` (blocks repeated failed logins) or `django-otp` (two-factor authentication).

---

## Quick Reference

| Task                            | Code / Command                                     |
|---------------------------------|----------------------------------------------------|
| Create admin user               | `python manage.py createsuperuser`                 |
| Reset a password                | `python manage.py changepassword <username>`       |
| Register a model (basic)        | `admin.site.register(Post)`                        |
| Register with options           | `@admin.register(Post)` on a `ModelAdmin` class    |
| List columns                    | `list_display = (...)`                             |
| Sidebar filters                 | `list_filter = (...)`                              |
| Search box                      | `search_fields = (...)`                            |
| Auto-fill slug                  | `prepopulated_fields = {'slug': ('title',)}`       |
| Non-editable fields             | `readonly_fields = (...)`                          |
| Group form fields               | `fieldsets = (...)`                                |
| Edit related objects            | `inlines = [MyInline]`                             |
| Bulk actions                    | `@admin.action` + `actions = [...]`                |
| Custom column                   | `@admin.display` method in `list_display`          |
| Change header text              | `admin.site.site_header = '...'`                   |

---

## Next Steps

- Explore third-party admin themes like **django-jazzmin** or **django-unfold**.
- Add data import/export with **django-import-export**.
- Read the official admin docs: https://docs.djangoproject.com/en/stable/ref/contrib/admin/
