# Django Content Types (`django.contrib.contenttypes`): Step-by-Step Guide

`django.contrib.contenttypes` is a small built-in Django app that **keeps a list of every model in your project** and lets you **point at "any model" from one place**. It gives you:

| Part                        | What it does                                                                                  |
|-----------------------------|-----------------------------------------------------------------------------------------------|
| **`ContentType` model**     | One database row per model (`blog.post`, `accounts.user`, …) — a registry of your models      |
| **`ContentTypeManager`**    | Fast, cached lookups: "give me the content type for `Post`" / "give me the model for id 7"    |
| **`GenericForeignKey`**     | A foreign key that can point to a row in **any** table (comments on posts *and* photos)       |
| **`GenericRelation`**       | The reverse side of a generic foreign key (`post.comments.all()`)                             |
| **Generic inlines (admin)** | `GenericTabularInline` / `GenericStackedInline` to edit generic objects inside the admin      |
| **Generic formsets**        | `generic_inlineformset_factory()` to edit generic objects in your own forms                   |
| **`GenericPrefetch`**       | Efficient prefetching across several models (Django 5.0+)                                     |
| **`shortcut` view**         | The admin's "View on site" redirect (`/admin/r/<content_type_id>/<object_id>/`)               |
| **`remove_stale_contenttypes`** | Management command that deletes rows for models that no longer exist                      |
| **Signals & checks**        | Creates/renames rows automatically during `migrate`, validates your generic fields            |

Two words you will see a lot:

- **Content type** = a row in the `django_content_type` table that *describes a model* (its app label + model name). It is **not** an HTTP `Content-Type` header — completely different thing.
- **Generic relation** = a relation where the target model is **not fixed in code**; it is stored per row as a content type + an object id.

This guide continues the `blog` project from [django-setup-guide.md](django-setup-guide.md), [django-admin-guide.md](django-admin-guide.md) and [django-auth-guide.md](django-auth-guide.md) (project package `config`, apps `blog` and `accounts`, custom user `accounts.User`). Every step builds on the previous one. Commands are for **Windows** (PowerShell); they are the same on macOS/Linux except where noted.

---

## Prerequisites

- The project from the earlier guides, with the virtual environment **activated** (`(venv)` in your prompt).
- **Django 5.0+** recommended (needed for `GenericPrefetch` in Step 19). Check with:
  ```bash
  python -m django --version
  ```
- Migrations applied (`python manage.py migrate`) and a superuser created.

---

## Step 0: How Content Types Work (Big Picture)

Normally a `ForeignKey` points to **one fixed table**:

```
Comment.post_id ──────────────► blog_post.id          (can only ever point to a Post)
```

With content types, a row stores **two** values — *which table* and *which row*:

```
                     ┌─────────────────────────────────┐
                     │      django_content_type        │
                     ├────┬───────────┬────────────────┤
                     │ id │ app_label │ model          │
                     ├────┼───────────┼────────────────┤
                     │  1 │ admin     │ logentry       │
                     │  2 │ auth      │ permission     │
                     │  3 │ auth      │ group          │
                     │  4 │ contenttypes │ contenttype │
                     │  5 │ sessions  │ session        │
                     │  6 │ accounts  │ user           │
                     │  7 │ blog      │ post   ◄───────┼──┐
                     │  8 │ blog      │ photo  ◄───────┼──┼──┐
                     └────┴───────────┴────────────────┘  │  │
                                                          │  │
  social_comment                                          │  │
  ┌────┬─────────────────┬───────────┬───────────────┐    │  │
  │ id │ content_type_id │ object_id │ text          │    │  │
  ├────┼─────────────────┼───────────┼───────────────┤    │  │
  │  1 │        7 ───────┼───────────┼───────────────┼────┘  │   → blog_post row 3
  │  2 │        8 ───────┼───────────┼───────────────┼───────┘   → blog_photo row 1
  └────┴─────────────────┴───────────┴───────────────┘
           "which table"   "which row"
```

The full flow:

1. You run `python manage.py migrate`.
2. After migrations finish, Django fires the **`post_migrate` signal**.
3. `contenttypes` listens to it and inserts **one row per model** into `django_content_type` (if missing).
4. Other apps use those rows:
   - **`auth`** — every `Permission` has a `content_type` (that is how `blog.add_post` knows it belongs to `Post`).
   - **`admin`** — every history `LogEntry` has a `content_type` (that is how admin history knows *what* was edited).
   - **Your code** — `GenericForeignKey` stores `content_type_id` + `object_id` to point at any model.
5. At runtime, `ContentType.objects.get_for_model(Post)` returns the row for `Post` (cached in memory after the first lookup), and `content_type.model_class()` turns a row back into the Python class.

> **IDs differ between databases.** `blog.post` might be id `7` on your laptop and id `12` in production. **Never hard-code content type IDs** — always look them up by model or by `(app_label, model)`.

---

## Step 1: Check That Content Types Are Enabled

Open `config/settings.py`. A new project already includes it:

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',   # 👈 this one
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'accounts',
    'blog',
]
```

Things to know:

| Question                                   | Answer                                                                                                  |
|--------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Is it on by default?                       | **Yes**, `startproject` adds it.                                                                        |
| Does order in `INSTALLED_APPS` matter?     | No, not for contenttypes.                                                                               |
| Can I remove it?                           | Only if you also remove `django.contrib.auth` **and** `django.contrib.admin` — both depend on it. In practice: **keep it**. |
| Does it need middleware?                   | **No.** No middleware, no context processor, no setting.                                               |
| Does it need URLs?                         | **No.** (Optional: the `shortcut` view — Step 28.)                                                      |
| Any settings to configure?                 | **None.** The app has no settings of its own.                                                           |
| Does it need extra packages?               | No, it ships with Django.                                                                               |

Who depends on it:

| App / feature                           | How it uses content types                                     |
|-----------------------------------------|----------------------------------------------------------------|
| `django.contrib.auth`                   | `Permission.content_type` (FK to `ContentType`)                |
| `django.contrib.admin`                  | `LogEntry.content_type`, "View on site" button                 |
| `django.contrib.sites` + `shortcut`     | Builds full URLs for "View on site"                            |
| `django.contrib.flatpages`, `redirects` | Indirectly (they rely on `sites`/admin)                        |
| Third-party apps                        | django-taggit, django-comments, django-guardian, django-reversion, django-polymorphic, django-activity-stream… all use content types |

---

## Step 2: Know What the App Contains

Everything lives in the `django.contrib.contenttypes` package:

| Import path                                         | What you get                                                                 |
|-----------------------------------------------------|------------------------------------------------------------------------------|
| `django.contrib.contenttypes.models`                | `ContentType`, `ContentTypeManager`                                          |
| `django.contrib.contenttypes.fields`                | `GenericForeignKey`, `GenericRelation`                                       |
| `django.contrib.contenttypes.admin`                 | `GenericInlineModelAdmin`, `GenericTabularInline`, `GenericStackedInline`    |
| `django.contrib.contenttypes.forms`                 | `BaseGenericInlineFormSet`, `generic_inlineformset_factory()`                |
| `django.contrib.contenttypes.prefetch`              | `GenericPrefetch` (Django 5.0+)                                              |
| `django.contrib.contenttypes.views`                 | `shortcut` (the "View on site" redirect view)                                |
| `django.contrib.contenttypes.management`            | `create_contenttypes()` and the rename helpers used during `migrate`         |
| `django.contrib.contenttypes.management.commands`   | `remove_stale_contenttypes` command                                          |
| `django.contrib.contenttypes.checks`                | System checks for generic foreign keys and model name length                 |
| `django.contrib.contenttypes.apps`                  | `ContentTypesConfig` (connects the signals and checks in `ready()`)          |

What `ContentTypesConfig.ready()` does when Django starts:

```python
# simplified from django/contrib/contenttypes/apps.py
class ContentTypesConfig(AppConfig):
    default_auto_field = "django.db.models.AutoField"
    name = "django.contrib.contenttypes"
    verbose_name = _("Content Types")

    def ready(self):
        pre_migrate.connect(inject_rename_contenttypes_operations, sender=self)
        post_migrate.connect(create_contenttypes)
        checks.register(check_generic_foreign_keys, checks.Tags.models)
        checks.register(check_model_name_lengths, checks.Tags.models)
```

So, four jobs:

1. **Before migrating** — teach `RenameModel` operations to also rename the matching content type row (Step 30).
2. **After migrating** — create missing content type rows (Step 4).
3. **On `check`** — validate every `GenericForeignKey` (Step 33).
4. **On `check`** — make sure no model name is longer than 100 characters (the `model` column limit).

> You can open the source yourself: `venv\Lib\site-packages\django\contrib\contenttypes\` (on macOS/Linux: `venv/lib/python3.X/site-packages/django/contrib/contenttypes/`).

---

## Step 3: Run Migrations and Look at the Table

### 3.1 The app's migrations

```bash
python manage.py showmigrations contenttypes
```

Output:

```
contenttypes
 [X] 0001_initial
 [X] 0002_remove_content_type_name
```

- `0001_initial` — creates the table (originally with a `name` column).
- `0002_remove_content_type_name` — removes `name` (it is now a computed property, see Step 5).

### 3.2 See the SQL

```bash
python manage.py sqlmigrate contenttypes 0001
```

On SQLite you will see something like:

```sql
CREATE TABLE "django_content_type" (
    "id" integer NOT NULL PRIMARY KEY AUTOINCREMENT,
    "name" varchar(100) NOT NULL,
    "app_label" varchar(100) NOT NULL,
    "model" varchar(100) NOT NULL
);
CREATE UNIQUE INDEX "django_content_type_app_label_model_..._uniq"
    ON "django_content_type" ("app_label", "model");
