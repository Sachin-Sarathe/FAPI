# FastAPI Study — CLAUDE.md

## Project Purpose

This repository is a personal learning project for **FastAPI**. The goal is to build progressively complex FastAPI applications while following a structured YouTube playlist, with detailed notes stored alongside the code.

## Project Structure

```
FastAPIStudy/
├── main.py              # Main FastAPI application (evolves with each lecture)
├── LectureNotes/        # Detailed notes for each lecture
│   └── Lecture_XX_<topic>.md
├── pyproject.toml       # Project dependencies (managed with uv)
├── .venv/               # Virtual environment
└── CLAUDE.md            # This file
```

## How Lecture Notes Work

When the user provides a YouTube link from the playlist:

1. **Fetch the video** — use WebSearch or WebFetch to retrieve the transcript, title, and description of the video.
2. **Identify the topic** — determine which FastAPI concepts are covered.
3. **Write comprehensive notes** in `LectureNotes/Lecture_XX_<topic>.md` using:
   - Clear headings and subheadings
   - Code examples with syntax highlighting (Python)
   - Symbols for callouts: `>` blockquotes, `⚠️` warnings, `💡` tips, `📌` key points, `🔗` references
   - External knowledge beyond the video — pull from FastAPI docs, Python docs, RFCs, blog posts, and other authoritative sources to enrich explanations
   - Citations/references at the bottom of each note file
4. **Keep notes beginner-friendly but technically deep** — explain the *why*, not just the *what*.

## Tech Stack

| Tool | Version | Purpose |
|------|---------|---------|
| Python | 3.13 | Language |
| FastAPI | ≥0.136.3 | Web framework |
| uv | latest | Package/env manager |
| Uvicorn | bundled via `fastapi[standard]` | ASGI server |

## Running the App

```bash
uv run fastapi dev main.py
```

## Note Naming Convention

```
Lecture_01_Introduction_and_Setup.md
Lecture_02_Path_Parameters.md
Lecture_03_Query_Parameters.md
...
```

## Learning Goals

- [ ] Core FastAPI concepts (routing, parameters, request bodies)
- [ ] Pydantic models and data validation
- [ ] Dependency injection
- [ ] Authentication & security
- [ ] Database integration (SQLAlchemy / SQLModel)
- [ ] Background tasks
- [ ] Testing FastAPI applications
- [ ] Deployment
