# Lecture 01 — Getting Started: Web App + REST API

> **Source:** [Python FastAPI Tutorial (Part 1): Getting Started — Corey Schafer](https://www.youtube.com/watch?v=7AMjmCTumuo)
> **Playlist:** [FastAPI Tutorials — Corey Schafer](https://www.youtube.com/playlist?list=PL-osiE80TeTsak-c-QsVeg0YYG_0TeyXI)

---

## Table of Contents

1. [What is FastAPI?](#1-what-is-fastapi)
2. [Why FastAPI over Flask / Django?](#2-why-fastapi-over-flask--django)
3. [WSGI vs ASGI — The Engine Under the Hood](#3-wsgi-vs-asgi--the-engine-under-the-hood)
4. [Key Features at a Glance](#4-key-features-at-a-glance)
5. [Installation & Project Setup](#5-installation--project-setup)
6. [Creating Your First FastAPI App](#6-creating-your-first-fastapi-app)
7. [Path Operations & HTTP Methods](#7-path-operations--http-methods)
8. [Running the Development Server](#8-running-the-development-server)
9. [Automatic Interactive Docs](#9-automatic-interactive-docs)
10. [Returning JSON vs HTML Responses](#10-returning-json-vs-html-responses)
11. [What's in `main.py` so far](#11-whats-in-mainpy-so-far)
12. [Summary & Key Takeaways](#12-summary--key-takeaways)
13. [References](#13-references)

---

## 1. What is FastAPI?

**FastAPI** is a modern, high-performance Python web framework for building APIs, released in 2018 by Sebastián Ramírez (tiangolo).

```
FastAPI = Starlette (web layer) + Pydantic (data validation layer)
```

- **Starlette** handles the low-level web stuff: routing, middleware, WebSockets, requests/responses.
- **Pydantic** handles data validation, serialization, and schema generation using Python type hints.

FastAPI itself **doesn't talk HTTP directly** — it is an ASGI application that receives structured dictionaries from an ASGI server (like Uvicorn), processes them, and returns response dictionaries.

📌 **FastAPI is not a framework that replaces your whole stack** — it is a focused API/web framework. For a full-stack app, you still bring your own frontend (Jinja2 templates, React, etc.) and database layer.

---

## 2. Why FastAPI over Flask / Django?

| Feature | Flask | Django | **FastAPI** |
|---|---|---|---|
| Type | Micro-framework | Full-stack | API/Web framework |
| Interface | WSGI | WSGI + ASGI (v3+) | **ASGI only** |
| Async support | Limited (via Quart) | Partial | **Native, first-class** |
| Data validation | Manual | Django Forms/Serializers | **Pydantic (automatic)** |
| Auto API docs | No | No (drf-yasg addon) | **Built-in Swagger + ReDoc** |
| Performance | Moderate | Moderate | **Comparable to Node.js / Go** |
| Type hints | Optional | Optional | **Central to how it works** |
| Learning curve | Low | High | Low-Medium |

> **When to choose FastAPI:**
> - Building REST APIs or microservices
> - You want async concurrency without added complexity
> - You want automatic docs, validation, and serialization out of the box
>
> **When to choose Django:**
> - You need a full-stack app with admin panel, ORM, auth — all batteries included
>
> **When to choose Flask:**
> - Tiny projects or scripts that need a quick HTTP server; minimal deps

---

## 3. WSGI vs ASGI — The Engine Under the Hood

Understanding this distinction explains *why* FastAPI is fast.

### WSGI — Web Server Gateway Interface (Legacy)

```
Browser → Web Server (nginx) → WSGI Server (Gunicorn) → Your Flask/Django App
                                     ↑
                              Synchronous: one request
                              blocks the thread until done
```

- Python standard since PEP 3333 (2003)
- **Synchronous** — each request occupies a thread/process until fully handled
- Cannot natively handle WebSockets or HTTP/2
- Works fine for traditional request-response apps with low concurrency

### ASGI — Asynchronous Server Gateway Interface (Modern)

```
Browser → Web Server (nginx) → ASGI Server (Uvicorn) → Your FastAPI App
                                      ↑
                              Async event loop: thousands of
                              concurrent connections on one thread
```

- Python standard introduced by Django Channels (formalized in 2019)
- **Asynchronous** — non-blocking I/O; while one request waits for a DB query, the server handles other requests
- Natively supports WebSockets, HTTP/2, long-polling, Server-Sent Events
- FastAPI is **ASGI-only**

### The Uvicorn ASGI Server

**Uvicorn** is the recommended ASGI server for FastAPI in development and production.

```bash
# Under the hood, fastapi dev runs:
uvicorn main:app --reload
```

| Server | Role | Protocol |
|---|---|---|
| Uvicorn | ASGI server (async I/O) | ASGI |
| Gunicorn | Process manager (multi-worker) | WSGI (+ UvicornWorker for ASGI) |
| nginx | Reverse proxy / static files | HTTP |

💡 **Production setup:** `Gunicorn + UvicornWorker` gives you multi-process (CPU parallelism via Gunicorn) + async I/O per worker (via Uvicorn). In development, just Uvicorn alone is fine.

---

## 4. Key Features at a Glance

FastAPI's headline claims — and they hold up:

1. **Fast to run** — Performance on par with Node.js and Go (benchmarks by TechEmpower)
2. **Fast to code** — Reduces bugs, increases developer speed significantly
3. **Fewer bugs** — Type hints + Pydantic catch errors at request time, not runtime
4. **Intuitive** — Great editor support (autocomplete, inline docs)
5. **Easy** — Designed to be easy to learn and use
6. **Short** — Minimal code duplication; declare a type once, get validation, serialization, and docs
7. **Robust** — Production-ready with automatic interactive docs
8. **Standards-based** — Built on OpenAPI (formerly Swagger) and JSON Schema

### What you get for free by just declaring types:

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/items/{item_id}")
def read_item(item_id: int, q: str | None = None):
    return {"item_id": item_id, "q": q}
```

From this **one function**, FastAPI automatically provides:
- ✅ URL routing (`GET /items/{item_id}`)
- ✅ Path parameter validation (`item_id` must be an integer)
- ✅ Query parameter parsing (`q` is optional string)
- ✅ Request serialization
- ✅ Response serialization (dict → JSON)
- ✅ OpenAPI schema entry
- ✅ Swagger UI documentation
- ✅ ReDoc documentation

---

## 5. Installation & Project Setup

### Using `uv` (modern Python package manager)

```bash
# Create a new project
uv init FastAPIStudy
cd FastAPIStudy

# Add FastAPI with all standard extras (includes uvicorn, pydantic, etc.)
uv add "fastapi[standard]"
```

The `[standard]` extra bundles:
- `uvicorn[standard]` — ASGI server with fast extras (uvloop, httptools)
- `pydantic` — data validation
- `python-multipart` — form data / file uploads
- `email-validator` — email type support
- `jinja2` — HTML templating
- `httpx` — async HTTP client (for testing)

### Using `pip` (classic approach)

```bash
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install "fastapi[standard]"
```

### Project layout after setup

```
FastAPIStudy/
├── main.py          ← your application code
├── pyproject.toml   ← project metadata & dependencies
├── .venv/           ← virtual environment
└── uv.lock          ← locked dependency versions
```

### `pyproject.toml` (what `uv init` generates)

```toml
[project]
name = "fastapistudy"
version = "0.1.0"
requires-python = ">=3.13"
dependencies = [
    "fastapi[standard]>=0.136.3",
]
```

---

## 6. Creating Your First FastAPI App

```python
# main.py
from fastapi import FastAPI

# Create the FastAPI application instance
app = FastAPI()

# Define a path operation
@app.get("/")
def home():
    return {"message": "Hello, World!"}
```

### Breaking it down line by line

| Line | What it does |
|---|---|
| `from fastapi import FastAPI` | Import the main FastAPI class |
| `app = FastAPI()` | Create the ASGI application instance (this is what Uvicorn serves) |
| `@app.get("/")` | Register `home` as the handler for `GET /` requests |
| `def home():` | A regular Python function (can also be `async def`) |
| `return {"message": "Hello, World!"}` | Return a dict — FastAPI auto-converts it to JSON |

💡 **`app` is the object Uvicorn imports.** When you run `uvicorn main:app`, `main` is the module name and `app` is the variable name. They can be anything as long as they match.

---

## 7. Path Operations & HTTP Methods

A **path operation** = HTTP method + URL path → handler function.

```
@app.<method>("<path>")
def handler():
    ...
```

### The four most common HTTP methods

| Method | Decorator | Semantic meaning | Example |
|---|---|---|---|
| GET | `@app.get()` | Read / retrieve data | `GET /posts` → list all posts |
| POST | `@app.post()` | Create new data | `POST /posts` → create a post |
| PUT | `@app.put()` | Replace/update existing | `PUT /posts/1` → update post #1 |
| DELETE | `@app.delete()` | Remove data | `DELETE /posts/1` → delete post #1 |

> ⚠️ These are **conventions** (REST), not enforced rules. FastAPI doesn't stop you from using GET to delete something — but don't do that.

### Multiple decorators on one function

```python
# Both routes serve the same handler
@app.get("/")
@app.get("/posts")
def home():
    return {"posts": posts}
```

📌 FastAPI stacks decorators, so the same function can handle multiple paths. This is useful for aliasing root `/` to a resource path during development.

### HTTP Response Status Codes (defaults)

FastAPI sets sensible defaults automatically:

| Method | Default Status Code | Meaning |
|---|---|---|
| GET | `200 OK` | Request succeeded |
| POST | `200 OK` | (override to `201 Created` for correct REST semantics) |
| PUT | `200 OK` | Updated successfully |
| DELETE | `200 OK` | (override to `204 No Content` when body is empty) |

```python
# Override the status code
@app.post("/posts", status_code=201)
def create_post():
    ...
```

---

## 8. Running the Development Server

### The recommended way (via `fastapi` CLI)

```bash
# With uv
uv run fastapi dev main.py

# Without uv (if installed globally)
fastapi dev main.py
```

What `fastapi dev` does:
- Starts Uvicorn with `--reload` enabled
- Watches your files for changes and restarts automatically
- Prints the local URL and docs URL

### The manual way (calling Uvicorn directly)

```bash
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

| Flag | Meaning |
|---|---|
| `main:app` | module `main`, variable `app` |
| `--reload` | auto-restart on code changes (dev only) |
| `--host 0.0.0.0` | listen on all interfaces (not just localhost) |
| `--port 8000` | port number (default is 8000) |

### Output you'll see

```
INFO:     Will watch for changes in these directories: ['/path/to/FastAPIStudy']
INFO:     Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)
INFO:     Started reloader process [12345] using WatchFiles
INFO:     Started server process [12346]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
```

> ⚠️ **Never use `--reload` in production.** It spawns a file-watcher process and has overhead. Use a process manager like `gunicorn` or `systemd` instead.

---

## 9. Automatic Interactive Docs

This is one of FastAPI's **killer features**. Without writing a single line of documentation, you get two fully interactive UIs.

### Swagger UI — `/docs`

```
http://127.0.0.1:8000/docs
```

- **Try it out** button lets you execute real API calls from the browser
- Shows request body schemas, parameters, response formats
- Generated from your code's type hints and docstrings in real time

### ReDoc — `/redoc`

```
http://127.0.0.1:8000/redoc
```

- Cleaner, read-only documentation format
- Better for sharing API docs with consumers/clients

### OpenAPI JSON schema — `/openapi.json`

```
http://127.0.0.1:8000/openapi.json
```

- Raw OpenAPI 3.x schema in JSON format
- Can be imported into Postman, Insomnia, API gateways, and auto-generates client SDKs

💡 **How it works:** FastAPI inspects every route's type annotations and docstring at startup and builds the OpenAPI schema. This means your docs are **always in sync with your code** — no manual updates ever.

### Hiding a route from docs

```python
@app.get("/", include_in_schema=False)
def home():
    ...
```

Use `include_in_schema=False` to exclude internal or redirect routes from the public API docs.

---

## 10. Returning JSON vs HTML Responses

### Default: returning a dict → JSON

```python
@app.get("/api/posts")
def get_posts():
    return posts  # FastAPI serializes this to JSON automatically
```

FastAPI's default response is `JSONResponse`. When you return a Python `dict` or `list`, FastAPI:
1. Serializes it using `jsonable_encoder`
2. Sets `Content-Type: application/json`
3. Returns it with the appropriate status code

### Returning HTML with `HTMLResponse`

```python
from fastapi.responses import HTMLResponse

@app.get("/", response_class=HTMLResponse)
def home():
    return "<h1>Hello from FastAPI!</h1>"
```

You must explicitly declare `response_class=HTMLResponse` to:
- Set `Content-Type: text/html`
- Tell FastAPI and Swagger UI this route returns HTML (not JSON)

### Other built-in response types

| Class | Content-Type | Use case |
|---|---|---|
| `JSONResponse` | `application/json` | Default — structured data |
| `HTMLResponse` | `text/html` | Rendered HTML pages |
| `PlainTextResponse` | `text/plain` | Simple text |
| `RedirectResponse` | — | HTTP redirects |
| `FileResponse` | auto-detected | Serve a file from disk |
| `StreamingResponse` | varies | Stream large data |

```python
from fastapi.responses import JSONResponse, HTMLResponse, RedirectResponse
```

---

## 11. What's in `main.py` so far

```python
from fastapi import FastAPI
from fastapi.responses import HTMLResponse

app = FastAPI()

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

# Route 1: Serves HTML at / and /posts (excluded from API docs)
@app.get("/", response_class=HTMLResponse, include_in_schema=False)
@app.get("/posts", response_class=HTMLResponse, include_in_schema=False)
def home():
    return f"<h1>{posts[0]['title']}</h1>"

# Route 2: REST API endpoint that returns JSON
@app.post("/api/posts")
def get_posts():
    return posts
```

### Notes on the current code

- `posts: list[dict]` — mock in-memory data store. Later we'll replace this with a real database.
- `@app.post("/api/posts")` returning all posts is semantically incorrect — listing posts should be `GET`. Likely a placeholder to be fixed.
- The HTML route uses `include_in_schema=False` to keep the Swagger docs clean (HTML routes aren't really part of the public API).

---

## 12. Summary & Key Takeaways

| # | Takeaway |
|---|---|
| 1 | FastAPI = Starlette + Pydantic. It is an ASGI framework. |
| 2 | ASGI is async and non-blocking; WSGI is synchronous. FastAPI requires ASGI. |
| 3 | Uvicorn is the ASGI server that actually runs FastAPI. |
| 4 | A **path operation** is a function decorated with `@app.<method>("<path>")`. |
| 5 | Returning a `dict` or `list` auto-serializes to JSON. |
| 6 | Use `response_class=HTMLResponse` to return HTML. |
| 7 | `/docs` (Swagger UI) and `/redoc` are generated automatically from your code. |
| 8 | `fastapi dev main.py` starts a hot-reloading dev server. |
| 9 | `include_in_schema=False` hides a route from API docs. |

Kyaa

---

## 13. References

- [FastAPI Official Docs — First Steps](https://fastapi.tiangolo.com/tutorial/first-steps/)
- [FastAPI Features](https://fastapi.tiangolo.com/features/)
- [ASGI vs WSGI — Complete Guide (Medium)](https://medium.com/@dynamicy/asgi-vs-wsgi-a-complete-guide-to-their-differences-and-fastapi-applications-9857f13c4521)
- [FastAPI vs Flask vs Django Comparison (Medium)](https://medium.com/@thiwankajayasiri/fastapi-flask-django-a-comprehensive-comparison-4bce6425b4ec)
- [Uvicorn Documentation](https://www.uvicorn.org/)
- [Gunicorn + Uvicorn Workers for Production (Medium)](https://medium.com/@iklobato/mastering-gunicorn-and-uvicorn-the-right-way-to-deploy-fastapi-applications-aaa06849841e)
- [Understanding Decorators in FastAPI (DEV Community)](https://dev.to/joe_/unlocking-the-magic-understanding-decorators-in-fastapi-for-beginners-5783)
- [FastAPI Path Operations (Orchestra)](https://www.getorchestra.io/guides/fast-api-path-operation-decorators-a-comprehensive-tutorial)
- [Starlette Documentation](https://www.starlette.io/)
- [Pydantic Documentation](https://docs.pydantic.dev/)