```

After `0002`, the final table is:

| Column      | Type           | Notes                                                    |
|-------------|----------------|----------------------------------------------------------|
| `id`        | integer, PK    | Auto-increment. **Differs between databases.**           |
| `app_label` | varchar(100)   | The app's label, e.g. `blog`, `auth`, `accounts`         |
| `model`     | varchar(100)   | The model's **lowercase** class name, e.g. `post`        |
|             | unique         | `(app_label, model)` together are unique                 |

> The `model` column is `Model._meta.model_name`, which is the class name lowercased: `BlogPost` → `blogpost` (no underscore!).

### 3.3 Look at the rows directly

Option A — Django shell (always works):

```bash
python manage.py shell
```

```python
>>> from django.contrib.contenttypes.models import ContentType
>>> for ct in ContentType.objects.order_by('id'):
...     print(ct.id, ct.app_label, ct.model)
...
1 admin logentry
2 auth permission
3 auth group
4 contenttypes contenttype
5 sessions session
6 accounts user
7 blog post
```

(Your IDs and order may differ.)

Option B — database shell:

```bash
python manage.py dbshell
```

```sql
SELECT * FROM django_content_type;
.quit
```

> `dbshell` on SQLite needs the `sqlite3` program on your PATH. On Windows, download "sqlite-tools" from sqlite.org and add the folder to PATH, or just use Option A.

Option C — the admin. `ContentType` is **not registered** in the admin by default (on purpose: you should not edit these rows by hand). You can register it read-only for learning — see Step 10.3.

---

## Step 4: Understand How Rows Are Created Automatically

### 4.1 The `post_migrate` signal

Every time `migrate` finishes for an app, Django sends `post_migrate`. The receiver `create_contenttypes()` then:

1. Clears the content type cache (`ContentType.objects.clear_cache()`).
2. Gets all models of that app (`app_config.get_models()`).
3. Compares them with existing rows for that `app_label`.
4. **Bulk-creates** rows for models that are missing.
5. **Never deletes** anything (deleting is the job of `remove_stale_contenttypes`, Step 29).

Then `auth`'s own `post_migrate` receiver (`create_permissions`) runs and creates `add_`, `change_`, `delete_`, `view_` permissions — which need the content type rows, so `create_permissions` calls `create_contenttypes` first.

### 4.2 Watch it happen

Add any new model, then:

```bash
python manage.py makemigrations
python manage.py migrate --verbosity 2
```

With `--verbosity 2` you will see lines like:

```
Running post-migrate handlers for application blog
Adding content type 'blog | photo'
Adding permission 'Blog | photo | Can add photo'
...
```

### 4.3 Which models get a content type?

| Model kind                              | Gets a content type row? | Notes                                                    |
|-----------------------------------------|--------------------------|----------------------------------------------------------|
| Normal model                            | ✅ Yes                    |                                                          |
| `abstract = True` model                 | ❌ No                     | It has no table; only its concrete children get rows     |
| `proxy = True` model                    | ✅ Yes, its own row       | See Step 8 for how lookups treat proxies                 |
| `managed = False` model                 | ✅ Yes                    | Django does not create its table, but it is still a model |
| Auto-created M2M "through" table        | ❌ No                     | e.g. `accounts_user_groups` has no content type          |
| Explicit `through=` model you wrote     | ✅ Yes                    | It is a normal model                                     |
| Model in an app **not** in `INSTALLED_APPS` | ❌ No                 | And old rows become "stale" (Step 29)                    |
| Model in an app with **no migrations**  | ✅ Yes                    | `migrate --run-syncdb` still fires `post_migrate`        |

### 4.4 When does it run?

- Every `python manage.py migrate` (even when there is nothing to migrate).
- When the test runner creates the test database.
- `flush` also emits `post_migrate`, so rows are re-created after flushing.

It does **not** run on `runserver` or on normal requests. That is why, in a **data migration** that creates a model, the content type may not exist yet (Step 30.2).

---

## Step 5: Know the `ContentType` Model Inside Out

### 5.1 Fields

| Field       | Type                       | Example   |
|-------------|----------------------------|-----------|
| `id`        | `AutoField`                | `7`       |
| `app_label` | `CharField(max_length=100)`| `"blog"`  |
| `model`     | `CharField(max_length=100)`| `"post"`  |

`Meta`:

```python
class Meta:
    verbose_name = _("content type")
    verbose_name_plural = _("content types")
    db_table = "django_content_type"
    unique_together = [["app_label", "model"]]
```

### 5.2 Properties

| Property               | Returns                                                                     | Example                 |
|------------------------|-----------------------------------------------------------------------------|-------------------------|
| `ct.name`              | The model's `verbose_name` (falls back to `ct.model` if the model is gone)  | `"post"`                |
| `ct.app_labeled_name`  | `"<app verbose_name> \| <model verbose_name>"`                               | `"Blog \| post"`         |
| `str(ct)`              | Same as `app_labeled_name`                                                  | `"Blog \| post"`         |

### 5.3 Methods

| Method                                        | What it does                                                                                 |
|-----------------------------------------------|----------------------------------------------------------------------------------------------|
| `ct.model_class()`                            | Returns the Python model class (`Post`), or **`None`** if the model no longer exists        |
| `ct.get_object_for_this_type(**kwargs)`       | Like `Post.objects.get(**kwargs)` — returns **one** object, raises `DoesNotExist` if missing |
| `ct.get_all_objects_for_this_type(**kwargs)`  | Like `Post.objects.filter(**kwargs)` — returns a **QuerySet**                                |
| `ct.natural_key()`                            | Returns `(app_label, model)`, e.g. `("blog", "post")` — used in fixtures (Step 31)          |

> `get_object_for_this_type()` also accepts `using="other_db"` (Django 5.1+) to choose the database.

### 5.4 Manager methods (`ContentType.objects`)

| Method                                                   | What it does                                                                                     |
|----------------------------------------------------------|--------------------------------------------------------------------------------------------------|
| `get_for_model(model, for_concrete_model=True)`          | Content type for a model **class or instance**. Creates the row if missing. **Cached.**          |
| `get_for_models(*models, for_concrete_models=True)`      | Several at once → returns a `dict` `{ModelClass: ContentType}`. **Cached.**                      |
| `get_for_id(id)`                                         | Content type by primary key. **Cached** (unlike `.get(pk=...)`).                                 |
| `get_by_natural_key(app_label, model)`                   | Content type by `("blog", "post")`. **Cached.**                                                   |
| `clear_cache()`                                          | Empties the in-memory cache.                                                                     |
| Normal manager methods                                   | `.all()`, `.filter()`, `.get()` etc. still work but are **not cached**.                          |

---

## Step 6: Work With Content Types in the Shell

```bash
python manage.py shell
```

### 6.1 Import

```python
>>> from django.contrib.contenttypes.models import ContentType
>>> from blog.models import Post
>>> from accounts.models import User
```

### 6.2 List all content types

```python
>>> ContentType.objects.all()
<QuerySet [<ContentType: Administration | log entry>, <ContentType: Authentication and Authorization | permission>, <ContentType: Authentication and Authorization | group>, <ContentType: Content Types | content type>, <ContentType: Sessions | session>, <ContentType: Accounts | user>, <ContentType: Blog | post>]>

>>> ContentType.objects.count()
7

>>> ContentType.objects.filter(app_label='blog')
<QuerySet [<ContentType: Blog | post>]>
```

### 6.3 From a model to its content type

```python
>>> ct = ContentType.objects.get_for_model(Post)
>>> ct
<ContentType: Blog | post>
>>> ct.id, ct.app_label, ct.model
(7, 'blog', 'post')
>>> ct.name
'post'
>>> ct.app_labeled_name
'Blog | post'
```

Works with an **instance** too:

```python
>>> post = Post.objects.first()
>>> ContentType.objects.get_for_model(post)
<ContentType: Blog | post>
```

Several at once:

```python
>>> ContentType.objects.get_for_models(Post, User)
{<class 'blog.models.Post'>: <ContentType: Blog | post>, <class 'accounts.models.User'>: <ContentType: Accounts | user>}
```

### 6.4 From strings to a content type

```python
>>> ContentType.objects.get_by_natural_key('blog', 'post')      # cached
<ContentType: Blog | post>

>>> ContentType.objects.get(app_label='blog', model='post')     # not cached, same result
<ContentType: Blog | post>
```

> Remember `model` is **lowercase**: `model='Post'` raises `ContentType.DoesNotExist`.

### 6.5 From an ID to a content type

```python
>>> ContentType.objects.get_for_id(7)      # cached
<ContentType: Blog | post>
```

### 6.6 From a content type back to the model class

```python
>>> ct = ContentType.objects.get_by_natural_key('blog', 'post')
>>> PostModel = ct.model_class()
>>> PostModel
<class 'blog.models.Post'>
>>> PostModel.objects.count()
3
```

### 6.7 Fetch objects through a content type

```python
>>> ct.get_object_for_this_type(pk=1)
<Post: Hello Django>

>>> ct.get_all_objects_for_this_type(status='published')
<QuerySet [<Post: Hello Django>, <Post: Second post>]>

>>> ct.get_object_for_this_type(pk=9999)
Traceback (most recent call last):
  ...
blog.models.Post.DoesNotExist: Post matching query does not exist.
```

### 6.8 A "load any object" helper

This pattern is the heart of every generic feature — turn two simple values (`"blog.post"`, `1`) into a real object:

```python
>>> def load_object(app_label, model_name, pk):
...     ct = ContentType.objects.get_by_natural_key(app_label, model_name.lower())
...     return ct.get_object_for_this_type(pk=pk)
...
>>> load_object('blog', 'Post', 1)
<Post: Hello Django>
>>> load_object('accounts', 'user', 1)
<User: admin>
```

### 6.9 Natural key

```python
>>> ct.natural_key()
('blog', 'post')
```

### 6.10 When the model no longer exists

If you delete a model class but its row is still in the table:

```python
>>> old = ContentType.objects.get(app_label='blog', model='oldmodel')
>>> old.model_class()
None                         # not an error — check for None!
>>> old.name
'oldmodel'                   # falls back to the raw model string
```

---

## Step 7: Understand the Cache

`get_for_model()`, `get_for_models()`, `get_for_id()` and `get_by_natural_key()` store results in a **per-process, per-database dictionary** inside `ContentTypeManager`. The first call hits the database; later calls do not.

Prove it in the shell:

```python
>>> from django.db import connection, reset_queries
>>> from django.conf import settings
>>> settings.DEBUG = True          # queries are only recorded when DEBUG is True
>>> reset_queries()
>>> ContentType.objects.clear_cache()
>>> ContentType.objects.get_for_model(Post)
<ContentType: Blog | post>
>>> len(connection.queries)
1
>>> ContentType.objects.get_for_model(Post)
<ContentType: Blog | post>
>>> len(connection.queries)
1                                  # still 1 — came from the cache
```

Rules:

| Rule                                                                     | Why                                                                |
|--------------------------------------------------------------------------|--------------------------------------------------------------------|
| Prefer `get_for_model()` / `get_for_id()` / `get_by_natural_key()`       | They are cached; `.get()` / `.filter()` always query               |
| The cache lives as long as the process (server worker)                   | Content types almost never change, so this is safe                 |
| Each worker process has its own cache                                    | No shared cache server needed                                      |
| Call `ContentType.objects.clear_cache()` after changing rows by hand     | Otherwise old values stay in memory                                |
| `get_for_model()` **creates** the row if it is missing                   | Uses `get_or_create`; so it can do an `INSERT` the first time      |

---

## Step 8: Proxy Models and `for_concrete_model`

A **proxy model** shares the table of its parent but has its own Python class (and its own content type row).

Add a proxy to `blog/models.py` for this step (you can keep it or delete it later):

```python
class PublishedPost(Post):
    class Meta:
        proxy = True
        verbose_name = 'published post'
```

```bash
python manage.py makemigrations blog
python manage.py migrate
```

Now in the shell:

```python
>>> from blog.models import Post, PublishedPost
>>> ContentType.objects.get_for_model(PublishedPost)
<ContentType: Blog | post>                      # default: the concrete parent's row!

>>> ContentType.objects.get_for_model(PublishedPost, for_concrete_model=False)
<ContentType: Blog | published post>            # the proxy's own row

>>> ContentType.objects.get_for_models(PublishedPost, for_concrete_models=False)
{<class 'blog.models.PublishedPost'>: <ContentType: Blog | published post>}
```

| Where you see `for_concrete_model`              | Default | Meaning when `True`                               |
|-------------------------------------------------|---------|---------------------------------------------------|
| `ContentType.objects.get_for_model()`           | `True`  | Proxy → parent's content type                     |
| `ContentType.objects.get_for_models()` (plural `for_concrete_models`) | `True` | Same, for many models          |
| `GenericForeignKey(for_concrete_model=...)`     | `True`  | Saving a proxy instance stores the parent's type  |
| `GenericRelation(for_concrete_model=...)`       | `True`  | Reverse lookup uses the parent's type             |
| `generic_inlineformset_factory(for_concrete_model=...)` | `True` | Same for formsets                         |

> **Keep both sides consistent.** If a `GenericForeignKey` uses `for_concrete_model=False`, the matching `GenericRelation` must use `for_concrete_model=False` too, or reverse lookups find nothing.

**Permissions and proxies:** since Django 2.2, permissions for a proxy model (`blog.add_publishedpost`, …) use the **proxy's own** content type.

---

## Step 9: See How Auth Permissions Use Content Types

The `auth_permission` table:

| Column            | Meaning                                       |
|-------------------|-----------------------------------------------|
| `id`              | PK                                            |
| `name`            | `"Can add post"`                              |
| `content_type_id` | FK → `django_content_type.id` (`on_delete=CASCADE`) |
| `codename`        | `"add_post"`                                  |
|                   | unique together: `(content_type, codename)`   |

So the string `'blog.add_post'` you pass to `has_perm()` is really **`content_type.app_label` + `"."` + `codename`**.

```python
>>> from django.contrib.auth.models import Permission
>>> ct = ContentType.objects.get_for_model(Post)
>>> Permission.objects.filter(content_type=ct).values_list('codename', flat=True)
<QuerySet ['add_post', 'change_post', 'delete_post', 'feature_post', 'publish_post', 'view_post']>

