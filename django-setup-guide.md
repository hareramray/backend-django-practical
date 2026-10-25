# How to Create a Django Application: Step-by-Step Guide

This guide walks you through creating a Django project from scratch on **Windows**, with notes for macOS/Linux where commands differ. By the end you will have a working app with a model, views, templates, URLs and the admin panel.

---

## Prerequisites

- **Python 3.10+** installed. Check with:
  ```bash
  python --version
  ```
- **pip** (comes with Python):
  ```bash
  pip --version
  ```
- A code editor such as VS Code.

---

## Step 1: Create a Project Folder

```bash
mkdir myproject
cd myproject
```

---

## Step 2: Create and Activate a Virtual Environment

A virtual environment keeps your project's packages separate from other projects.

```bash
python -m venv venv
```

Activate it:

| OS / Shell              | Command                         |
|-------------------------|---------------------------------|
| Windows (PowerShell)    | `venv\Scripts\Activate.ps1`     |
| Windows (CMD)           | `venv\Scripts\activate.bat`     |
| Windows (Git Bash)      | `source venv/Scripts/activate`  |
| macOS / Linux           | `source venv/bin/activate`      |

When it's active, you'll see `(venv)` at the start of your terminal prompt.

> **PowerShell error "running scripts is disabled"?** Run this once:
> `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`

---

## Step 3: Install Django

```bash
pip install django
```

Verify the installation:

```bash
python -m django --version
```

Save your dependencies so others can install them later:

```bash
pip freeze > requirements.txt
```

---

## Step 4: Create a Django Project

```bash
django-admin startproject config .
```

> The `.` at the end creates the project in the current folder rather than in an extra nested folder.

Resulting structure:

```
myproject/
├── manage.py          # Command-line tool for the project
├── config/
│   ├── __init__.py
│   ├── settings.py    # Project settings
│   ├── urls.py        # Root URL configuration
│   ├── asgi.py
│   └── wsgi.py
└── venv/
```

---

## Step 5: Run the Development Server

```bash
python manage.py runserver
```

Open **http://127.0.0.1:8000/** in your browser. You should see the Django welcome page with a rocket. 🚀

Press `Ctrl + C` to stop the server.

---

## Step 6: Create an App

A Django **project** is made up of one or more **apps**. Each app handles one feature (blog, users, payments, etc.).

```bash
python manage.py startapp blog
```

App structure:

```
blog/
├── __init__.py
├── admin.py        # Register models for the admin panel
├── apps.py
├── migrations/     # Database migration files
├── models.py       # Database models
├── tests.py
└── views.py        # Request handlers
```

---

## Step 7: Register the App in Settings

Open `config/settings.py` and add your app to `INSTALLED_APPS`:

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'blog',  # 👈 add this
]
```

---

## Step 8: Create a Model

Models define your database tables. Edit `blog/models.py`:

```python
from django.db import models


class Post(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.title
```

---

## Step 9: Make and Apply Migrations

Migrations turn your models into database tables (SQLite by default).

```bash
python manage.py makemigrations
python manage.py migrate
```

- `makemigrations` creates migration files from model changes.
- `migrate` applies them to the database.

> Run both commands **every time** you change a model.

---

## Step 10: Create a Superuser and Use the Admin Panel

```bash
python manage.py createsuperuser
```

Enter a username, email (optional) and password.

Register the model in `blog/admin.py`:

```python
from django.contrib import admin
from .models import Post

admin.site.register(Post)
```

Run the server and go to **http://127.0.0.1:8000/admin/**. Log in and add a few posts.

---

## Step 11: Write a View

Edit `blog/views.py`:

```python
from django.shortcuts import render, get_object_or_404
from .models import Post


def post_list(request):
    posts = Post.objects.order_by('-created_at')
    return render(request, 'blog/post_list.html', {'posts': posts})


def post_detail(request, pk):
    post = get_object_or_404(Post, pk=pk)
    return render(request, 'blog/post_detail.html', {'post': post})
```

---

## Step 12: Configure URLs

**1. Create `blog/urls.py`** (this file doesn't exist yet):

```python
from django.urls import path
from . import views

app_name = 'blog'

urlpatterns = [
    path('', views.post_list, name='post_list'),
    path('post/<int:pk>/', views.post_detail, name='post_detail'),
]
```

**2. Include it in `config/urls.py`:**

```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('blog.urls')),
]
```

---

## Step 13: Create Templates

Django looks for templates inside each app's `templates/` folder. Create this structure:

```
blog/
└── templates/
    └── blog/
        ├── base.html
        ├── post_list.html
        └── post_detail.html
