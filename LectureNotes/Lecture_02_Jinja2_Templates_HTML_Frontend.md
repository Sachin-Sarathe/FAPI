# Lecture 02 — HTML Frontend for Your API: Jinja2 Templates

> **Source:** [Python FastAPI Tutorial (Part 2): HTML Frontend for Your API - Jinja2 Templates — Corey Schafer](https://www.youtube.com/watch?v=G4NIB9Rx9Qs)
> **Playlist:** [FastAPI Tutorials — Corey Schafer](https://www.youtube.com/playlist?list=PL-osiE80TeTsak-c-QsVeg0YYG_0TeyXI)

---

## Table of Contents

1. [Why Templates? The Problem with Inline HTML](#1-why-templates-the-problem-with-inline-html)
2. [What is Jinja2?](#2-what-is-jinja2)
3. [Project Structure for Templates](#3-project-structure-for-templates)
4. [Serving Static Files — `StaticFiles`](#4-serving-static-files--staticfiles)
5. [Setting Up Jinja2 in FastAPI](#5-setting-up-jinja2-in-fastapi)
6. [The `Request` Object — Why Routes Need It Now](#6-the-request-object--why-routes-need-it-now)
7. [Returning a `TemplateResponse`](#7-returning-a-templateresponse)
8. [Jinja2 Template Syntax](#8-jinja2-template-syntax)
9. [Template Inheritance — Base Layout Pattern](#9-template-inheritance--base-layout-pattern)
10. [Breaking Down `layout.html`](#10-breaking-down-layouthml)
11. [Breaking Down `home.html`](#11-breaking-down-homehtml)
12. [`url_for()` — Dynamic URL Generation in Templates](#12-url_for--dynamic-url-generation-in-templates)
13. [Full Updated `main.py` Walkthrough](#13-full-updated-mainpy-walkthrough)
14. [Bootstrap Integration](#14-bootstrap-integration)
15. [Common Mistakes & Gotchas](#15-common-mistakes--gotchas)
16. [Summary & Key Takeaways](#16-summary--key-takeaways)
17. [References](#17-references)

---

## 1. Why Templates? The Problem with Inline HTML

In Lecture 1, we returned raw HTML strings directly from route functions:

```python
# Lecture 1 — works but doesn't scale
@app.get("/", response_class=HTMLResponse)
def home():
    return f"<h1>{posts[0]['title']}</h1>"
```

This falls apart immediately for real pages — you'd be writing hundreds of lines of HTML inside Python strings, with no syntax highlighting, no editor support, and no reuse. The solution is a **template engine**.

> A **template** is an HTML file with special placeholder syntax that gets filled in with dynamic data at request time. The template engine (Jinja2) processes the file, substitutes the placeholders, and hands a complete HTML string to FastAPI.

---

## 2. What is Jinja2?

**Jinja2** is a fast, expressive templating engine for Python — the same one used by Flask, Django (optionally), and many other frameworks.

```
Python data (dict)  +  HTML template file  →  Jinja2  →  Final HTML string
```

- Created by Armin Ronacher (same person as Flask)
- Syntax inspired by Django templates, but more powerful
- Sandboxed execution — templates cannot run arbitrary Python
- FastAPI uses **Starlette's Jinja2 integration** under the hood

💡 FastAPI supports any template engine, but Jinja2 is the recommended and officially documented choice.

---

## 3. Project Structure for Templates

FastAPI expects templates and static files in specific directories. The convention:

```
FastAPIStudy/
├── main.py
├── templates/           ← all .html template files go here
│   ├── layout.html      ← base layout (shared across all pages)
│   └── home.html        ← page-specific templates
├── static/              ← CSS, JS, images, icons
│   ├── css/
│   │   └── main.css
│   ├── js/
│   │   └── utils.js
│   ├── icons/
│   │   ├── favicon.ico
│   │   └── icon.svg
│   └── profile_pics/
│       └── default.jpg
└── pyproject.toml
```

📌 These directory names (`templates/`, `static/`) are **conventions, not requirements**. You can name them anything, but you'll need to update the corresponding code. Stick to the convention.

---

## 4. Serving Static Files — `StaticFiles`

Before templates can reference CSS, images, or JS, you need to tell FastAPI where those files live and under what URL to serve them.

```python
from fastapi.staticfiles import StaticFiles

app.mount("/static", StaticFiles(directory="static"), name="static")
```

### Breaking this down

| Part | Meaning |
|---|---|
| `"/static"` | The URL prefix — any request starting with `/static/...` is handled here |
| `StaticFiles(directory="static")` | Look for files in the `static/` folder on disk |
| `name="static"` | Internal name used by `url_for()` to generate URLs to static files |

### What "mounting" means

`app.mount()` attaches a **completely independent sub-application** to a URL path. This is different from a regular route — mounted apps handle the entire sub-path tree themselves. Uvicorn routes `/static/*` requests directly to the file server without them ever reaching your Python route functions.

```
GET /static/css/main.css  →  StaticFiles serves the file directly from disk
GET /posts                →  Goes through your FastAPI route functions
```

> ⚠️ **Mount order matters.** Call `app.mount()` before defining routes that reference static files. In practice, put it right after creating `app`.

> ⚠️ **Mounted paths are excluded from API docs.** `/static` won't appear in Swagger UI — that's correct and expected.

---

## 5. Setting Up Jinja2 in FastAPI

```python
from fastapi.templating import Jinja2Templates

templates = Jinja2Templates(directory="templates")
```

- `Jinja2Templates(directory="templates")` creates a reusable templates object that knows to look inside the `templates/` folder for `.html` files.
- You create this **once** at module level and reuse it across all routes.
- Under the hood this is `starlette.templating.Jinja2Templates` — FastAPI re-exports it for convenience.

### Full import block (updated `main.py`)

```python
from fastapi import FastAPI, Request
from fastapi.templating import Jinja2Templates
from fastapi.staticfiles import StaticFiles

app = FastAPI()

app.mount("/static", StaticFiles(directory="static"), name="static")
templates = Jinja2Templates(directory="templates")
```

---

## 6. The `Request` Object — Why Routes Need It Now

When returning plain JSON, route functions don't need to know anything about the HTTP request. But templates do — Jinja2 uses the request to generate correct URLs (via `url_for()`), handle cookies, and more.

```python
from fastapi import Request

@app.get("/")
def home(request: Request):      # ← must declare Request as a parameter
    return templates.TemplateResponse(request, "home.html", {})
```

FastAPI sees `request: Request` and automatically injects the current HTTP request object into the function — no manual wiring needed. This is FastAPI's **dependency injection** at its simplest form (more on DI in a later lecture).

📌 `Request` comes from `fastapi` (which re-exports it from Starlette). It holds everything about the incoming HTTP request: URL, headers, cookies, body, client IP, etc.

---

## 7. Returning a `TemplateResponse`

```python
return templates.TemplateResponse(request, "home.html", {"posts": posts, "title": "home"})
```

### Signature (FastAPI ≥ 0.108.0)

```python
templates.TemplateResponse(
    request,          # ← the Request object (first positional arg)
    name,             # ← template filename, relative to templates/ directory
    context,          # ← dict of variables available inside the template
)
```

> ⚠️ **Breaking change in FastAPI 0.108.0:** In older versions, `name` was the first argument and `request` was passed *inside* the context dict. The new signature (shown above) is the current standard:
> ```python
> # OLD (before 0.108.0) — don't use this
> templates.TemplateResponse("home.html", {"request": request, "posts": posts})
>
> # NEW (current) — use this
> templates.TemplateResponse(request, "home.html", {"posts": posts})
> ```

### What the context dict does

Every key-value pair in the context dict becomes a **variable available inside the template**:

```python
context = {"posts": posts, "title": "home"}
```

Inside the template you can now use `{{ posts }}`, `{{ title }}`, loop over `posts`, etc.

### Response type

`TemplateResponse` automatically sets:
- `Content-Type: text/html; charset=utf-8`
- `200 OK` status (by default)

No need for `response_class=HTMLResponse` on the decorator when returning a `TemplateResponse` — though it's good practice to add it so Swagger UI knows this route returns HTML:

```python
from fastapi.responses import HTMLResponse

@app.get("/", response_class=HTMLResponse, include_in_schema=False)
def home(request: Request):
    return templates.TemplateResponse(request, "home.html", {"posts": posts})
```

---

## 8. Jinja2 Template Syntax

Jinja2 uses three delimiter types. Memorise these — they're the core of everything.

| Delimiter | Purpose | Example |
|---|---|---|
| `{{ ... }}` | Output/expression — render a variable | `{{ post.title }}` |
| `{% ... %}` | Statement — logic (loops, if, extends) | `{% for post in posts %}` |
| `{# ... #}` | Comment — not rendered in output | `{# TODO: add pagination #}` |

### Variables

```html
<h2>{{ post.title }}</h2>          <!-- dict/object attribute access -->
<p>{{ post['content'] }}</p>       <!-- dict key access (both work) -->
<small>{{ post.author | upper }}</small>  <!-- with a filter -->
```

### Conditionals

```html
{% if title %}
  <title>FastAPI Blog - {{ title }}</title>
{% else %}
  <title>FastAPI Blog</title>
{% endif %}
```

```html
{% if user.is_authenticated %}
  <a href="/logout">Logout</a>
{% elif user.is_guest %}
  <a href="/register">Register</a>
{% else %}
  <a href="/login">Login</a>
{% endif %}
```

### For Loops

```html
{% for post in posts %}
  <article>
    <h2>{{ post.title }}</h2>
    <p>{{ post.content }}</p>
  </article>
{% endfor %}
```

#### Loop special variables

Inside a `{% for %}` block, Jinja2 provides a `loop` object:

| Variable | Description |
|---|---|
| `loop.index` | Current iteration (1-based) |
| `loop.index0` | Current iteration (0-based) |
| `loop.first` | `True` if first iteration |
| `loop.last` | `True` if last iteration |
| `loop.length` | Total number of items |

```html
{% for post in posts %}
  <p>Post {{ loop.index }} of {{ loop.length }}: {{ post.title }}</p>
{% else %}
  <p>No posts found.</p>   <!-- rendered when the list is empty -->
{% endfor %}
```

### Filters

Filters transform a value using the pipe `|` operator:

```html
{{ post.title | upper }}          <!-- FASTAPI IS AWESOME -->
{{ post.title | lower }}          <!-- fastapi is awesome -->
{{ post.title | title }}          <!-- Fastapi Is Awesome -->
{{ post.content | truncate(50) }} <!-- Truncates to 50 chars -->
{{ posts | length }}              <!-- Number of posts -->
{{ post.title | replace("FastAPI", "Python") }}
```

Common built-in filters:

| Filter | Example | Result |
|---|---|---|
| `upper` | `{{ "hello" \| upper }}` | `HELLO` |
| `lower` | `{{ "HELLO" \| lower }}` | `hello` |
| `title` | `{{ "hello world" \| title }}` | `Hello World` |
| `length` | `{{ posts \| length }}` | `2` |
| `truncate(n)` | `{{ text \| truncate(80) }}` | first 80 chars + `...` |
| `default(val)` | `{{ name \| default("Guest") }}` | value or fallback |
| `safe` | `{{ html_content \| safe }}` | render raw HTML (don't escape) |

> ⚠️ **Auto-escaping:** Jinja2 in FastAPI **auto-escapes** HTML by default in `.html` files. This means `{{ "<script>alert(1)</script>" }}` renders as `&lt;script&gt;` — safe from XSS. Only use `| safe` when you fully trust the content.

---

## 9. Template Inheritance — Base Layout Pattern

The most powerful Jinja2 feature. Instead of duplicating the navbar, footer, and `<head>` in every HTML file, you define them once in a **base template** and every other page just fills in the unique content.

### How it works

```
layout.html (base)          home.html (child)
─────────────────           ─────────────────
<html>                      {% extends "layout.html" %}
  <head>...</head>
  <body>                    {% block content %}
    <navbar>...</navbar>      <!-- page-specific HTML -->
    {% block content %}     {% endblock content %}
    {% endblock content %}
    <footer>...</footer>
  </body>
</html>
```

### In the base template — define blocks

```html
<!-- layout.html -->
<main>
  <div class="row">
    <div class="col-md-8">
      {% block content %}
      {% endblock content %}   <!-- child templates fill this in -->
    </div>
  </div>
</main>
```

- `{% block content %}` — declares a replaceable region named `content`
- `{% endblock content %}` — closing tag (the name is optional but recommended for clarity)
- Any HTML between `{% block %}` and `{% endblock %}` is the **default content** shown if a child doesn't override it

### In the child template — extend and fill blocks

```html
<!-- home.html -->
{% extends "layout.html" %}

{% block content %}
  <!-- Everything here replaces the block in layout.html -->
  {% for post in posts %}
    <article>{{ post.title }}</article>
  {% endfor %}
{% endblock content %}
```

- `{% extends "layout.html" %}` **must be the very first line** of the child template
- A child can define multiple blocks to override multiple regions of the parent
- Content outside of `{% block %}` tags in a child template is **ignored**

💡 You can have as many named blocks as you want: `{% block title %}`, `{% block sidebar %}`, `{% block scripts %}`, etc.

---

## 10. Breaking Down `layout.html`

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <!-- Dynamic page title with conditional -->
    {% if title %}
      <title>FastAPI Blog - {{ title }}</title>
    {% else %}
      <title>FastAPI Blog</title>
    {% endif %}

    <!-- Bootstrap CSS from CDN -->
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.8/dist/css/bootstrap.min.css" rel="stylesheet">

    <!-- Custom stylesheet via url_for() -->
    <link rel="stylesheet" href="{{ url_for('static', path='css/main.css') }}">
  </head>

  <body class="d-flex flex-column min-vh-100">

    <!-- Navbar -->
    <header class="site-header">
      <nav class="navbar navbar-expand-md bg-steel fixed-top" data-bs-theme="dark">
        <div class="container">
          <a class="navbar-brand" href="{{ url_for('home') }}">FastAPI Blog</a>
          <!-- ... -->
        </div>
      </nav>
    </header>

    <!-- Main content area -->
    <main role="main" class="container">
      <div class="row">
        <div class="col-md-8">
          {% block content %}
          {% endblock content %}      <!-- child pages fill this -->
        </div>
        <aside class="col-md-4">
          <!-- Sidebar (static for now) -->
        </aside>
      </div>
    </main>

    <!-- Footer with dynamic year -->
    <footer class="mt-auto py-3 bg-body-tertiary border-top">
      <p>© <span id="year"></span> Corey Schafer</p>
    </footer>

    <script>document.getElementById('year').textContent = new Date().getFullYear();</script>

    <!-- Bootstrap JS + Dark Mode Toggle script -->
  </body>
</html>
```

### Notable patterns in `layout.html`

**Dynamic page title:**
```html
{% if title %}
  <title>FastAPI Blog - {{ title }}</title>
{% else %}
  <title>FastAPI Blog</title>
{% endif %}
```
The `title` variable comes from the context dict passed in `TemplateResponse`. If no title is passed, it falls back to a default.

**Dynamic copyright year:**
```javascript
document.getElementById('year').textContent = new Date().getFullYear();
```
JavaScript sets the year at render time — no need to update it annually.

**Dark mode toggle:**
```javascript
const setTheme = (theme) => {
  document.documentElement.setAttribute('data-bs-theme', theme === 'auto'
    ? (window.matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light')
    : theme);
  localStorage.setItem('theme', theme);
};
// Initialize from localStorage on page load
setTheme(localStorage.getItem('theme') || 'auto');
```
Bootstrap 5.3+ supports `data-bs-theme="dark"` / `"light"` / `"auto"` on the `<html>` element. The script persists the user's choice in `localStorage`.

**Open Graph tags** (for social sharing previews):
```html
<meta property="og:title" content="FastAPI Blog">
<meta property="og:type" content="website">
<meta property="og:url" content="">
<meta property="og:image" content="">
```
When a link to your site is shared on Twitter, LinkedIn, Slack, etc., these tags control the preview card title, image, and description.

---

## 11. Breaking Down `home.html`

```html
{% extends "layout.html" %}

{% block content %}
  {% for post in posts %}
    <article class="content-section py-3 px-4 mb-4">
      <div class="d-flex align-items-start gap-4">

        <!-- Author profile picture -->
        <img class="rounded-circle article-img flex-shrink-0"
             src="/static/profile_pics/default.jpg"
             alt="{{ post.author }}'s profile picture"
             width="64" height="64" loading="lazy">

        <div class="flex-grow-1">
          <div class="article-metadata mb-2">
            <a class="me-2" href="#">{{ post.author }}</a>
            <small class="text-body-secondary">{{ post.date_posted }}</small>
          </div>
          <h2>
            <a class="article-title" href="#">{{ post.title }}</a>
          </h2>
          <p class="article-content">{{ post.content }}</p>
        </div>

      </div>
    </article>
  {% endfor %}
{% endblock content %}
```

### What's happening here

1. `{% extends "layout.html" %}` — inherit the full base layout
2. `{% block content %}...{% endblock content %}` — fill in the content block defined in `layout.html`
3. `{% for post in posts %}` — iterate over the `posts` list from the context dict
4. `{{ post.author }}`, `{{ post.title }}`, etc. — render dict values
5. `loading="lazy"` on `<img>` — browser-native lazy loading, images only load when scrolled into view

### Why `posts` is available here

The route passes it in the context:
```python
return templates.TemplateResponse(request, "home.html", {"posts": posts, "title": "home"})
```
`posts` → available as `{{ posts }}` in any template rendered with this response.

---

## 12. `url_for()` — Dynamic URL Generation in Templates

`url_for()` is a Jinja2 function that FastAPI/Starlette injects into every template automatically.

### Two uses

**1. Generate a URL to a route function (by its Python function name):**
```html
<a href="{{ url_for('home') }}">Home</a>
<!-- Renders: <a href="/">Home</a> -->

<a href="{{ url_for('get_post', post_id=1) }}">Post 1</a>
<!-- Renders: <a href="/posts/1">Post 1</a>  (when the route exists) -->
```

**2. Generate a URL to a static file:**
```html
<link rel="stylesheet" href="{{ url_for('static', path='css/main.css') }}">
<!-- Renders: <link rel="stylesheet" href="/static/css/main.css"> -->

<img src="{{ url_for('static', path='profile_pics/default.jpg') }}">
<!-- Renders: <img src="/static/profile_pics/default.jpg"> -->
```

### Why use `url_for()` instead of hardcoding paths?

| Hardcoded | `url_for()` |
|---|---|
| `href="/static/css/main.css"` | `href="{{ url_for('static', path='css/main.css') }}"` |
| Breaks if you move the static mount | Updates automatically |
| Breaks if you change the route path | Updates automatically |
| No type safety | Fails loudly if the route name doesn't exist |

📌 The `name` in `app.mount("/static", ..., name="static")` is what `url_for('static', ...)` looks up. If you changed it to `name="assets"`, you'd write `url_for('assets', path='...')`.

---

## 13. Full Updated `main.py` Walkthrough

```python
from fastapi import FastAPI, Request
from fastapi.templating import Jinja2Templates
from fastapi.staticfiles import StaticFiles

app = FastAPI()

# 1. Mount static files at /static URL prefix
app.mount("/static", StaticFiles(directory="static"), name="static")

# 2. Point Jinja2 at the templates/ directory
templates = Jinja2Templates(directory="templates")

# 3. Mock data (will be replaced with a DB later)
posts: list[dict] = [
    {
        "id": 1,
        "author": "Corey Schafer",
        "title": "FastAPI is Awesome",
        "content": "This framework is really easy to use and super fast.",
        "date_posted": "April 20, 2025",
    },
    {
        "id": 2,
        "author": "Jane Doe",
        "title": "Python is Great for Web Development",
        "content": "Python is a great language for web development, and FastAPI makes it even better.",
        "date_posted": "April 21, 2025",
    },
]

# 4. HTML routes — serve the Jinja2 template
@app.get("/", include_in_schema=False)
@app.get("/posts", include_in_schema=False)
def home(request: Request):
    return templates.TemplateResponse(request, "home.html", {"posts": posts, "title": "home"})

# 5. API route — still returns JSON
@app.post("/api/posts")
def get_posts():
    return posts
```

### Changes from Lecture 1

| What changed | Why |
|---|---|
| Removed `HTMLResponse` import | Not returning raw HTML strings anymore |
| Added `Request` import | Templates require the request object |
| Added `Jinja2Templates` import | Template engine setup |
| Added `StaticFiles` import | To serve CSS/images/JS |
| Added `app.mount(...)` | Register the static file server |
| Added `templates = Jinja2Templates(...)` | Create the reusable template renderer |
| `home()` now accepts `request: Request` | Required by `TemplateResponse` |
| Returns `TemplateResponse` instead of raw HTML string | Renders the Jinja2 template with data |

---

## 14. Bootstrap Integration

The project uses **Bootstrap 5.3.8** loaded from a CDN:

```html
<!-- CSS -->
<link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.8/dist/css/bootstrap.min.css" rel="stylesheet">

<!-- JS Bundle (includes Popper for dropdowns/tooltips) -->
<script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.8/dist/js/bootstrap.bundle.min.js"></script>
```

### Key Bootstrap 5 classes used

| Class | Purpose |
|---|---|
| `d-flex`, `align-items-start`, `gap-4` | Flexbox layout for the post card |
| `rounded-circle` | Makes the profile picture circular |
| `flex-shrink-0` | Prevents the image from shrinking |
| `flex-grow-1` | Post content takes remaining space |
| `navbar-expand-md` | Collapses navbar on mobile |
| `fixed-top` | Navbar stays at the top while scrolling |
| `d-flex flex-column min-vh-100` on `<body>` | Makes the footer stick to the bottom |
| `mt-auto` on `<footer>` | Pushes footer to the bottom (works with above) |
| `col-md-8` / `col-md-4` | 8-column content + 4-column sidebar layout |

💡 **Sticky footer pattern:** `body { display: flex; flex-direction: column; min-height: 100vh; }` + `footer { margin-top: auto; }` is the modern CSS approach. Bootstrap classes `d-flex flex-column min-vh-100` + `mt-auto` achieve exactly this.

---

## 15. Common Mistakes & Gotchas

### 1. Forgetting `request` as the first argument to `TemplateResponse`

```python
# ❌ Wrong
return templates.TemplateResponse("home.html", {"posts": posts})

# ✅ Correct (FastAPI ≥ 0.108.0)
return templates.TemplateResponse(request, "home.html", {"posts": posts})
```

### 2. Forgetting `request: Request` in the route function

```python
# ❌ Wrong — FastAPI has nothing to inject
def home():
    return templates.TemplateResponse(request, "home.html", {})

# ✅ Correct
def home(request: Request):
    return templates.TemplateResponse(request, "home.html", {})
```

### 3. Wrong `url_for()` syntax for static files

```html
<!-- ❌ Wrong -->
{{ url_for("static") ,path="css/main.css" }}

<!-- ✅ Correct -->
{{ url_for('static', path='css/main.css') }}
```

### 4. `{% extends %}` not on the first line

```html
<!-- ❌ Wrong — whitespace or HTML before extends causes errors -->
<html>
{% extends "layout.html" %}

<!-- ✅ Correct — must be the absolute first thing -->
{% extends "layout.html" %}
```

### 5. Using `| safe` on untrusted content

```html
<!-- ❌ Dangerous — XSS risk if user-controlled content -->
{{ user_bio | safe }}

<!-- ✅ Only use | safe for content you control -->
{{ hardcoded_html_snippet | safe }}
```

---

## 16. Summary & Key Takeaways

| # | Takeaway |
|---|---|
| 1 | Jinja2 templates separate HTML structure from Python logic. |
| 2 | `app.mount("/static", StaticFiles(directory="static"), name="static")` serves CSS/JS/images. |
| 3 | `Jinja2Templates(directory="templates")` creates the template renderer — do this once at module level. |
| 4 | Routes that render templates must accept `request: Request` as a parameter. |
| 5 | `templates.TemplateResponse(request, "file.html", {"key": value})` renders a template with data. |
| 6 | `{{ var }}` outputs a variable; `{% statement %}` runs logic; `{# comment #}` is ignored. |
| 7 | Template inheritance: `{% extends "layout.html" %}` + `{% block content %}` eliminates HTML repetition. |
| 8 | `url_for('static', path='css/main.css')` generates correct static file URLs dynamically. |
| 9 | `url_for('home')` generates the URL for a route by its Python function name — avoids hardcoding paths. |
| 10 | Jinja2 auto-escapes HTML in `.html` files — protection against XSS is on by default. |

---

## 17. References

- [FastAPI — Templates (Official Docs)](https://fastapi.tiangolo.com/advanced/templates/)
- [FastAPI — Static Files (Official Docs)](https://fastapi.tiangolo.com/tutorial/static-files/)
- [How to Serve a Website With FastAPI Using HTML and Jinja2 — Real Python](https://realpython.com/fastapi-jinja2-template/)
- [Primer on Jinja Templating — Real Python](https://realpython.com/primer-on-jinja-templating/)
- [Jinja2 Tutorial Part 2 — Loops and Conditionals](https://ttl255.com/jinja2-tutorial-part-2-loops-and-conditionals/)
- [Build a Dynamic HTML Frontend with FastAPI and Jinja2 — OVEX TECH](https://blog.ovexro.com/build-a-dynamic-html-frontend-with-fastapi-and-jinja2/)
- [The Ultimate FastAPI Tutorial Part 6 — Jinja Templates (Christopher Samiullah)](https://christophergs.com/tutorials/ultimate-fastapi-tutorial-pt-6-jinja-templates/)
- [Jinja2 Official Documentation](https://jinja.palletsprojects.com/en/3.1.x/)
- [Starlette — Templates](https://www.starlette.io/templates/)
- [Bootstrap 5.3 Documentation](https://getbootstrap.com/docs/5.3/)