>>> p = Permission.objects.get(codename='publish_post')
>>> p.content_type
<ContentType: Blog | post>
>>> f"{p.content_type.app_label}.{p.codename}"
'blog.publish_post'
```

Create a permission **without a model** (e.g. "can view reports") by attaching it to a content type of your choice:

```python
>>> ct = ContentType.objects.get_for_model(Post)
>>> Permission.objects.create(codename='view_reports', name='Can view reports', content_type=ct)
<Permission: Blog | post | Can view reports>
>>> # now checkable as 'blog.view_reports'
```

Give a user every permission for one model:

```python
>>> user = User.objects.get(username='alice')
>>> user.user_permissions.add(*Permission.objects.filter(content_type=ct))
```

> **Cascade warning:** because `content_type` is `on_delete=CASCADE`, deleting a content type row **deletes its permissions** — and removes them from every user and group. This is why `remove_stale_contenttypes` asks for confirmation (Step 29).

---

## Step 10: See How the Admin Uses Content Types

### 10.1 History (`LogEntry`)

Every add/change/delete in the admin creates a `django.contrib.admin.models.LogEntry`:

| Field            | Meaning                                                     |
|------------------|-------------------------------------------------------------|
| `user`           | Who did it                                                  |
| `content_type`   | FK → `ContentType` (`null=True`, `on_delete=SET_NULL`)      |
| `object_id`      | `TextField` — the PK as text (works for int **and** UUID)   |
| `object_repr`    | `str(obj)` at the time                                      |
| `action_flag`    | 1 = addition, 2 = change, 3 = deletion                      |
| `change_message` | JSON describing what changed                                |
| `action_time`    | When                                                        |

This is a hand-made generic relation (no `GenericForeignKey` field, just `content_type` + `object_id`):

```python
>>> from django.contrib.admin.models import LogEntry
>>> entry = LogEntry.objects.first()
>>> entry.content_type
<ContentType: Blog | post>
>>> entry.object_id
'1'
>>> entry.get_edited_object()             # uses content_type.get_object_for_this_type(pk=object_id)
<Post: Hello Django>
>>> entry.get_admin_url()
'/admin/blog/post/1/change/'

>>> # all admin history for posts:
>>> LogEntry.objects.filter(content_type=ContentType.objects.get_for_model(Post))
```

### 10.2 "View on site" button

When a model has `get_absolute_url()` (your `Post` does), the admin change page shows **View on site**. The link looks like:

```
/admin/r/7/1/
         │ └── object id
         └──── content type id of blog.post
```

That URL is handled by `django.contrib.contenttypes.views.shortcut` (details in Step 28).

### 10.3 (Optional) Show content types in the admin, read-only

Good for learning; never let anyone edit these rows. In `blog/admin.py`:

```python
from django.contrib import admin
from django.contrib.contenttypes.models import ContentType


@admin.register(ContentType)
class ContentTypeAdmin(admin.ModelAdmin):
    list_display = ('id', 'app_label', 'model', 'name', 'model_exists')
    list_filter = ('app_label',)
    search_fields = ('app_label', 'model')
    ordering = ('app_label', 'model')

    @admin.display(boolean=True, description='Model exists')
    def model_exists(self, obj):
        return obj.model_class() is not None

    def has_add_permission(self, request):
        return False

    def has_change_permission(self, request, obj=None):
        return False

    def has_delete_permission(self, request, obj=None):
        return False
```

Visit `/admin/contenttypes/contenttype/`.

---

## Step 11: Understand the Problem Generic Relations Solve

You want **comments** on posts **and** on photos (and maybe later on videos, users…).

### ❌ Approach 1 — one comment model per target

```python
class PostComment(models.Model):
    post = models.ForeignKey(Post, on_delete=models.CASCADE)
    text = models.TextField()

class PhotoComment(models.Model):
    photo = models.ForeignKey(Photo, on_delete=models.CASCADE)
    text = models.TextField()
```

Duplicate code, duplicate views, duplicate templates. Every new target = another model.

### ❌ Approach 2 — many nullable foreign keys

```python
class Comment(models.Model):
    post = models.ForeignKey(Post, null=True, blank=True, on_delete=models.CASCADE)
    photo = models.ForeignKey(Photo, null=True, blank=True, on_delete=models.CASCADE)
    # video = ..., user = ..., one more column per target
    text = models.TextField()
```

Mostly-empty columns, needs a check constraint "exactly one is set", needs a migration for every new target.

### ✅ Approach 3 — a generic foreign key

```python
class Comment(models.Model):
    content_type = models.ForeignKey(ContentType, on_delete=models.CASCADE)  # which table
    object_id = models.PositiveBigIntegerField()                             # which row
    content_object = GenericForeignKey('content_type', 'object_id')          # the object itself
    text = models.TextField()
```

One model, one table, works for **any** model, today and in the future.

### Trade-offs (be honest about them)

| ✅ Pros                                           | ❌ Cons                                                                  |
|---------------------------------------------------|--------------------------------------------------------------------------|
| One model/view/template for many targets          | **No real database foreign key** on `object_id` → no DB-level integrity |
| New targets need no migration on `Comment`        | Can't `filter(content_object=...)` or `select_related('content_object')` |
| Used by many popular reusable apps                | Orphans possible if a target has no `GenericRelation` (Step 17)          |
| Easy "activity feed", "tags", "likes" features    | Joins are a bit slower; you must add an index yourself                   |

> **Rule of thumb:** if you know the targets and there are only one or two, use normal `ForeignKey`s. Use generic relations when targets are **many**, **open-ended**, or the feature is **reusable** across projects.

---

## Step 12: Create an App for the Generic Models

Keep generic "social" features in their own app so any model can use them:

```bash
python manage.py startapp social
```

Register it in `config/settings.py`:

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'accounts',
    'blog',
    'social',  # 👈 add this
]
```

Add a second target model so we can prove "generic" works. In `blog/models.py`, below `Post`:

```python
class Photo(models.Model):
    title = models.CharField(max_length=200)
    image_url = models.URLField()          # URLField keeps it simple (no Pillow needed)
    uploaded_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ['-uploaded_at']

    def __str__(self):
        return self.title
```

---

## Step 13: Build Your First Generic Model (`Comment`)

Edit `social/models.py`:

```python
from django.conf import settings
from django.contrib.contenttypes.fields import GenericForeignKey
from django.contrib.contenttypes.models import ContentType
from django.db import models


class Comment(models.Model):
    # 1) WHICH TABLE — a normal ForeignKey to ContentType
    content_type = models.ForeignKey(ContentType, on_delete=models.CASCADE)
    # 2) WHICH ROW — the primary key of the target object
    object_id = models.PositiveBigIntegerField()
    # 3) THE OBJECT — not a database column; combines 1) and 2)
    content_object = GenericForeignKey('content_type', 'object_id')

    author = models.ForeignKey(
        settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name='comments'
    )
    text = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ['-created_at']
        indexes = [
            models.Index(fields=['content_type', 'object_id']),
        ]

    def __str__(self):
        return f'{self.author} on {self.content_object}: {self.text[:30]}'
```

### 13.1 Every line explained

| Line                                       | Why                                                                                                       |
|--------------------------------------------|-----------------------------------------------------------------------------------------------------------|
| `content_type = ForeignKey(ContentType, …)`| A **real** FK column `content_type_id`. Name it `content_type` so you can use the defaults.               |
| `on_delete=models.CASCADE`                 | If the content type row is deleted (stale model removed), its comments go too. Use `PROTECT` to block that. |
| `object_id = PositiveBigIntegerField()`    | Stores the target's PK. **Must match the type of the targets' primary keys** (see 13.2).                  |
| `content_object = GenericForeignKey(...)`  | A Python-only descriptor. Reading it loads the object; assigning it fills the two fields above.          |
| `GenericForeignKey('content_type', 'object_id')` | Arguments are the **names** of the two fields. These are the defaults, so `GenericForeignKey()` alone also works. |
| `Index(fields=['content_type', 'object_id'])` | Django does **not** index this pair automatically. Every lookup ("comments for this post") uses both columns, so add it. |

Full `GenericForeignKey` signature:

```python
GenericForeignKey(ct_field='content_type', fk_field='object_id', for_concrete_model=True)
```

### 13.2 Choosing the type of `object_id`

| Target primary keys are…                                      | Use for `object_id`                     |
|---------------------------------------------------------------|-----------------------------------------|
| `BigAutoField` (default for new projects: `DEFAULT_AUTO_FIELD = 'django.db.models.BigAutoField'`) | `PositiveBigIntegerField()` |
| `AutoField` (older projects)                                  | `PositiveIntegerField()` (or Big — safe either way) |
| `UUIDField`                                                   | `UUIDField()`                           |
| Mixed (some int, some UUID, some string)                      | `CharField(max_length=255)` or `TextField()` — like admin `LogEntry` |

> ⚠️ With a text `object_id` and **integer** target PKs, `GenericRelation` reverse queries compare text to integer. SQLite tolerates it; **PostgreSQL raises an error** (`operator does not exist: character varying = integer`). Prefer matching types; use text only when you really have mixed PKs and you don't need `GenericRelation` joins.

Check your setting:

```python
# config/settings.py
DEFAULT_AUTO_FIELD = 'django.db.models.BigAutoField'
```

### 13.3 Using non-default field names

You can name the two fields anything — then pass the names:

```python
class Comment(models.Model):
    target_type = models.ForeignKey(ContentType, on_delete=models.CASCADE)
    target_id = models.PositiveBigIntegerField()
    target = GenericForeignKey('target_type', 'target_id')
```

…and every `GenericRelation`, admin inline and formset pointing at it must say so too (`content_type_field='target_type', object_id_field='target_id'`, `ct_field=`, `ct_fk_field=`). The defaults save you a lot of typing.

### 13.4 Two generic foreign keys on one model

Allowed — give each its own pair of fields:

```python
class Activity(models.Model):
    actor_type = models.ForeignKey(ContentType, on_delete=models.CASCADE, related_name='+')
    actor_id = models.PositiveBigIntegerField()
    actor = GenericForeignKey('actor_type', 'actor_id')

    target_type = models.ForeignKey(ContentType, on_delete=models.CASCADE, related_name='+')
    target_id = models.PositiveBigIntegerField()
    target = GenericForeignKey('target_type', 'target_id')
```

`related_name='+'` avoids the reverse-accessor clash on `ContentType` (two FKs to the same model from one model need distinct or disabled reverse names).

---

## Step 14: Migrate and Create Generic Objects in the Shell

```bash
python manage.py makemigrations blog social
python manage.py migrate
```

The `social_comment` table has columns `id`, `content_type_id`, `object_id`, `author_id`, `text`, `created_at` — **no `content_object` column** (it is not a database field). Check:

```bash
python manage.py sqlmigrate social 0001
```

Content type rows for `blog.photo` and `social.comment` now exist (created by `post_migrate`, Step 4).

### 14.1 Create test data

```bash
python manage.py shell
```

```python
>>> from accounts.models import User
>>> from blog.models import Post, Photo
>>> from social.models import Comment
>>> from django.contrib.contenttypes.models import ContentType

>>> alice = User.objects.get(username='alice')        # use any existing user
>>> post = Post.objects.first()
>>> photo = Photo.objects.create(title='Sunset', image_url='https://example.com/sunset.jpg')
```

### 14.2 Way 1 — assign `content_object` (recommended)

```python
>>> c1 = Comment.objects.create(content_object=post, author=alice, text='Great post!')
>>> c2 = Comment.objects.create(content_object=photo, author=alice, text='Beautiful!')
```

