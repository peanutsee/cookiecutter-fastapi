# {{ cookiecutter.project_name }}

{{ cookiecutter.project_description }}

## Setup

```bash
uv sync --extra dev
cp .env.example .env
pre-commit install
```

## Run

```bash
uv run uvicorn app.main:app --reload
```

## Pre-commit

Hooks run automatically on `git commit`. To run manually:

```bash
uv run pre-commit run --all-files
```
