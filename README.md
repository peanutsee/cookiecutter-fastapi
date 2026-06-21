# FastAPI Project Structure (Cookiecutter)

A [Cookiecutter](https://cookiecutter.readthedocs.io/) template for bootstrapping FastAPI backends with a modular, layered layout. Generate a new project in seconds, then grow it feature-by-feature without fighting the folder structure.

## Quick start

### Prerequisites

- Python 3.12+
- [uv](https://docs.astral.sh/uv/) (recommended) or pip
- [Cookiecutter](https://cookiecutter.readthedocs.io/en/latest/installation.html)

```bash
pip install cookiecutter
# or: uv tool install cookiecutter
```

### Generate a project

```bash
cookiecutter gh:YOUR_USERNAME/fastapi-project-structure
```

Cookiecutter will prompt for values defined in `cookiecutter.json` (project name, slug, Python version, initial version, and so on). It then writes a new directory on disk — your generated app is ready to develop in.

To pin a specific **template** release when generating:

```bash
cookiecutter gh:YOUR_USERNAME/fastapi-project-structure --checkout v1.0.0
```

### Run the generated project

```bash
cd my-awesome-api
uv sync --extra dev
cp .env.example .env
pre-commit install
uv run uvicorn app.main:app --reload
```

---

## Folder structure

After generation, the project looks like this:

```
my-awesome-api/
├── app/
│   ├── api/
│   │   └── v1/
│   │       ├── __init__.py
│   │       └── api.py          # v1 router — aggregates all v1 endpoints
│   ├── core/
│   │   ├── db.py               # database engine, session factory, Base
│   │   ├── dependencies.py     # shared FastAPI Depends() callables
│   │   └── security.py         # auth helpers (JWT, password hashing, etc.)
│   ├── models/
│   │   └── __init__.py         # SQLAlchemy ORM models (shared across modules)
│   ├── modules/                # domain / feature modules (see below)
│   ├── config.py               # settings via pydantic-settings
│   ├── main.py                 # FastAPI app factory & lifespan
│   └── __init__.py
├── alembic/                    # database migration scripts
├── .env.example                # environment variable reference
├── .pre-commit-config.yaml     # lint/format hooks (black, ruff, mypy, …)
├── .python-version             # pyenv / uv Python pin
├── .gitignore
├── pyproject.toml              # dependencies & project metadata
└── README.md
```

### `app/` — application package

The top-level Python package. Everything importable by the running server lives here.

| Path | Purpose |
|------|---------|
| `main.py` | Creates the `FastAPI` instance, registers middleware, mounts routers, and defines startup/shutdown lifespan hooks. Keep this thin — wire things together, don't put business logic here. |
| `config.py` | Centralised settings loaded from environment variables (via `pydantic-settings`). One place to read `DATABASE_URL`, `SECRET_KEY`, feature flags, etc. |
| `api/v1/` | **HTTP routing layer** for API version 1. `api.py` is the v1 aggregator router; individual modules register their routers here. When you need a breaking API change, add `api/v2/` alongside v1. |
| `core/` | **Shared infrastructure** used across the whole app — not tied to any single domain. Database sessions, security primitives, and reusable `Depends()` helpers belong here. |
| `models/` | **Shared ORM models** (SQLAlchemy) that span multiple modules or represent cross-cutting tables (e.g. `User`). Module-specific models can live inside the module instead if they are never imported elsewhere. |
| `modules/` | **Domain / feature modules** — the heart of the app. Each module is a self-contained vertical slice (see below). |

### `app/modules/` — feature modules

Each business domain gets its own sub-package under `modules/`. A typical module looks like:

```
app/modules/users/
├── __init__.py
├── router.py       # FastAPI APIRouter with HTTP endpoints
├── service.py      # business logic (no HTTP awareness)
├── schemas.py      # Pydantic request/response models
└── models.py       # module-local ORM models (optional)
```

**Why modules?** As the API grows, a flat `routers/` + `services/` tree becomes hard to navigate. Colocating everything for "users" in one folder makes it obvious where to add code, what to delete when a feature is removed, and what to open when debugging a user-related bug.

**Rules of thumb:**

- `router.py` calls `service.py`; services never import routers.
- Cross-module calls go through services, not by reaching into another module's internals.
- Register each module's router in `app/api/v1/api.py`.

### `alembic/` — database migrations

Holds Alembic migration scripts. Initialise with `alembic init` (or ship a pre-configured `alembic.ini` in the template) and point it at `app/core/db.py` for your `Base.metadata`.

### Root files

| File | Purpose |
|------|---------|
| `pyproject.toml` | Project metadata, dependency pins, and tool config (ruff, pytest, etc.). |
| `.env.example` | Documents required environment variables; copy to `.env` locally. Never commit `.env`. |
| `.python-version` | Pins the Python interpreter for pyenv and uv. |
| `.pre-commit-config.yaml` | Git hooks for formatting and linting (autoflake, pyupgrade, black, ruff, mypy). Run `pre-commit install` once after generating. |

### Pre-commit hooks

Generated projects ship with [pre-commit](https://pre-commit.com/) hooks based on [cookiecutter-fastapi](https://github.com/Tobi-De/cookiecutter-fastapi). After generating:

```bash
uv sync --extra dev
pre-commit install        # install git hooks
pre-commit run --all-files  # run on existing files
```

Hooks included: `check-toml`, `end-of-file-fixer`, `trailing-whitespace`, `autoflake`, `pyupgrade`, `black`, `reorder-python-imports`, `ruff`, and `mypy`.

---

## How to customise

### At generation time (Cookiecutter prompts)

Edit `cookiecutter.json` at the template root to control what users are asked when they run `cookiecutter`. Common variables:

```json
{
  "project_name": "My Awesome API",
  "project_slug": "{{ cookiecutter.project_name.lower().replace(' ', '-').replace('_', '-') }}",
  "project_description": "A FastAPI backend",
  "author_name": "Your Name",
  "author_email": "you@example.com",
  "python_version": "3.12",
  "project_version": "0.1.0",
  "use_postgres": ["yes", "no"],
  "use_redis": ["yes", "no"]
}
```

Use Jinja2 syntax in template files to substitute these values:

```toml
# pyproject.toml
[project]
name = "{{ cookiecutter.project_slug }}"
version = "{{ cookiecutter.project_version }}"
requires-python = ">={{ cookiecutter.python_version }}"
```

```python
# app/config.py
class Settings(BaseSettings):
    app_name: str = "{{ cookiecutter.project_name }}"
```

Conditional blocks let you include or exclude files:

```jinja
{% if cookiecutter.use_postgres == "yes" %}
DATABASE_URL: str = "postgresql+asyncpg://..."
{% else %}
DATABASE_URL: str = "sqlite+aiosqlite:///./app.db"
{% endif %}
```

### After generation (in the new project)

| Goal | Where to change |
|------|-----------------|
| Add a new API endpoint | Create or extend a module under `app/modules/`, register its router in `app/api/v1/api.py` |
| Add a config variable | `app/config.py` + `.env.example` |
| Add a DB table | Model in `app/models/` or `app/modules/<name>/models.py`, then `alembic revision --autogenerate` |
| Bump API version | Add `app/api/v2/`, mount alongside v1 in `main.py` |
| Add a dependency | `pyproject.toml`, then `uv sync` |

### Customising the template itself

1. Edit files in this repo (the template).
2. Test locally without pushing:

   ```bash
   cookiecutter /path/to/fastapi-project-structure --no-input
   ```

3. Tag a release (see versioning below).
4. Users pull the new tag with `--checkout vX.Y.Z`.

---

## Versioning

There are **three separate version streams** to manage. Keeping them distinct avoids confusion.

### 1. Template version (this repo)

The version of the Cookiecutter template itself. Users choose which template generation they want.

**How to manage:**

- Tag releases on this repo: `git tag v1.0.0 && git push origin v1.0.0`
- Document breaking template changes in a `CHANGELOG.md`
- Users generate from a specific tag:

  ```bash
  cookiecutter gh:YOUR_USERNAME/fastapi-project-structure --checkout v1.0.0
  ```

The template's own version can live in `pyproject.toml` at the template root (for your tooling) or simply in git tags — it does **not** need to match generated project versions.

### 2. Generated project version

The `version` field in the **new project's** `pyproject.toml`. Set via a Cookiecutter variable:

```json
{
  "project_version": "0.1.0"
}
```

```toml
# {{cookiecutter.project_slug}}/pyproject.toml
version = "{{ cookiecutter.project_version }}"
```

After generation, the developer bumps this with their own release process (manual edit, `uv version`, or CI). The template only sets the **initial** value.

### 3. Dependency versions

Python package versions pinned in the generated `pyproject.toml`. Two common strategies:

**Pinned in the template (recommended for stability):**

```toml
dependencies = [
    "fastapi>=0.115.0,<0.116.0",
    "uvicorn[standard]>=0.32.0,<0.33.0",
]
```

Update pins when you cut a new template release. Everyone who generates from `v1.1.0` gets the same dependency set.

**Prompted via Cookiecutter (for flexibility):**

```json
{
  "fastapi_version": "0.115.0",
  "uvicorn_version": "0.32.0"
}
```

```toml
"fastapi=={{ cookiecutter.fastapi_version }}",
```

This adds prompt fatigue; prefer pinning in the template and bumping on template releases unless you have a strong reason to let users choose.

### Versioning checklist

| What | Where | Who bumps it |
|------|-------|--------------|
| Template release | Git tag on this repo (`v1.0.0`) | Template maintainer |
| Generated app version | `pyproject.toml` in new project | Project developer |
| Python version | `cookiecutter.json` → `.python-version` + `requires-python` | Template maintainer |
| Library pins | Template `pyproject.toml` | Template maintainer (per template release) |

### Template repo layout

```
fastapi-project-structure/          ← template repo root
├── cookiecutter.json               # prompts & default values
├── README.md                       # this file (template docs)
└── {{cookiecutter.project_slug}}/  # everything that gets copied
    ├── app/
    ├── alembic/
    ├── .pre-commit-config.yaml
    ├── pyproject.toml
    └── ...
```

Files that should **not** be templated (e.g. binary assets) can be listed in `_copy_without_render` inside `cookiecutter.json`.

---

## Local development of the template

```bash
# Generate a test project with defaults
cookiecutter . --no-input -o /tmp

# Generate with overrides
cookiecutter . --no-input project_name="Test API" project_slug="test-api" -o /tmp

# Inspect the output
ls /tmp/test-api
```

---

## License

MIT (or your chosen license — add a `LICENSE` file).