Django fills `content_type` and `object_id` for you:

```python
>>> c1.content_type, c1.object_id
(<ContentType: Blog | post>, 1)
>>> c2.content_type, c2.object_id
(<ContentType: Blog | photo>, 1)
```

### 14.3 Way 2 — set the two fields yourself

```python
>>> c3 = Comment.objects.create(
...     content_type=ContentType.objects.get_for_model(Post),
...     object_id=post.pk,
...     author=alice,
...     text='Set by hand',
... )
```

Useful when you only have IDs (e.g. from a URL or an API) and don't want to load the object.

### 14.4 Way 3 — build then save

```python
>>> c4 = Comment(author=alice, text='Built then saved')
>>> c4.content_object = photo
>>> c4.save()
```

### 14.5 Read the object back

```python
>>> c = Comment.objects.get(pk=c1.pk)
>>> c.content_object                 # 1 query for the Post (content type comes from cache)
<Post: Hello Django>
>>> c.content_object                 # cached on the instance — no new query
<Post: Hello Django>
>>> type(c.content_object)
<class 'blog.models.Post'>
```

### 14.6 Things that happen at the edges

```python
>>> # Assigning an UNSAVED object:
>>> Comment(content_object=Photo(title='x', image_url='https://e.com/x.jpg'), author=alice, text='?').save()
ValueError: save() prohibited to prevent data loss due to unsaved related object 'content_object'.

>>> # Target deleted but comment kept (no GenericRelation yet — see Step 17):
>>> photo2 = Photo.objects.create(title='Temp', image_url='https://e.com/t.jpg')
>>> c5 = Comment.objects.create(content_object=photo2, author=alice, text='orphan soon')
>>> photo2.delete()
>>> Comment.objects.get(pk=c5.pk).content_object
None                                  # the GFK returns None when the row is missing
```

---

## Step 15: Add the Reverse Side With `GenericRelation`

Without help, going from a post to its comments needs both values:

```python
>>> Comment.objects.filter(
...     content_type=ContentType.objects.get_for_model(post),
...     object_id=post.pk,
... )
```

`GenericRelation` gives you a normal-looking reverse manager. Edit `blog/models.py`:

```python
from django.conf import settings
from django.contrib.contenttypes.fields import GenericRelation
from django.db import models
from django.urls import reverse


class Post(models.Model):
    # ... all the existing fields (title, slug, author, content, status, created_at, updated_at)

    comments = GenericRelation('social.Comment', related_query_name='post')

    # ... existing Meta, __str__, get_absolute_url


class Photo(models.Model):
    title = models.CharField(max_length=200)
    image_url = models.URLField()
    uploaded_at = models.DateTimeField(auto_now_add=True)

    comments = GenericRelation('social.Comment', related_query_name='photo')

    class Meta:
        ordering = ['-uploaded_at']

    def __str__(self):
        return self.title
```

Full signature:

```python
GenericRelation(
    to,                                   # the model that has the GenericForeignKey ('social.Comment')
    object_id_field='object_id',          # name of its object id field
    content_type_field='content_type',    # name of its content type field
    for_concrete_model=True,              # match the GenericForeignKey's setting
    related_query_name=None,              # lets you query FROM Comment back TO Post
)
```

Notes:

- Use the **string** `'social.Comment'` to avoid circular imports.
- `GenericRelation` adds **no database column**; `makemigrations` may say "No changes detected" for `blog`. That is normal.
- Add a `GenericRelation` to **every** model that can be a target. It enables reverse access, reverse filtering, aggregation and **cascade delete** (Step 17).

### 15.1 Use it

```python
>>> post = Post.objects.first()
>>> post.comments.all()
<QuerySet [<Comment: alice on Hello Django: Set by hand>, <Comment: alice on Hello Django: Great post!>]>

>>> post.comments.count()
2

>>> post.comments.create(author=alice, text='Created through the relation')   # sets content_type/object_id for you
>>> post.comments.filter(author=alice).exists()
True
```

All related-manager methods work:

| Method                                   | Notes                                                                                 |
|------------------------------------------|---------------------------------------------------------------------------------------|
| `.all()`, `.filter()`, `.exclude()`, `.count()` | As usual                                                                       |
| `.create(**kwargs)`                      | Fills `content_type` and `object_id`                                                  |
| `.get_or_create()`, `.update_or_create()`| Same                                                                                  |
| `.add(obj, bulk=True)`                   | Re-points an existing comment at this object                                          |
| `.remove(obj)`, `.clear()`               | **Deletes** the comments (the fields are not nullable, so there is nothing to "unset") |
| `.set([objs])`                           | Replace the set                                                                       |

---

## Step 16: Query Generic Relations — What Works and What Doesn't

### 16.1 ❌ You cannot filter on the `GenericForeignKey` itself

```python
>>> Comment.objects.filter(content_object=post)
django.core.exceptions.FieldError: Field 'content_object' does not generate an automatic reverse relation and therefore cannot be used for reverse querying. If it is a GenericForeignKey, consider adding a GenericRelation.
```

Also not possible: `.exclude(content_object=...)`, `.order_by('content_object')`, `.values('content_object')`, `.select_related('content_object')`.

### 16.2 ✅ Filter on the two real fields

```python
>>> ct = ContentType.objects.get_for_model(post)
>>> Comment.objects.filter(content_type=ct, object_id=post.pk)

>>> # All comments on any Photo:
>>> Comment.objects.filter(content_type=ContentType.objects.get_for_model(Photo))

>>> # Comments on posts OR photos:
>>> Comment.objects.filter(content_type__in=ContentType.objects.get_for_models(Post, Photo).values())

>>> # By strings (joins django_content_type):
>>> Comment.objects.filter(content_type__app_label='blog', content_type__model='post')
```

### 16.3 ✅ Filter through `related_query_name`

Because we set `related_query_name='post'`, you can query **from** `Comment` **into** `Post` fields:

```python
>>> Comment.objects.filter(post__status='published')
>>> Comment.objects.filter(post__title__icontains='django')
>>> Comment.objects.filter(photo__title='Sunset')
```

### 16.4 ✅ Filter targets by their generic children

Through the `GenericRelation` on `Post`:

```python
>>> Post.objects.filter(comments__author=alice).distinct()      # posts alice commented on
>>> Post.objects.filter(comments__isnull=False).distinct()      # posts with at least one comment
>>> Post.objects.filter(comments__isnull=True)                  # posts with no comments
>>> Post.objects.exclude(comments__text__icontains='spam')
```

### 16.5 Group comments by target type

```python
>>> from django.db.models import Count
>>> Comment.objects.values('content_type__app_label', 'content_type__model').annotate(total=Count('id'))
<QuerySet [{'content_type__app_label': 'blog', 'content_type__model': 'photo', 'total': 1}, {'content_type__app_label': 'blog', 'content_type__model': 'post', 'total': 4}]>
```

---

## Step 17: Deleting — Cascades and Orphans

The `object_id` column has **no database foreign key**, so the database cannot cascade. Django does it in Python — **only through a `GenericRelation`**.

| Situation                                                   | What happens to comments                         |
|-------------------------------------------------------------|--------------------------------------------------|
| Target model **has** `GenericRelation` and you `post.delete()` | ✅ Its comments are deleted too (Django collects them) |
| Target model has **no** `GenericRelation`                   | ❌ Comments stay — **orphans** (`content_object` is `None`) |
| `Post.objects.filter(...).delete()` (QuerySet delete)       | ✅ Also cascades through `GenericRelation`        |
| Raw SQL `DELETE` / deleting in the DB tool                  | ❌ Orphans — Django is bypassed                   |
| The **content type row** is deleted                         | ✅ Comments deleted via the real FK (`CASCADE`)   |

Prove it:

```python
>>> p = Post.objects.create(title='Delete me', slug='delete-me', content='...', author=alice)
>>> p.comments.create(author=alice, text='will vanish')
>>> p.delete()
(2, {'social.Comment': 1, 'blog.Post': 1})       # the comment is in the count
```

The admin's delete confirmation page also lists the generic objects that will be deleted.

### 17.1 Find and clean orphans

```python
>>> from django.contrib.contenttypes.models import ContentType
>>> def orphan_comments():
...     orphans = []
...     for ct_id in Comment.objects.values_list('content_type', flat=True).distinct():
...         ct = ContentType.objects.get_for_id(ct_id)
...         model = ct.model_class()
...         ids = Comment.objects.filter(content_type=ct).values_list('object_id', flat=True)
...         if model is None:                                     # model removed from code
...             orphans += list(Comment.objects.filter(content_type=ct).values_list('pk', flat=True))
...             continue
...         existing = set(model._base_manager.filter(pk__in=ids).values_list('pk', flat=True))
...         orphans += list(
...             Comment.objects.filter(content_type=ct).exclude(object_id__in=existing).values_list('pk', flat=True)
...         )
...     return orphans
...
>>> Comment.objects.filter(pk__in=orphan_comments()).delete()
```

### 17.2 Don't want cascades?

There is no `on_delete` option on `GenericRelation`. If you want to keep generic children after the target is deleted, **don't** add a `GenericRelation` (and accept orphans), or handle `pre_delete` yourself.

---

## Step 18: Restrict Which Models Can Be Targets

By default a `Comment` could point to **anything** — including `accounts.User` or `auth.Permission`. Limit the choices with `limit_choices_to` on the content type FK:

```python
from django.db.models import Q


class Comment(models.Model):
    content_type = models.ForeignKey(
        ContentType,
        on_delete=models.CASCADE,
        limit_choices_to=Q(app_label='blog', model__in=['post', 'photo']),
    )
    # ...
```

What this does and doesn't do:

| Does                                                          | Doesn't                                                        |
|---------------------------------------------------------------|----------------------------------------------------------------|
| Limits the dropdown in the admin and in `ModelForm`s          | Stop `Comment.objects.create(content_object=user, ...)` in code |
| Makes model forms reject other types during validation        | Add a database constraint                                      |

For a hard rule, validate in `clean()`:

```python
from django.core.exceptions import ValidationError

ALLOWED_TARGETS = {('blog', 'post'), ('blog', 'photo')}


class Comment(models.Model):
    # ... fields ...

    def clean(self):
        super().clean()
        if self.content_type_id:
            key = (self.content_type.app_label, self.content_type.model)
            if key not in ALLOWED_TARGETS:
                raise ValidationError({'content_type': 'You cannot comment on this kind of object.'})
            # the target row must exist
            model = self.content_type.model_class()
            if model is None or not model._base_manager.filter(pk=self.object_id).exists():
                raise ValidationError({'object_id': 'That object does not exist.'})
```

> `clean()` runs in `ModelForm`s and in the admin, **not** on `.save()`. Call `comment.full_clean()` before `save()` in your own code paths.

Run `makemigrations social` + `migrate` after changing `limit_choices_to` (it is stored in the migration).

---

## Step 19: Performance — Avoid N+1 Queries

### 19.1 The problem

```python
>>> for c in Comment.objects.all():        # 1 query
...     print(c.content_object)            # +1 query PER comment  → N+1
```

`select_related('content_object')` is **not** possible (different tables).

### 19.2 `prefetch_related` works on a `GenericForeignKey`

```python
>>> comments = Comment.objects.prefetch_related('content_object')
>>> for c in comments:
...     print(c.content_object)
```

Queries: 1 for comments + **1 per content type** (one for all posts, one for all photos). Two target types → 3 queries total, no matter how many comments.

Also prefetch normal FKs at the same time:

```python
>>> Comment.objects.select_related('author', 'content_type').prefetch_related('content_object')
```

### 19.3 `GenericPrefetch` — custom querysets per model (Django 5.0+)

```python
from django.contrib.contenttypes.prefetch import GenericPrefetch

comments = Comment.objects.prefetch_related(
    GenericPrefetch(
        'content_object',
        [
            Post.objects.select_related('author'),        # how to load Posts
            Photo.objects.all(),                          # how to load Photos
        ],
    )
)
```