```

> The extra `blog/` folder inside `templates/` prevents name clashes with other apps.

**`blog/templates/blog/base.html`**

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>{% block title %}My Blog{% endblock %}</title>
</head>
<body>
    <h1><a href="{% url 'blog:post_list' %}">My Blog</a></h1>
    <hr>
    {% block content %}{% endblock %}
</body>
</html>
```

**`blog/templates/blog/post_list.html`**

```html
{% extends 'blog/base.html' %}

{% block content %}
    {% for post in posts %}
        <h2><a href="{% url 'blog:post_detail' post.pk %}">{{ post.title }}</a></h2>
        <p>{{ post.created_at|date:"d M Y" }}</p>
    {% empty %}
        <p>No posts yet.</p>
    {% endfor %}
{% endblock %}
```

**`blog/templates/blog/post_detail.html`**

```html
{% extends 'blog/base.html' %}

{% block title %}{{ post.title }}{% endblock %}

{% block content %}
    <h2>{{ post.title }}</h2>
    <p><small>{{ post.created_at|date:"d M Y, H:i" }}</small></p>
    <p>{{ post.content|linebreaks }}</p>
{% endblock %}
```

Run the server and visit **http://127.0.0.1:8000/** to see your posts.

---

## Step 14: Add Static Files (CSS, JS, Images)

Create the folder `blog/static/blog/` and add `style.css`:

```css
body {
    font-family: Arial, sans-serif;
    max-width: 800px;
    margin: 40px auto;
}
```

Load it in `base.html` (add `{% load static %}` at the very top):

```html
{% load static %}
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>{% block title %}My Blog{% endblock %}</title>
    <link rel="stylesheet" href="{% static 'blog/style.css' %}">
</head>
```

---

## Step 15: Write a Basic Test

Edit `blog/tests.py`:

```python
from django.test import TestCase
from django.urls import reverse
from .models import Post


class PostTests(TestCase):
    def setUp(self):
        self.post = Post.objects.create(title='Test Post', content='Hello')

    def test_post_list_page(self):
        response = self.client.get(reverse('blog:post_list'))
        self.assertEqual(response.status_code, 200)
        self.assertContains(response, 'Test Post')

    def test_post_detail_page(self):
        response = self.client.get(reverse('blog:post_detail', args=[self.post.pk]))
        self.assertEqual(response.status_code, 200)
```

Run the tests:

```bash
python manage.py test
```

---

## Step 16: Prepare for Production (Overview)

Before deploying, update `config/settings.py`:

```python
DEBUG = False
ALLOWED_HOSTS = ['yourdomain.com']
STATIC_ROOT = BASE_DIR / 'staticfiles'
```

- Move `SECRET_KEY` to an environment variable — never commit it to Git.
- Collect static files: `python manage.py collectstatic`
- Use a production server such as **Gunicorn** (Linux) or **Waitress** (Windows) instead of `runserver`.
- Consider **PostgreSQL** instead of SQLite.

---

## Final Project Structure

```
myproject/
├── manage.py
├── requirements.txt
├── db.sqlite3
├── config/
│   ├── settings.py
│   └── urls.py
└── blog/
    ├── admin.py
    ├── models.py
    ├── urls.py
    ├── views.py
    ├── tests.py
    ├── migrations/
    ├── static/blog/style.css
    └── templates/blog/
        ├── base.html
        ├── post_list.html
        └── post_detail.html
```

---

## Quick Command Reference

| Task                         | Command                                   |
|------------------------------|-------------------------------------------|
| Create virtual environment   | `python -m venv venv`                     |
| Install Django               | `pip install django`                      |
| Create project               | `django-admin startproject config .`      |
| Create app                   | `python manage.py startapp <app_name>`    |
| Run dev server               | `python manage.py runserver`              |
| Create migrations            | `python manage.py makemigrations`         |
| Apply migrations             | `python manage.py migrate`                |
| Create admin user            | `python manage.py createsuperuser`        |
| Open Django shell            | `python manage.py shell`                  |
| Run tests                    | `python manage.py test`                   |
| Collect static files         | `python manage.py collectstatic`          |
| Save dependencies            | `pip freeze > requirements.txt`           |
| Install from requirements    | `pip install -r requirements.txt`         |

---

## Next Steps

- Add forms with `django.forms` / `ModelForm` to create posts from the website.
- Use class-based views (`ListView`, `DetailView`).
- Add user authentication (login, logout, signup).
- Build a REST API with **Django REST Framework**.
- Official docs: https://docs.djangoproject.com/
