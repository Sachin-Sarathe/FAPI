# Lecture 03 — Path Parameters: Validation and Error Handling

> **Source:** [Python FastAPI Tutorial (Part 3): Path Parameters - Validation and Error Handling — Corey Schafer](https://www.youtube.com/watch?v=WRjXIA5pMtk)
> **Playlist:** [FastAPI Tutorials — Corey Schafer](https://www.youtube.com/playlist?list=PL-osiE80TeTsak-c-QsVeg0YYG_0TeyXI)

---

## Table of Contents

1. [What are Path Parameters?](#1-what-are-path-parameters)
2. [Declaring Path Parameters with Type Hints](#2-declaring-path-parameters-with-type-hints)
3. [Automatic Type Conversion & Validation](#3-automatic-type-conversion--validation)
4. [Route Order Matters — Fixed Before Dynamic](#4-route-order-matters--fixed-before-dynamic)
5. [HTTP Status Codes — `fastapi.status`](#5-http-status-codes--fastapistatus)
6. [Raising `HTTPException`](#6-raising-httpexception)
7. [FastAPI's `HTTPException` vs Starlette's `HTTPException`](#7-fastapis-httpexception-vs-starlettes-httpexception)
8. [Custom Exception Handlers — `@app.exception_handler()`](#8-custom-exception-handlers--appexception_handler)
9. [Handling `RequestValidationError`](#9-handling-requestvalidationerror)
10. [Smart Error Routing — JSON for APIs, HTML for Browser](#10-smart-error-routing--json-for-apis-html-for-browser)
11. [New Routes Added This Lecture](#11-new-routes-added-this-lecture)
12. [New Templates — `post.html` and `error.html`](#12-new-templates--posthtml-and-errorhtml)
13. [Bug Fixed — Unused `title` Variable](#13-bug-fixed--unused-title-variable)
14. [Enum Path Parameters (Bonus)](#14-enum-path-parameters-bonus)
15. [Path Parameters Containing Slashes (Bonus)](#15-path-parameters-containing-slashes-bonus)
16. [Summary & Key Takeaways](#16-summary--key-takeaways)
17. [References](#17-references)

---

## 1. What are Path Parameters?

**Path parameters** are dynamic segments of a URL — values embedded directly inside the URL path, wrapped in `{}`.

```
GET /posts/1       →  post_id = 1
GET /posts/42      →  post_id = 42
GET /users/sachin  →  username = "sachin"
```

They identify **which specific resource** you're requesting. Contrast with:

| Parameter type | Location | Example | Use case |
|---|---|---|---|
| **Path parameter** | URL path | `/posts/{post_id}` | Identify a specific resource |
| **Query parameter** | After `?` | `/posts?page=2&limit=10` | Filter, sort, paginate |
| **Request body** | HTTP body | `{"title": "..."}` | Send complex data (POST/PUT) |

---

## 2. Declaring Path Parameters with Type Hints

```python
@app.get("/posts/{post_id}")
def get_post(post_id: int):
    ...
```

The `{post_id}` in the URL path and the `post_id: int` in the function signature are **linked by name**. FastAPI:
1. Extracts the value from the URL string (e.g., `"42"`)
2. Converts it to the declared type (`int` → `42`)
3. Passes it to your function

📌 The parameter name in `{}` **must exactly match** the function argument name. Case-sensitive.

```python
# ✅ Names match — works
@app.get("/posts/{post_id}")
def get_post(post_id: int): ...

# ❌ Names don't match — post_id in URL, pid in function
@app.get("/posts/{post_id}")
def get_post(pid: int): ...  # FastAPI treats pid as a query parameter
```

---

## 3. Automatic Type Conversion & Validation

This is where FastAPI's magic shines. All powered by **Pydantic** under the hood.

### Type conversion

```
URL string  →  FastAPI extracts  →  Pydantic converts  →  Python type in your function
  "42"      →      "42"          →       int("42")     →        42  (int)
  "3.14"    →      "3.14"        →       float("3.14") →       3.14 (float)
  "true"    →      "true"        →       bool(...)     →       True (bool)
```

### Validation on success

```
GET /api/posts/1    →  post_id = 1  ✅  function runs normally
GET /api/posts/42   →  post_id = 42 ✅  function runs normally
```

### Validation on failure — automatic `422`

```
GET /api/posts/foo
```

FastAPI automatically returns a `422 Unprocessable Entity` with a detailed error — **no code needed from you**:

```json
{
  "detail": [
    {
      "type": "int_parsing",
      "loc": ["path", "post_id"],
      "msg": "Input should be a valid integer, unable to parse string as an integer",
      "input": "foo",
      "url": "https://errors.pydantic.dev/..."
    }
  ]
}
```

The error tells the client:
- **`type`** — machine-readable error category
- **`loc`** — where the bad value came from (`["path", "post_id"]`)
- **`msg`** — human-readable explanation
- **`input`** — the exact value that failed

💡 This entire validation and error response is generated automatically. You write zero extra code for it.

### Supported path parameter types

| Type | Example URL | Received as |
|---|---|---|
| `int` | `/posts/42` | `42` |
| `float` | `/prices/9.99` | `9.99` |
| `str` | `/users/sachin` | `"sachin"` |
| `bool` | `/active/true` | `True` |
| `UUID` | `/items/550e8400-e29b-41d4-a716-446655440000` | `UUID(...)` |

---

## 4. Route Order Matters — Fixed Before Dynamic

⚠️ **This is one of the most common FastAPI beginner bugs.**

FastAPI evaluates routes **top to bottom in the order they are defined**. A dynamic route like `/{id}` will greedily match anything — including paths you intended to be fixed.

### The bug

```python
# ❌ Wrong order — /posts/latest will NEVER be reached
@app.get("/posts/{post_id}")   # defined first — matches /posts/latest with post_id="latest"
def get_post(post_id: int): ...

@app.get("/posts/latest")      # never reached — shadowed by the route above
def get_latest_post(): ...
```

When you request `GET /posts/latest`, FastAPI hits the first matching route (`/posts/{post_id}`), tries to convert `"latest"` to `int`, and returns a `422` error.

### The fix — fixed paths first

```python
# ✅ Correct order — specific routes before parameterized ones
@app.get("/posts/latest")      # defined first — matches only /posts/latest
def get_latest_post(): ...

@app.get("/posts/{post_id}")   # defined second — matches /posts/1, /posts/42, etc.
def get_post(post_id: int): ...
```

📌 **Rule:** Always declare fixed/literal paths **before** dynamic/parameterized paths that share the same prefix.

---

## 5. HTTP Status Codes — `fastapi.status`

Instead of remembering magic numbers like `404`, `422`, `201`, FastAPI provides named constants via `fastapi.status`:

```python
from fastapi import status

status.HTTP_200_OK                    # 200
status.HTTP_201_CREATED               # 201
status.HTTP_204_NO_CONTENT            # 204
status.HTTP_400_BAD_REQUEST           # 400
status.HTTP_401_UNAUTHORIZED          # 401
status.HTTP_403_FORBIDDEN             # 403
status.HTTP_404_NOT_FOUND             # 404
status.HTTP_409_CONFLICT              # 409
status.HTTP_422_UNPROCESSABLE_CONTENT # 422
status.HTTP_500_INTERNAL_SERVER_ERROR # 500
```

### Why use named constants instead of numbers?

| Raw number | `fastapi.status` constant |
|---|---|
| `404` | `status.HTTP_404_NOT_FOUND` |
| Typo `440` compiles silently | Typo `HTTP_440_NOT_FOUND` is a `AttributeError` immediately |
| No IDE autocomplete | Full IDE autocomplete |
| Harder to read | Self-documenting |

```python
# ❌ Magic number — easy to mistype, hard to read
raise HTTPException(status_code=404, detail="Not found")

# ✅ Named constant — readable, type-safe, IDE-friendly
raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="Post not found")
```

### Quick HTTP status code reference

| Range | Category | Common codes |
|---|---|---|
| `2xx` | Success | `200 OK`, `201 Created`, `204 No Content` |
| `3xx` | Redirection | `301 Moved Permanently`, `302 Found` |
| `4xx` | Client error | `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `422 Unprocessable Entity` |
| `5xx` | Server error | `500 Internal Server Error`, `503 Service Unavailable` |

---

## 6. Raising `HTTPException`

`HTTPException` is FastAPI's standard way to return HTTP error responses from your route functions.

```python
from fastapi import FastAPI, HTTPException, status

@app.get("/api/posts/{post_id}")
def get_post(post_id: int):
    for post in posts:
        if post.get("id") == post_id:
            return post
    raise HTTPException(
        status_code=status.HTTP_404_NOT_FOUND,
        detail="Post not found"
    )
```

### Key points

- You **`raise`** it, never `return` it. Raising immediately stops function execution and sends the error response.
- `status_code` — the HTTP status code to return
- `detail` — the error message; can be a `str`, `dict`, `list`, or any JSON-serializable value
- Optionally add custom `headers`:
  ```python
  raise HTTPException(
      status_code=status.HTTP_404_NOT_FOUND,
      detail="Post not found",
      headers={"X-Error": "Resource missing"},
  )
  ```

### Default JSON error response

FastAPI wraps the `detail` in a `{"detail": ...}` envelope:

```json
// GET /api/posts/999
{
  "detail": "Post not found"
}
```

The `detail` field can be anything JSON-serializable:

```python
raise HTTPException(
    status_code=400,
    detail={"error": "invalid_id", "received": post_id, "expected": "positive integer"}
)
```

---

## 7. FastAPI's `HTTPException` vs Starlette's `HTTPException`

This is a subtle but important distinction.

```python
from fastapi import HTTPException                         # FastAPI's version
from starlette.exceptions import HTTPException as StarletteHTTPException  # Starlette's version
```

| | FastAPI `HTTPException` | Starlette `HTTPException` |
|---|---|---|
| `detail` field | Any JSON-serializable value | Strings only |
| Inherits from | Starlette's `HTTPException` | `Exception` |
| Used in your routes? | ✅ Yes | Generally no |
| Register handler for? | ❌ No — use Starlette's | ✅ Yes |

### Why register the handler on Starlette's version?

FastAPI's `HTTPException` **inherits** from Starlette's. But Starlette's internal code (middleware, routing) can raise Starlette's version directly — not FastAPI's. If you only handle FastAPI's version, you'll miss those.

```python
# ✅ Correct — handles BOTH FastAPI's and Starlette's HTTPException
from starlette.exceptions import HTTPException as StarletteHTTPException

@app.exception_handler(StarletteHTTPException)
def http_exception_handler(request: Request, exception: StarletteHTTPException):
    ...
```

---

## 8. Custom Exception Handlers — `@app.exception_handler()`

FastAPI has built-in default handlers for `HTTPException` (returns JSON) and `RequestValidationError` (returns JSON with validation errors). You can **override** these with your own handlers to customize the error response format.

```python
from fastapi import Request
from fastapi.responses import JSONResponse
from starlette.exceptions import HTTPException as StarletteHTTPException

@app.exception_handler(StarletteHTTPException)
def general_http_exception_handler(request: Request, exception: StarletteHTTPException):
    message = (
        exception.detail
        if exception.detail
        else "An error occurred. Please check your request and try again."
    )

    if request.url.path.startswith("/api"):
        return JSONResponse(
            status_code=exception.status_code,
            content={"detail": message},
        )
    return templates.TemplateResponse(
        request,
        "error.html",
        {
            "status_code": exception.status_code,
            "title": exception.status_code,
            "message": message,
        },
        status_code=exception.status_code,
    )
```

### How the handler works

The handler function receives two arguments:
- `request: Request` — the incoming HTTP request that caused the error
- `exception: StarletteHTTPException` — the raised exception, with `.status_code` and `.detail`

It must return a **Response object** — either `JSONResponse`, `TemplateResponse`, `PlainTextResponse`, etc.

### Anatomy of the `exception` object

```python
exception.status_code  # int — e.g. 404
exception.detail       # str or None — e.g. "Post not found"
exception.headers      # dict or None — custom headers set when raising
```

---

## 9. Handling `RequestValidationError`

`RequestValidationError` is raised by FastAPI/Pydantic when incoming request data fails type validation — e.g., sending `"foo"` where an `int` is expected.

```python
from fastapi.exceptions import RequestValidationError
from fastapi import status

@app.exception_handler(RequestValidationError)
def validation_exception_handler(request: Request, exception: RequestValidationError):
    if request.url.path.startswith("/api"):
        return JSONResponse(
            status_code=status.HTTP_422_UNPROCESSABLE_CONTENT,
            content={"detail": exception.errors()},
        )
    return templates.TemplateResponse(
        request,
        "error.html",
        {
            "status_code": status.HTTP_422_UNPROCESSABLE_CONTENT,
            "title": status.HTTP_422_UNPROCESSABLE_CONTENT,
            "message": "Invalid request. Please check your input and try again.",
        },
        status_code=status.HTTP_422_UNPROCESSABLE_CONTENT,
    )
```

### `exception.errors()` — the validation detail

Returns a list of all validation failures:

```json
[
  {
    "type": "int_parsing",
    "loc": ["path", "post_id"],
    "msg": "Input should be a valid integer, unable to parse string as an integer",
    "input": "foo"
  }
]
```

### `RequestValidationError` vs `HTTPException`

| | `HTTPException` | `RequestValidationError` |
|---|---|---|
| Triggered by | Your code calling `raise HTTPException(...)` | FastAPI/Pydantic failing to parse/validate request data |
| Default status | Whatever you set | `422 Unprocessable Entity` |
| When it happens | Business logic errors (not found, forbidden) | Bad input format errors |
| Example | Post ID `999` doesn't exist | Path param `"foo"` can't be converted to `int` |

---

## 10. Smart Error Routing — JSON for APIs, HTML for Browser

The most important pattern introduced in this lecture: **the same error, two different response formats** depending on who's asking.

```python
if request.url.path.startswith("/api"):
    # API client (Postman, JS fetch, curl) — return machine-readable JSON
    return JSONResponse(
        status_code=exception.status_code,
        content={"detail": message},
    )

# Browser request — return human-readable HTML error page
return templates.TemplateResponse(
    request,
    "error.html",
    {"status_code": exception.status_code, "message": message},
    status_code=exception.status_code,
)
```

### Why this matters

| Requester | Wants | Why |
|---|---|---|
| Browser (`/posts/999`) | HTML error page | Shows a styled "404 Not Found" page to the user |
| API client (`/api/posts/999`) | JSON error body | The client code needs to parse the error programmatically |

The `request.url.path.startswith("/api")` check is the branching logic. All your API routes are prefixed with `/api/`, and all browser-facing routes are not.

```
GET /posts/999      → path starts with /posts → return error.html (HTML)
GET /api/posts/999  → path starts with /api   → return JSON error
```

This is a clean **content negotiation** pattern. A more advanced alternative is checking the `Accept` header:
```python
if "application/json" in request.headers.get("accept", ""):
    # return JSON
```

But the `/api` prefix approach is simpler and completely reliable when you control the URL structure.

---

## 11. New Routes Added This Lecture

### Individual post page — HTML

```python
@app.get("/posts/{post_id}", include_in_schema=False)
def post_page(request: Request, post_id: int):
    for post in posts:
        if post.get("id") == post_id:
            title = post['title'][:50]
            return templates.TemplateResponse(
                request,
                "post.html",
                {"post": post, "title": title}
            )
    raise HTTPException(
        status_code=status.HTTP_404_NOT_FOUND,
        detail="Post not Found"
    )
```

- `post_id: int` — FastAPI validates and converts the path segment automatically
- Loops through the in-memory `posts` list to find a match
- `title = post['title'][:50]` — truncates the title to 50 characters for the `<title>` tag
- If not found → `raise HTTPException(404)` → caught by the global handler → rendered as `error.html`
- `include_in_schema=False` — hides this HTML route from Swagger docs

### Individual post — JSON API

```python
@app.get("/api/posts/{post_id}")
def get_post(post_id: int):
    for post in posts:
        if post.get("id") == post_id:
            return post
    raise HTTPException(
        status_code=status.HTTP_404_NOT_FOUND,
        detail="Post not Found"
    )
```

### Full route table after Lecture 3

| Method | Path | Returns | Description |
|---|---|---|---|
| `GET` | `/` | HTML | Home page |
| `GET` | `/posts` | HTML | Home page alias |
| `GET` | `/posts/{post_id}` | HTML | Single post page |
| `GET` | `/api/posts` | JSON | List all posts |
| `GET` | `/api/posts/{post_id}` | JSON | Get single post |

---

## 12. New Templates — `post.html` and `error.html`

### `error.html` — generic error page

```html
{% extends "layout.html" %}
{% block content %}
  <article class="content-section py-3 px-4 mb-4">
    <h1>
      <a class="article-title" href="#">Oops... {{ status_code }} Error</a>
    </h1>
    <p class="article-content">{{ message }}</p>
  </article>
{% endblock content %}
```

Receives `status_code` and `message` from the context — both set by the exception handlers. Works for any HTTP error: 404, 422, 500, etc.

### `post.html` — single post detail page

Key additions over `home.html`:
- Displays a single `post` object (not a list)
- Shows **Edit** and **Delete** buttons (placeholders — auth not implemented yet)
- Delete button opens a **Bootstrap modal** confirmation dialog

```html
<!-- Delete modal trigger -->
<button data-bs-toggle="modal" data-bs-target="#deleteModal">Delete Post</button>

<!-- The modal itself (defined further down in the template) -->
<div class="modal fade" id="deleteModal" ...>
  <div class="modal-dialog">
    <div class="modal-content">
      <div class="modal-header">
        <h5 class="modal-title">Delete Post?</h5>
      </div>
      <form action="#">
        <div class="modal-body">
          Are you sure you want to delete this post? This action cannot be undone.
        </div>
        <div class="modal-footer">
          <button data-bs-dismiss="modal">Cancel</button>
          <button type="submit" class="btn btn-outline-danger">Delete</button>
        </div>
      </form>
    </div>
  </div>
</div>
```

How Bootstrap modals work:
- `data-bs-toggle="modal"` on the trigger button tells Bootstrap to intercept the click
- `data-bs-target="#deleteModal"` identifies which modal to open (by `id`)
- `data-bs-dismiss="modal"` on Cancel closes it
- No JavaScript needed — Bootstrap handles it entirely via `data-*` attributes

---

## 13. Bug Fixed — Unused `title` Variable

The Pylance warning `"title" is not accessed` on line 46 pointed to this bug:

```python
# ❌ Before — title computed but hardcoded "home" passed to template
title = post['title'][:50]
return templates.TemplateResponse(request, "post.html", {"post": post, "title": "home"})
#                                                                         ^^^^^^^
#                                              This should use the variable, not "home"

# ✅ After — title variable correctly passed to template
title = post['title'][:50]
return templates.TemplateResponse(request, "post.html", {"post": post, "title": title})
```

**Impact:** Without this fix, every post page would show `"FastAPI Blog - home"` in the browser tab instead of the actual post title. The `{% if title %}` block in `layout.html` would render `FastAPI Blog - home` for every post.

---

## 14. Enum Path Parameters (Bonus)

When a path parameter should only accept a fixed set of values, use Python's `Enum`:

```python
from enum import Enum
from fastapi import FastAPI

class PostStatus(str, Enum):
    draft = "draft"
    published = "published"
    archived = "archived"

@app.get("/posts/status/{status}")
def get_posts_by_status(status: PostStatus):
    return {"status": status, "posts": [...]}
```

Benefits:
- FastAPI validates the value is one of the allowed options — anything else returns `422`
- Swagger UI shows a **dropdown** of allowed values instead of a free-text input
- Inheriting from `str` means the enum value serializes to a string in JSON responses

```
GET /posts/status/published  → ✅ works
GET /posts/status/deleted    → ❌ 422 Unprocessable Entity
```

---

## 15. Path Parameters Containing Slashes (Bonus)

Normally a path parameter can't contain `/` because that would be interpreted as a URL separator. Use the `:path` type annotation to allow it:

```python
@app.get("/files/{file_path:path}")
def read_file(file_path: str):
    return {"file_path": file_path}
```

```
GET /files/documents/reports/q4.pdf  →  file_path = "documents/reports/q4.pdf"
```

> ⚠️ OpenAPI doesn't formally support this pattern, so Swagger UI can't document it properly. Use with caution.

---

## 16. Summary & Key Takeaways

| # | Takeaway |
|---|---|
| 1 | Path parameters are declared with `{name}` in the URL and `name: type` in the function. |
| 2 | FastAPI auto-converts the URL string to the declared Python type via Pydantic. |
| 3 | Invalid types return `422 Unprocessable Entity` automatically — no extra code needed. |
| 4 | Fixed routes (`/posts/latest`) must be defined **before** dynamic routes (`/posts/{id}`). |
| 5 | Use `fastapi.status` constants (`HTTP_404_NOT_FOUND`) instead of raw integer codes. |
| 6 | `raise HTTPException(status_code=..., detail=...)` — use `raise`, not `return`. |
| 7 | Register error handlers on `StarletteHTTPException` to catch all HTTP errors, including Starlette internals. |
| 8 | `@app.exception_handler(ExceptionType)` registers a global handler for that exception type. |
| 9 | `RequestValidationError` fires on bad input format; `HTTPException` fires on business logic errors. |
| 10 | Check `request.url.path.startswith("/api")` to decide between JSON and HTML error responses. |

---

## 17. References

- [FastAPI — Path Parameters (Official Docs)](https://fastapi.tiangolo.com/tutorial/path-params/)
- [FastAPI — Handling Errors (Official Docs)](https://fastapi.tiangolo.com/tutorial/handling-errors/)
- [FastAPI — Status Codes Reference](https://fastapi.tiangolo.com/reference/status/)
- [FastAPI Error Handling: Types, Methods, and Best Practices — Honeybadger](https://www.honeybadger.io/blog/fastapi-error-handling/)
- [Understanding Path Parameters and Route Order in FastAPI — Medium](https://medium.com/@eastlight90KR/understanding-path-parameters-and-route-order-in-fastapi-a-practical-guide-7c77b06969a7)
- [FastAPI Path Parameters Comprehensive Guide — CodingEasyPeasy](https://www.codingeasypeasy.com/blog/fastapi-path-parameters-a-comprehensive-guide-with-examples)
- [FastAPI Error Handling and Custom Exception Handlers — TheCodeForge](https://thecodeforge.io/python/fastapi-error-handling/)
- [Pydantic Validation Errors Documentation](https://docs.pydantic.dev/latest/errors/errors/)
- [HTTP Status Codes — MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)
- [Bootstrap 5.3 — Modal Component](https://getbootstrap.com/docs/5.3/components/modal/)