Signature: `GenericPrefetch(lookup, querysets, to_attr=None)`. You may give **at most one queryset per model**.

### 19.4 Prefetch a `GenericRelation` (the reverse side)

```python
>>> posts = Post.objects.prefetch_related('comments')                     # 2 queries total
>>> posts = Post.objects.prefetch_related('comments__author')             # 3 queries total

>>> from django.db.models import Prefetch
>>> posts = Post.objects.prefetch_related(
...     Prefetch('comments', queryset=Comment.objects.select_related('author'))
... )                                                                     # 2 queries, authors joined in
```

### 19.5 Index checklist

- ✅ `Index(fields=['content_type', 'object_id'])` on every generic model (Step 13).
- ✅ Content type lookups via `get_for_model()` (cached).
- ✅ `prefetch_related` in list views.

---

## Step 20: Aggregation and Annotation

Through `GenericRelation` you can count and aggregate like any reverse FK:

```python
>>> from django.db.models import Count, Max

>>> posts = Post.objects.annotate(num_comments=Count('comments'))
>>> [(p.title, p.num_comments) for p in posts]
[('Hello Django', 4), ('Second post', 0)]

>>> Post.objects.annotate(last_comment=Max('comments__created_at'))

>>> Post.objects.annotate(n=Count('comments')).order_by('-n')[:5]   # most discussed posts

>>> Post.objects.aggregate(total=Count('comments'))
{'total': 4}
```

> Two different `Count`s over two different relations in one `annotate()` multiply rows. Use `Count('comments', distinct=True)` and `Count('likes', distinct=True)` together (Step 21).

---

## Step 21: Second Example — Likes (One Per User Per Object)

Add to `social/models.py`:

```python
class Like(models.Model):
    content_type = models.ForeignKey(ContentType, on_delete=models.CASCADE)
    object_id = models.PositiveBigIntegerField()
    content_object = GenericForeignKey()            # defaults: 'content_type', 'object_id'

    user = models.ForeignKey(
        settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name='likes'
    )
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        constraints = [
            models.UniqueConstraint(
                fields=['user', 'content_type', 'object_id'],
                name='unique_like_per_user_per_object',
            ),
        ]
        indexes = [
            models.Index(fields=['content_type', 'object_id']),
        ]

    def __str__(self):
        return f'{self.user} likes {self.content_object}'
```

> You can put the **real** fields (`content_type`, `object_id`) in `UniqueConstraint`, `unique_together` and `Index` — never the `GenericForeignKey` name.

Add the reverse relation in `blog/models.py`:

```python
class Post(models.Model):
    # ...
    comments = GenericRelation('social.Comment', related_query_name='post')
    likes = GenericRelation('social.Like', related_query_name='post')


class Photo(models.Model):
    # ...
    comments = GenericRelation('social.Comment', related_query_name='photo')
    likes = GenericRelation('social.Like', related_query_name='photo')
```

```bash
python manage.py makemigrations social
python manage.py migrate
```

A reusable toggle helper in `social/services.py`:

```python
from django.contrib.contenttypes.models import ContentType

from .models import Like


def toggle_like(user, obj):
    """Like obj if not liked yet, otherwise unlike. Returns True if now liked."""
    ct = ContentType.objects.get_for_model(obj)
    like, created = Like.objects.get_or_create(user=user, content_type=ct, object_id=obj.pk)
    if not created:
        like.delete()
    return created


def is_liked(user, obj):
    if not user.is_authenticated:
        return False
    ct = ContentType.objects.get_for_model(obj)
    return Like.objects.filter(user=user, content_type=ct, object_id=obj.pk).exists()
```

Try it:

```python
>>> from social.services import toggle_like
>>> toggle_like(alice, post)
True
>>> toggle_like(alice, post)
False
>>> toggle_like(alice, photo)
True
>>> Post.objects.annotate(
...     n_comments=Count('comments', distinct=True),
...     n_likes=Count('likes', distinct=True),
... ).values('title', 'n_comments', 'n_likes')
```

---

## Step 22: Third Example — Tags on Anything

Add to `social/models.py`:

```python
class Tag(models.Model):
    name = models.SlugField(max_length=50, unique=True)

    def __str__(self):
        return self.name


class TaggedItem(models.Model):
    tag = models.ForeignKey(Tag, on_delete=models.CASCADE, related_name='items')
    content_type = models.ForeignKey(ContentType, on_delete=models.CASCADE)
    object_id = models.PositiveBigIntegerField()
    content_object = GenericForeignKey()

    class Meta:
        constraints = [
            models.UniqueConstraint(
                fields=['tag', 'content_type', 'object_id'], name='unique_tag_per_object'
            ),
        ]
        indexes = [models.Index(fields=['content_type', 'object_id'])]

    def __str__(self):
        return f'{self.tag} → {self.content_object}'
```

In `blog/models.py`:

```python
class Post(models.Model):
    # ...
    tagged_items = GenericRelation('social.TaggedItem', related_query_name='post')

    @property
    def tags(self):
        from social.models import Tag
        return Tag.objects.filter(items__in=self.tagged_items.all())
```

(Add the same `tagged_items` line to `Photo` with `related_query_name='photo'`.)

```bash
python manage.py makemigrations social
python manage.py migrate
```

Use it:

```python
>>> from social.models import Tag, TaggedItem
>>> django_tag, _ = Tag.objects.get_or_create(name='django')
>>> TaggedItem.objects.create(tag=django_tag, content_object=post)
>>> TaggedItem.objects.create(tag=django_tag, content_object=photo)

>>> post.tags
<QuerySet [<Tag: django>]>

>>> Post.objects.filter(tagged_items__tag__name='django')          # posts tagged "django"
>>> TaggedItem.objects.filter(tag__name='django').prefetch_related('content_object')   # everything tagged "django"
>>> [ti.content_object for ti in django_tag.items.prefetch_related('content_object')]
[<Post: Hello Django>, <Photo: Sunset>]
```

> In real projects, **django-taggit** does exactly this with content types — now you know how it works inside.

---

## Step 23: Generic Relations in the Admin

### 23.1 Register the generic models

`social/admin.py`:

```python
from django.contrib import admin

from .models import Comment, Like, Tag, TaggedItem


@admin.register(Comment)
class CommentAdmin(admin.ModelAdmin):
    list_display = ('id', 'author', 'content_type', 'object_id', 'target', 'short_text', 'created_at')
    list_filter = ('content_type', 'created_at')
    search_fields = ('text', 'author__username')
    list_select_related = ('author', 'content_type')
    raw_id_fields = ('author',)

    @admin.display(description='Target')
    def target(self, obj):
        return obj.content_object

    @admin.display(description='Text')
    def short_text(self, obj):
        return obj.text[:50]

    def get_queryset(self, request):
        return super().get_queryset(request).prefetch_related('content_object')


admin.site.register(Like)
admin.site.register(Tag)
admin.site.register(TaggedItem)
```

Notes:

- `content_object` **cannot** be in `fields`, `list_display` directly as a sortable column, `list_filter` or `search_fields` — use a method (`target` above) and filter on `content_type`.
- The add form shows a **content type dropdown** + an **object id** number box. Not friendly, which is why inlines (below) are better.

### 23.2 Edit comments inside the Post page (generic inline)

`blog/admin.py`:

```python
from django.contrib import admin
from django.contrib.contenttypes.admin import GenericStackedInline, GenericTabularInline

from social.models import Comment, TaggedItem
from .models import Photo, Post


class CommentInline(GenericTabularInline):
    model = Comment
    extra = 1
    fields = ('author', 'text', 'created_at')
    readonly_fields = ('created_at',)
    raw_id_fields = ('author',)
    # ct_field = 'content_type'       # defaults — set only if you renamed the fields
    # ct_fk_field = 'object_id'


class TaggedItemInline(GenericTabularInline):
    model = TaggedItem
    extra = 1


@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    # ... your existing PostAdmin options ...
    inlines = [CommentInline, TaggedItemInline]


@admin.register(Photo)
class PhotoAdmin(admin.ModelAdmin):
    list_display = ('title', 'uploaded_at')
    inlines = [CommentInline, TaggedItemInline]
```

(If `Post` was registered before with `admin.site.register(Post)` or another `@admin.register(Post)`, merge into that one class — registering twice raises `AlreadyRegistered`.)

| Inline class              | Looks like                                    |
|---------------------------|-----------------------------------------------|
| `GenericTabularInline`    | Compact table, one row per object             |
| `GenericStackedInline`    | Full form per object, stacked vertically      |
| `GenericInlineModelAdmin` | Base class; set `template` yourself           |

Extra options on generic inlines (on top of all normal `InlineModelAdmin` options: `extra`, `max_num`, `min_num`, `fields`, `readonly_fields`, `can_delete`, `show_change_link`, `classes`, …):

| Option         | Default          | Meaning                                    |
|----------------|------------------|--------------------------------------------|
| `ct_field`     | `'content_type'` | Name of the content type FK on the inline model |
| `ct_fk_field`  | `'object_id'`    | Name of the object id field                 |
| `formset`      | `BaseGenericInlineFormSet` | Formset class                       |

The inline automatically hides `content_type` and `object_id` and fills them with the parent object.

---

## Step 24: Generic Relations in Your Own Forms

### 24.1 A simple comment form

`social/forms.py`:

```python
from django import forms

from .models import Comment


class CommentForm(forms.ModelForm):
    class Meta:
        model = Comment
        fields = ['text']                    # content_type/object_id/author set in the view
        widgets = {'text': forms.Textarea(attrs={'rows': 3})}
```

### 24.2 A generic inline formset (edit many comments at once)

```python
from django.contrib.contenttypes.forms import generic_inlineformset_factory

from .models import Comment

CommentFormSet = generic_inlineformset_factory(
    Comment,
    fields=['text'],
    extra=1,
    can_delete=True,
    # ct_field='content_type', fk_field='object_id',  # defaults
)
```

Full signature (same as `inlineformset_factory` plus the generic bits):

```python
generic_inlineformset_factory(
    model, form=ModelForm, formset=BaseGenericInlineFormSet,
    ct_field='content_type', fk_field='object_id',
    fields=None, exclude=None, extra=3, can_order=False, can_delete=True,
    max_num=None, formfield_callback=None, validate_max=False,
    for_concrete_model=True, min_num=None, validate_min=False,
    absolute_max=None, can_delete_extra=True,
)
```

Use it in a view:

```python
from django.contrib.auth.decorators import login_required
from django.shortcuts import get_object_or_404, redirect, render

from blog.models import Post
from social.forms import CommentFormSet


@login_required
def edit_post_comments(request, pk):
    post = get_object_or_404(Post, pk=pk)
    formset = CommentFormSet(request.POST or None, instance=post)   # instance = the TARGET object
    if request.method == 'POST' and formset.is_valid():
        comments = formset.save(commit=False)
        for comment in comments:
            comment.author = request.user
            comment.save()
        for obj in formset.deleted_objects:
            obj.delete()
        return redirect(post)
    return render(request, 'social/edit_comments.html', {'post': post, 'formset': formset})
```

Template `social/templates/social/edit_comments.html`:

```html
{% extends 'blog/base.html' %}

{% block content %}
  <h2>Comments on "{{ post.title }}"</h2>
  <form method="post">
    {% csrf_token %}
    {{ formset.management_form }}
    {% for form in formset %}
      {{ form.as_p }}
      <hr>
    {% endfor %}
    <button type="submit">Save</button>
  </form>
{% endblock %}
```

> Forget `{{ formset.management_form }}` and you get "ManagementForm data is missing or has been tampered with".

---

## Step 25: Views and URLs — "Comment on Anything"

One view that works for posts, photos and any future model. The URL carries `app_label`, `model` and `object_id`.

### 25.1 The view

`social/views.py`:

```python
from django.contrib import messages
from django.contrib.auth.decorators import login_required
from django.contrib.contenttypes.models import ContentType
from django.http import Http404
from django.shortcuts import redirect
from django.views.decorators.http import require_POST

from .forms import CommentForm
from .services import toggle_like

# SECURITY: only these models may be commented on / liked
ALLOWED_TARGETS = {('blog', 'post'), ('blog', 'photo')}


def get_target_or_404(app_label, model, object_id):
    if (app_label, model) not in ALLOWED_TARGETS:
        raise Http404('Unknown target type')
    try:
        ct = ContentType.objects.get_by_natural_key(app_label, model)
    except ContentType.DoesNotExist:
        raise Http404('Unknown target type')
    model_class = ct.model_class()
    if model_class is None:
        raise Http404('Unknown target type')
    try:
        return model_class._default_manager.get(pk=object_id)
    except model_class.DoesNotExist:
        raise Http404('Object not found')


def back_to(request, obj):
    if hasattr(obj, 'get_absolute_url'):
        return redirect(obj)
    return redirect(request.META.get('HTTP_REFERER', '/'))


@login_required
@require_POST
def add_comment(request, app_label, model, object_id):
    target = get_target_or_404(app_label, model, object_id)
    form = CommentForm(request.POST)
    if form.is_valid():
        comment = form.save(commit=False)
        comment.author = request.user
        comment.content_object = target          # sets content_type + object_id
        comment.save()
        messages.success(request, 'Comment added.')
    else:
        messages.error(request, 'Comment cannot be empty.')
    return back_to(request, target)


@login_required
@require_POST
def like(request, app_label, model, object_id):
    target = get_target_or_404(app_label, model, object_id)
    liked = toggle_like(request.user, target)
    messages.info(request, 'Liked!' if liked else 'Like removed.')
    return back_to(request, target)
```

> ⚠️ **Why the whitelist matters:** without `ALLOWED_TARGETS`, a user could POST to `/social/comment/accounts/user/1/` or `/social/comment/auth/permission/5/` and attach comments to private objects, or probe which objects exist. Never trust `app_label`/`model`/`content_type_id` coming from a request.

> Using `content_type_id` in the URL instead of `app_label`/`model` also works (`ContentType.objects.get_for_id(ct_id)`), but IDs differ between databases and are less readable. Same whitelist rule applies.

### 25.2 URLs

`social/urls.py`:

```python
from django.urls import path

from . import views

app_name = 'social'

urlpatterns = [
    path('comment/<slug:app_label>/<slug:model>/<int:object_id>/', views.add_comment, name='add_comment'),
    path('like/<slug:app_label>/<slug:model>/<int:object_id>/', views.like, name='like'),
]
```

`config/urls.py`:

```python
urlpatterns = [
    path('admin/', admin.site.urls),
    path('accounts/', include('django.contrib.auth.urls')),
    path('accounts/', include('accounts.urls')),
    path('social/', include('social.urls')),       # 👈 add this
    path('', include('blog.urls')),
]
```

---

## Step 26: Templates — Reusable Comment and Like Blocks

### 26.1 A template tag that gives you the target's identity

Templates can't call `ContentType.objects.get_for_model(obj)` (and `obj._meta` is blocked because it starts with `_`). Make a tiny tag library.

Create `social/templatetags/__init__.py` (empty) and `social/templatetags/social_tags.py`:

```python
from django import template
from django.contrib.contenttypes.models import ContentType

from social.services import is_liked

register = template.Library()


@register.simple_tag
def target_args(obj):
    """Return 'app_label/model/pk' parts for the generic URLs."""
    ct = ContentType.objects.get_for_model(obj)
    return {'app_label': ct.app_label, 'model': ct.model, 'object_id': obj.pk}


@register.simple_tag(takes_context=True)
def liked_by_me(context, obj):
    return is_liked(context['request'].user, obj)


@register.inclusion_tag('social/_comments.html', takes_context=True)
def comments_block(context, obj):
    ct = ContentType.objects.get_for_model(obj)
    return {
        'request': context['request'],
        'user': context['user'],
        'obj': obj,
        'app_label': ct.app_label,
        'model': ct.model,
        'comments': obj.comments.select_related('author'),
    }
```

Restart `runserver` after creating a new `templatetags` folder.

### 26.2 The comments partial

`social/templates/social/_comments.html`:

```html
<section class="comments">
  <h3>Comments ({{ comments|length }})</h3>

  {% for c in comments %}
    <p><strong>{{ c.author }}</strong> · {{ c.created_at|timesince }} ago<br>{{ c.text|linebreaksbr }}</p>
  {% empty %}
    <p>No comments yet.</p>
  {% endfor %}

  {% if user.is_authenticated %}
    <form method="post" action="{% url 'social:add_comment' app_label model obj.pk %}">
      {% csrf_token %}
      <textarea name="text" rows="3" required></textarea>
      <button type="submit">Add comment</button>
    </form>
  {% else %}
    <p><a href="{% url 'login' %}?next={{ request.path }}">Log in</a> to comment.</p>
  {% endif %}
</section>
```

### 26.3 Use it on the post detail page

In `blog/templates/blog/post_detail.html`, near the bottom of `{% block content %}`:

```html
{% load social_tags %}

{% target_args post as t %}
{% liked_by_me post as liked %}
<form method="post" action="{% url 'social:like' t.app_label t.model t.object_id %}">
  {% csrf_token %}
  <button type="submit">{% if liked %}♥ Unlike{% else %}♡ Like{% endif %}</button>
  {{ post.likes.count }} like{{ post.likes.count|pluralize }}
</form>

{% comments_block post %}
```

The **same two lines** work in a `photo_detail.html` with `photo` instead of `post` — that is the point of generic relations.

> `{% load %}` must come after `{% extends %}` (which must be the first tag) — put `{% load social_tags %}` right below the `extends` line.

`context['request']` requires `django.template.context_processors.request` in `TEMPLATES['OPTIONS']['context_processors']` — it is there by default.

---

## Step 27: Activity Feed With Signals (Real-World Pattern)

Record "alice commented on Hello Django", "bob liked Sunset", … in one table.

`social/models.py`:

```python
class Activity(models.Model):
    VERB_CHOICES = [
        ('commented', 'commented on'),
        ('liked', 'liked'),
        ('created', 'created'),
    ]

    actor = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name='activities')
    verb = models.CharField(max_length=20, choices=VERB_CHOICES)

    target_type = models.ForeignKey(ContentType, on_delete=models.CASCADE, related_name='+')
    target_id = models.PositiveBigIntegerField()
    target = GenericForeignKey('target_type', 'target_id')

    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ['-created_at']
        indexes = [models.Index(fields=['target_type', 'target_id'])]
        verbose_name_plural = 'activities'

    def __str__(self):
        return f'{self.actor} {self.get_verb_display()} {self.target}'
```

`social/signals.py`:

```python
from django.db.models.signals import post_save
from django.dispatch import receiver

from blog.models import Post
from .models import Activity, Comment, Like


@receiver(post_save, sender=Comment)
def comment_created(sender, instance, created, **kwargs):
    if created:
        Activity.objects.create(actor=instance.author, verb='commented', target=instance.content_object)


@receiver(post_save, sender=Like)
def like_created(sender, instance, created, **kwargs):
    if created:
        Activity.objects.create(actor=instance.user, verb='liked', target=instance.content_object)


@receiver(post_save, sender=Post)
def post_created(sender, instance, created, **kwargs):
    if created and instance.author:
        Activity.objects.create(actor=instance.author, verb='created', target=instance)
```

Connect them in `social/apps.py`:

```python
from django.apps import AppConfig


class SocialConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'social'

    def ready(self):
        from . import signals  # noqa: F401
```

Note the GFK uses custom names (`target_type`, `target_id`), so a reverse relation must name them:

```python
# blog/models.py — Post
activities = GenericRelation(
    'social.Activity',
    content_type_field='target_type',
    object_id_field='target_id',
    related_query_name='post',
)
```

```bash
python manage.py makemigrations social
python manage.py migrate
```

Show the feed:

```python
>>> from social.models import Activity
>>> for a in Activity.objects.select_related('actor', 'target_type').prefetch_related('target')[:20]:
...     print(a)
alice liked Sunset
alice commented on Hello Django
```

---

## Step 28: The `shortcut` View ("View on Site")

`django.contrib.contenttypes.views.shortcut(request, content_type_id, object_id)`:

1. Loads the content type by id → 404 if missing.
2. Loads the object → 404 if missing.
3. Calls `obj.get_absolute_url()` → 404 if the model doesn't have it.
4. If the URL already starts with `http://` or `https://` → redirect there.
5. Otherwise, if `django.contrib.sites` is installed → builds `https://<site domain><url>` using the object's related `Site` or the current site.
6. Otherwise → redirects to the relative URL.

The admin already wires it as `admin:view_on_site` (`/admin/r/<int:content_type_id>/<path:object_id>/`). You can expose it on your own site too:

```python
# config/urls.py
from django.contrib.contenttypes.views import shortcut

urlpatterns = [
    # ...
    path('r/<int:content_type_id>/<path:object_id>/', shortcut, name='shortcut'),
]
```

Build a link in a template — first add a small tag to `social/templatetags/social_tags.py`:

```python
@register.simple_tag
def content_type_id(obj):
    return ContentType.objects.get_for_model(obj).id
```

```html
{% load social_tags %}
{% content_type_id post as ct_id %}
<a href="{% url 'shortcut' ct_id post.pk %}">Permalink</a>
```

Control the admin button per model:

```python
class PostAdmin(admin.ModelAdmin):
    view_on_site = True                     # default if the model has get_absolute_url
    # view_on_site = False                  # hide the button
    # def view_on_site(self, obj):          # custom URL
    #     return f'https://example.com/p/{obj.slug}/'
```

---

## Step 29: Remove Stale Content Types

When you **delete a model** or **remove an app** from `INSTALLED_APPS`, its rows stay in `django_content_type` (Step 4.1 — creation never deletes). Those are **stale** content types. They can break things: `model_class()` returns `None`, admin history shows broken links, permissions point to nothing.

### 29.1 Example

Delete the `PublishedPost` proxy from Step 8 in `blog/models.py`, then:

```bash
python manage.py makemigrations blog
python manage.py migrate
python manage.py remove_stale_contenttypes
```

Output:

```
Some content types in your database are stale and can be deleted.
Any objects that depend on these content types will also be deleted.
The content types and dependent objects that would be deleted are:

    - Content type for blog.publishedpost
    - 4 auth.Permission object(s)

This list doesn't include any cascade deletions to data outside of Django's
models (uncommon).

Are you sure you want to delete these content types?
If you're unsure, answer 'no'.
Type 'yes' to continue, or 'no' to cancel: yes
```

### 29.2 Options

| Option                    | Meaning                                                                                       |
|---------------------------|-----------------------------------------------------------------------------------------------|
| *(none)*                  | Removes stale types for apps that are **still installed** (model was deleted)                 |
| `--include-stale-apps`    | Also removes types for apps **no longer in `INSTALLED_APPS`**                                 |
| `--database <alias>`      | Which database (default `default`)                                                            |
| `--noinput` / `--no-input`| Don't ask; delete immediately (for scripts/CI — be careful)                                   |
| `-v 2`                    | Print what was deleted                                                                        |

```bash
python manage.py remove_stale_contenttypes --include-stale-apps
python manage.py remove_stale_contenttypes --noinput -v 2
```

### 29.3 Read the list before typing "yes"

Everything with `on_delete=CASCADE` to `ContentType` goes too: **permissions** (and therefore user/group permission assignments), your **comments, likes, tags** for that model, third-party data (django-guardian object permissions, reversion history…). Admin `LogEntry` uses `SET_NULL`, so history rows stay but lose their type. Back up the database first in production.

### 29.4 Renamed a model? Don't let it go stale

Use `RenameModel` (Step 30.1) instead of delete + create, and the content type row is renamed — no stale row, no lost permissions or comments.

---

## Step 30: Content Types in Migrations

### 30.1 Renaming a model

When you rename `Photo` → `Picture`, `makemigrations` asks:

```
Was the model blog.Photo renamed to Picture? [y/N] y
```

Answer **y**. The migration contains `migrations.RenameModel('Photo', 'Picture')`. Because of the `pre_migrate` hook from Step 2, Django injects a `RenameContentType` step after it, so the row `blog | photo` becomes `blog | picture` and **keeps its id** — comments, likes, permissions all stay attached. (Permission *names* like "Can add photo" are not renamed; update them in a data migration if you care.)

If you answer **N**, Django deletes the old model and creates a new one: the old content type becomes stale and new permissions are created. Existing generic objects still point at the stale type.

### 30.2 Use `ContentType` in a data migration

```bash
python manage.py makemigrations social --empty --name tag_existing_posts
```

```python
from django.db import migrations


def tag_existing_posts(apps, schema_editor):
    ContentType = apps.get_model('contenttypes', 'ContentType')
    Post = apps.get_model('blog', 'Post')
    Tag = apps.get_model('social', 'Tag')
    TaggedItem = apps.get_model('social', 'TaggedItem')

    # The content type might not exist yet (post_migrate hasn't run) — create it if needed.
    post_ct, _ = ContentType.objects.get_or_create(app_label='blog', model='post')

    legacy, _ = Tag.objects.get_or_create(name='legacy')
    TaggedItem.objects.bulk_create(
        [TaggedItem(tag=legacy, content_type_id=post_ct.id, object_id=p.pk) for p in Post.objects.all()],
        ignore_conflicts=True,
    )


class Migration(migrations.Migration):
    dependencies = [
        ('contenttypes', '0002_remove_content_type_name'),
        ('blog', '0005_photo'),              # use your real latest blog migration
        ('social', '0004_tag_taggeditem'),   # use your real latest social migration
    ]

    operations = [
        migrations.RunPython(tag_existing_posts, migrations.RunPython.noop),
    ]
```

Rules for migrations:

| Rule                                                                                      | Why                                                                     |
|-------------------------------------------------------------------------------------------|-------------------------------------------------------------------------|
| Use `apps.get_model('contenttypes', 'ContentType')`, not a direct import                  | Historical models match the schema at that point in time                |
| Use `get_or_create(app_label=..., model=...)`                                             | Rows are created **after** migrations, so on a fresh DB they don't exist yet |
| `ContentType.objects.get_for_model(Post)` works in migrations too (the manager has `use_in_migrations = True`) | It also does `get_or_create`                     |
| Depend on `('contenttypes', '0002_remove_content_type_name')`                             | Guarantees the table has its final shape                                |
| `GenericForeignKey` is **not** available on historical models                             | Set `content_type_id` / `object_id` directly, as above                  |

### 30.3 Renaming an app (changing its label)

Django does **not** update content types when an app label changes. You must do it yourself in a migration of the renamed app:

```python
def rename_app_label(apps, schema_editor):
    ContentType = apps.get_model('contenttypes', 'ContentType')
    ContentType.objects.filter(app_label='oldapp').update(app_label='newapp')
```

(App renames also need the tables renamed and `django_migrations` rows updated — a bigger job; plan it carefully.)

---

## Step 31: Fixtures, `dumpdata` and Natural Keys

### 31.1 The problem

`dumpdata` writes content types as **numbers**:

```json
{"model": "social.comment", "pk": 1, "fields": {"content_type": 7, "object_id": 1, "text": "Great post!"}}
```

On another database `7` may be `auth.group`. `loaddata` would then attach the comment to the wrong model, and loading the `contenttypes.contenttype` rows themselves fails with `IntegrityError: UNIQUE constraint failed: django_content_type.app_label, django_content_type.model` (because `migrate` already created them).

### 31.2 The fix — natural keys

`ContentType` defines `natural_key()` → `("blog", "post")`, and its manager has `get_by_natural_key()`. Use them:

```bash
python manage.py dumpdata --natural-foreign --natural-primary -e contenttypes -e auth.Permission -e admin.logentry -e sessions --indent 2 -o data.json
```

| Flag                     | Effect                                                                          |
|--------------------------|---------------------------------------------------------------------------------|
| `--natural-foreign`      | FKs to `ContentType` (and `Permission`) are written as `["blog", "post"]`        |
| `--natural-primary`      | Omits `pk` for models that have a natural key                                   |
| `-e contenttypes`        | Don't dump content types — the target DB creates its own during `migrate`       |
| `-e auth.Permission`     | Same for permissions (created automatically)                                    |
| `-e admin.logentry`      | Admin history usually isn't needed                                              |
| `-o data.json`           | Write to a file (avoids Windows PowerShell encoding issues with `>`)            |

Result:

```json
{"model": "social.comment", "pk": 1, "fields": {"content_type": ["blog", "post"], "object_id": 1, "text": "Great post!"}}
```

Load on the other machine:

```bash
python manage.py migrate
python manage.py loaddata data.json
```

> **PowerShell tip:** `python manage.py dumpdata > data.json` writes UTF-16 in Windows PowerShell 5.1 and `loaddata` fails with "invalid start byte". Use `-o data.json` instead.

### 31.3 Fixtures for tests

The same applies to test fixtures: always dump with `--natural-foreign` and exclude `contenttypes` / `auth.Permission`.

---

## Step 32: Multiple Databases

Each database has its own `django_content_type` table, created by `migrate --database=<alias>`.

```python
>>> ContentType.objects.db_manager('other').get_for_model(Post)
>>> ContentType.objects.using('other').filter(app_label='blog')
>>> ct.get_object_for_this_type(using='other', pk=1)          # Django 5.1+
```

- The cache is kept **per database alias**.
- A `GenericForeignKey` loads its target from the same database as the object holding it.
- A generic object and its target should live in the **same** database (Django can't join across databases); a router that sends `blog` and `social` to different DBs will break `GenericRelation` queries.
- `remove_stale_contenttypes --database other` cleans one database at a time.

---

## Step 33: System Checks for Generic Fields

`contenttypes` registers checks that run on `runserver`, `migrate` and `python manage.py check`.

```bash
python manage.py check
```

| ID                   | Message (shortened)                                                                     | Fix                                                        |
|----------------------|------------------------------------------------------------------------------------------|------------------------------------------------------------|
| `contenttypes.E001`  | The GenericForeignKey object ID references the nonexistent field '…'                     | Fix the `fk_field` name or add the field                   |
| `contenttypes.E002`  | The GenericForeignKey content type references the nonexistent field '…'                  | Fix the `ct_field` name or add the field                   |
| `contenttypes.E003`  | '…' is not a ForeignKey                                                                  | The content type field must be a `ForeignKey`              |
| `contenttypes.E004`  | '…' is not a ForeignKey to 'contenttypes.ContentType'                                    | Point it at `ContentType`                                  |
| `contenttypes.E005`  | Model names must be at most 100 characters                                              | Shorten the class name (the `model` column is 100 chars)   |

Try one on purpose — change `GenericForeignKey('content_type', 'object_id')` to `GenericForeignKey('content_type', 'obj_id')` and run `check`:

```
social.Comment.content_object: (contenttypes.E001) The GenericForeignKey object ID references the nonexistent field 'obj_id'.
```

Change it back afterwards.

---

## Step 34: Test Content Types and Generic Relations

`social/tests.py`:

```python
from django.contrib.contenttypes.models import ContentType
from django.db import IntegrityError
from django.test import TestCase
from django.urls import reverse

from accounts.models import User
from blog.models import Photo, Post
from .models import Comment, Like
from .services import toggle_like


class ContentTypeTests(TestCase):
    def test_content_type_for_post(self):
        ct = ContentType.objects.get_for_model(Post)
        self.assertEqual((ct.app_label, ct.model), ('blog', 'post'))
        self.assertIs(ct.model_class(), Post)

    def test_get_for_model_is_cached(self):
        ContentType.objects.clear_cache()
        ContentType.objects.get_for_model(Post)          # fills the cache
        with self.assertNumQueries(0):
            ContentType.objects.get_for_model(Post)


class GenericRelationTests(TestCase):
    @classmethod
    def setUpTestData(cls):
        cls.user = User.objects.create_user('alice', password='pass12345')
        cls.post = Post.objects.create(
            title='Hello', slug='hello', content='...', author=cls.user, status='published'
        )
        cls.photo = Photo.objects.create(title='Sunset', image_url='https://example.com/s.jpg')

    def test_comment_on_post_and_photo(self):
        c1 = Comment.objects.create(content_object=self.post, author=self.user, text='a')
        c2 = Comment.objects.create(content_object=self.photo, author=self.user, text='b')
        self.assertEqual(c1.content_object, self.post)
        self.assertEqual(c2.content_object, self.photo)
        self.assertEqual(list(self.post.comments.all()), [c1])
        self.assertEqual(list(self.photo.comments.all()), [c2])

    def test_delete_target_cascades(self):
        Comment.objects.create(content_object=self.post, author=self.user, text='a')
        self.post.delete()
        self.assertEqual(Comment.objects.count(), 0)

    def test_like_is_unique(self):
        Like.objects.create(content_object=self.post, user=self.user)
        with self.assertRaises(IntegrityError):
            Like.objects.create(content_object=self.post, user=self.user)

    def test_toggle_like(self):
        self.assertTrue(toggle_like(self.user, self.post))
        self.assertFalse(toggle_like(self.user, self.post))

    def test_prefetch_content_object(self):
        Comment.objects.create(content_object=self.post, author=self.user, text='a')
        Comment.objects.create(content_object=self.photo, author=self.user, text='b')
        ContentType.objects.get_for_models(Post, Photo)          # warm the cache
        with self.assertNumQueries(3):                           # comments + posts + photos
            [c.content_object for c in Comment.objects.prefetch_related('content_object')]


class CommentViewTests(TestCase):
    @classmethod
    def setUpTestData(cls):
        cls.user = User.objects.create_user('bob', password='pass12345')
        cls.post = Post.objects.create(
            title='Hi', slug='hi', content='...', author=cls.user, status='published'
        )

    def test_add_comment(self):
        self.client.login(username='bob', password='pass12345')
        url = reverse('social:add_comment', args=['blog', 'post', self.post.pk])
        self.client.post(url, {'text': 'Nice'})
        self.assertEqual(self.post.comments.count(), 1)

    def test_disallowed_target_returns_404(self):
        self.client.login(username='bob', password='pass12345')
        url = reverse('social:add_comment', args=['accounts', 'user', self.user.pk])
        response = self.client.post(url, {'text': 'Hacked'})
        self.assertEqual(response.status_code, 404)
```

Run:

```bash
python manage.py test social
```

Testing rules:

- The test database is built with `migrate`, so content types **exist** in tests — but their IDs are not the same as in your dev DB. Never assert on hard-coded IDs.
- The content type cache survives between tests in the same process. If a test creates/deletes `ContentType` rows by hand, call `ContentType.objects.clear_cache()` in `tearDown()`.
- `assertNumQueries` counts may include a content type lookup the first time; warm the cache first (as above) to keep counts stable.

---

## Step 35: Best Practices and When Not to Use Generic Relations

**Do:**

- ✅ Always look content types up with `get_for_model()` / `get_by_natural_key()` / `get_for_id()` (cached).
- ✅ Add `Index(fields=['content_type', 'object_id'])` to every generic model.
- ✅ Match `object_id`'s type to the target primary keys (`PositiveBigIntegerField` for `BigAutoField`).
- ✅ Add a `GenericRelation` on every target model (reverse access + cascade delete).
- ✅ Whitelist target types in views and validate in `clean()`.
- ✅ Use `prefetch_related('content_object')` / `GenericPrefetch` in list pages.
- ✅ Dump fixtures with `--natural-foreign` and exclude `contenttypes` / `auth.Permission`.
- ✅ Use `RenameModel` (answer **y**) when renaming models.
- ✅ Check `model_class()` for `None` before using it.

**Don't:**

- ❌ Hard-code content type IDs anywhere (code, fixtures, settings, JS).
- ❌ Edit or delete `django_content_type` rows by hand (permissions and generic objects cascade).
- ❌ Accept raw `content_type_id` from users without a whitelist.
- ❌ Expect database-level integrity on `object_id`.
- ❌ Use a generic relation when a plain `ForeignKey` would do (one or two known targets).

**Alternatives when generic relations feel wrong:**

| Need                                         | Alternative                                                         |
|----------------------------------------------|---------------------------------------------------------------------|
| 2–3 known targets, need DB integrity         | Several nullable `ForeignKey`s + a `CheckConstraint` "exactly one set" |
| Shared fields across models                  | Abstract base model                                                 |
| One "parent" type with subtypes              | Multi-table inheritance, or django-polymorphic (uses content types) |
| Tags                                         | django-taggit                                                       |
| Object-level permissions                     | django-guardian                                                     |
| Activity streams                             | django-activity-stream                                              |

---

## Reference: `ContentType` API

| Call                                                              | Returns                       | Cached |
|-------------------------------------------------------------------|-------------------------------|--------|
| `ContentType.objects.get_for_model(Post)`                         | `ContentType`                 | ✅     |
| `ContentType.objects.get_for_model(post_instance)`                | `ContentType`                 | ✅     |
| `ContentType.objects.get_for_model(Proxy, for_concrete_model=False)` | proxy's own `ContentType`  | ✅     |
| `ContentType.objects.get_for_models(Post, Photo)`                 | `dict {Model: ContentType}`   | ✅     |
| `ContentType.objects.get_for_id(7)`                               | `ContentType`                 | ✅     |
| `ContentType.objects.get_by_natural_key('blog', 'post')`          | `ContentType`                 | ✅     |
| `ContentType.objects.get(app_label='blog', model='post')`         | `ContentType`                 | ❌     |
| `ContentType.objects.clear_cache()`                               | `None`                        | —      |
| `ct.model_class()`                                                | model class or `None`         | —      |
| `ct.get_object_for_this_type(**kw)`                               | one object                    | —      |
| `ct.get_all_objects_for_this_type(**kw)`                          | `QuerySet`                    | —      |
| `ct.natural_key()`                                                | `('blog', 'post')`            | —      |
| `ct.name`                                                         | `'post'`                      | —      |
| `ct.app_labeled_name` / `str(ct)`                                 | `'Blog \| post'`               | —      |

## Reference: Generic Fields

| Class                | Where it goes            | DB column? | Key arguments                                                                  |
|----------------------|--------------------------|------------|---------------------------------------------------------------------------------|
| `GenericForeignKey`  | The "child" (`Comment`)  | ❌ No      | `ct_field='content_type'`, `fk_field='object_id'`, `for_concrete_model=True`    |
| `GenericRelation`    | Each "target" (`Post`)   | ❌ No      | `to`, `content_type_field`, `object_id_field`, `related_query_name`, `for_concrete_model` |
| `ForeignKey(ContentType)` | The child           | ✅ Yes     | `on_delete`, `limit_choices_to`, `related_name`                                 |
| `object_id` field    | The child                | ✅ Yes     | Type must match target PKs                                                     |

## Reference: What Works With a `GenericForeignKey`

| Operation                                    | Works?                                         |
|----------------------------------------------|------------------------------------------------|
| `Comment(content_object=obj)` / `.create(content_object=obj)` | ✅                             |
| `comment.content_object` (read)              | ✅ (1 query, then cached on the instance)       |
| `comment.content_object = obj` (assign)      | ✅                                             |
| `prefetch_related('content_object')`         | ✅                                             |
| `GenericPrefetch('content_object', [...])`   | ✅ (Django 5.0+)                                |
| `select_related('content_object')`           | ❌                                             |
| `filter(content_object=obj)` / `exclude()`   | ❌ — filter on `content_type` + `object_id`     |
| `order_by('content_object')` / `values('content_object')` | ❌                                |
| In `ModelForm.Meta.fields` / admin `fields`  | ❌ — use the two real fields or a generic inline |
| In `unique_together` / `UniqueConstraint` / `Index` | ❌ — use `content_type` + `object_id`   |
| Cascade delete when target is deleted        | ✅ only if the target has a `GenericRelation`   |

## Reference: Management Commands

| Command                                                                  | What it does                                            |
|--------------------------------------------------------------------------|---------------------------------------------------------|
| `python manage.py migrate`                                               | Creates missing content types (via `post_migrate`)     |
| `python manage.py migrate --verbosity 2`                                 | Same, and prints "Adding content type …"               |
| `python manage.py showmigrations contenttypes`                           | Shows `0001_initial`, `0002_remove_content_type_name`  |
| `python manage.py sqlmigrate contenttypes 0001`                          | Shows the table SQL                                     |
| `python manage.py remove_stale_contenttypes`                             | Deletes types for deleted models (asks first)          |
| `python manage.py remove_stale_contenttypes --include-stale-apps`        | Also for uninstalled apps                               |
| `python manage.py remove_stale_contenttypes --noinput --database other`  | No prompt, specific DB                                  |
| `python manage.py dumpdata --natural-foreign --natural-primary -e contenttypes -e auth.Permission -o data.json` | Portable fixtures |
| `python manage.py check`                                                 | Runs the generic foreign key checks                     |

---

## Troubleshooting

| Problem / error                                                                                      | Cause                                                                                    | Fix                                                                                         |
|------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------|
| `ContentType matching query does not exist`                                                         | Wrong case (`model='Post'`), model not migrated yet, or wrong `app_label`                | Use lowercase / `get_for_model()`; run `migrate`                                            |
| `ct.model_class()` returns `None`                                                                    | Stale content type — model deleted or app removed                                        | Handle `None`; run `remove_stale_contenttypes`                                              |
| `FieldError: Field 'content_object' does not generate an automatic reverse relation…`                | Filtering on a `GenericForeignKey`                                                       | Filter on `content_type` + `object_id`, or via `GenericRelation`'s `related_query_name`     |
| `FieldError: Cannot resolve keyword 'post' into field`                                               | No `related_query_name='post'` on the `GenericRelation`                                  | Add it                                                                                      |
| Reverse lookup `post.comments.all()` is empty but comments exist                                     | Field names differ (`target_type`/`target_id`) or `for_concrete_model` mismatch          | Pass `content_type_field=`, `object_id_field=`, matching `for_concrete_model`               |
| Comments not deleted when the post is deleted                                                        | Target has no `GenericRelation`                                                          | Add `GenericRelation` to the target model                                                  |
| `ValueError: save() prohibited … unsaved related object 'content_object'`                            | Assigned an unsaved object                                                               | `save()` the target first                                                                   |
| `IntegrityError: UNIQUE constraint failed: django_content_type.app_label, …` on `loaddata`           | Fixture contains content type rows                                                       | Dump with `-e contenttypes -e auth.Permission --natural-foreign`                            |
| Comments attached to the wrong model after `loaddata`                                                | Fixture has numeric content type ids from another DB                                     | Re-dump with `--natural-foreign`                                                           |
| `operator does not exist: character varying = integer` (PostgreSQL)                                  | `object_id` is text, target PK is integer                                                | Use an integer `object_id` matching the PK type                                             |
| `fields.E304 Reverse accessor … clashes` on two FKs to `ContentType`                                 | Two `ForeignKey(ContentType)` on one model                                               | Give each `related_name='+'` or unique names                                                |
| `contenttypes.E001` – `E004` on `check`                                                              | `GenericForeignKey` arguments don't match field names/types                              | See Step 33                                                                                 |
| Permissions disappeared after `remove_stale_contenttypes`                                            | They cascaded with the stale type                                                        | Restore from backup; next time use `RenameModel`                                            |
| Data migration: content type not found on a fresh database                                           | Content types are created **after** migrations                                           | `ContentType.objects.get_or_create(app_label=..., model=...)` in the migration              |
| `'social_tags' is not a registered tag library`                                                      | Missing `templatetags/__init__.py`, app not installed, or server not restarted           | Add the file, check `INSTALLED_APPS`, restart `runserver`                                    |
| `AlreadyRegistered: The model Post is already registered`                                            | Registered `Post` twice in admin                                                         | Merge into one `PostAdmin`                                                                  |
| `loaddata` fails with "invalid start byte" on Windows                                                | `dumpdata > file` in PowerShell 5.1 wrote UTF-16                                         | Use `dumpdata -o data.json`                                                                 |

---

## Final Project Structure

```
myproject/
├── manage.py
├── db.sqlite3
├── config/
│   ├── settings.py              # 'django.contrib.contenttypes' + 'social' in INSTALLED_APPS
│   └── urls.py                  # includes social.urls (+ optional shortcut view)
├── accounts/                    # custom User (from the auth guide)
├── blog/
│   ├── admin.py                 # PostAdmin / PhotoAdmin with GenericTabularInline
│   ├── models.py                # Post, Photo with GenericRelation(comments, likes, tagged_items, activities)
│   └── templates/blog/
│       └── post_detail.html     # {% load social_tags %}, like button, {% comments_block post %}
└── social/
    ├── admin.py                 # CommentAdmin, Like, Tag, TaggedItem
    ├── apps.py                  # ready() imports signals
    ├── forms.py                 # CommentForm, CommentFormSet (generic_inlineformset_factory)
    ├── migrations/
    ├── models.py                # Comment, Like, Tag, TaggedItem, Activity (GenericForeignKey)
    ├── services.py              # toggle_like, is_liked
    ├── signals.py               # post_save → Activity
    ├── templatetags/
    │   ├── __init__.py
    │   └── social_tags.py       # target_args, liked_by_me, comments_block
    ├── templates/social/
    │   ├── _comments.html
    │   └── edit_comments.html
    ├── tests.py
    ├── urls.py                  # add_comment, like
    └── views.py                 # whitelisted generic views
```

---

## Quick Reference

```python
# Imports
from django.contrib.contenttypes.models import ContentType
from django.contrib.contenttypes.fields import GenericForeignKey, GenericRelation
from django.contrib.contenttypes.admin import GenericTabularInline, GenericStackedInline
from django.contrib.contenttypes.forms import generic_inlineformset_factory
from django.contrib.contenttypes.prefetch import GenericPrefetch      # Django 5.0+
from django.contrib.contenttypes.views import shortcut

# Model → content type → model
ct = ContentType.objects.get_for_model(Post)
ct = ContentType.objects.get_by_natural_key('blog', 'post')
ct = ContentType.objects.get_for_id(ct_id)
Model = ct.model_class()
obj = ct.get_object_for_this_type(pk=1)

# Generic child model
class Comment(models.Model):
    content_type = models.ForeignKey(ContentType, on_delete=models.CASCADE)
    object_id = models.PositiveBigIntegerField()
    content_object = GenericForeignKey('content_type', 'object_id')
    class Meta:
        indexes = [models.Index(fields=['content_type', 'object_id'])]

# Target model
class Post(models.Model):
    comments = GenericRelation('social.Comment', related_query_name='post')

# Use
Comment.objects.create(content_object=post, author=user, text='Hi')
post.comments.all()
Comment.objects.filter(content_type=ct, object_id=post.pk)
Comment.objects.filter(post__status='published')
Comment.objects.prefetch_related('content_object')
Post.objects.annotate(n=Count('comments'))
```

```bash
python manage.py migrate --verbosity 2
python manage.py remove_stale_contenttypes --include-stale-apps
python manage.py dumpdata --natural-foreign --natural-primary -e contenttypes -e auth.Permission -o data.json
python manage.py check
```
