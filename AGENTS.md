# AGENTS.md — Paperless-ngx

> A community-supported supercharged document management system: scan, index and archive all your physical documents.

## Project Overview

- **Version**: 2.20.x
- **Python**: >=3.11 (tested on 3.11, 3.12, 3.13, 3.14)
- **Node**: 24.x
- **Package manager (backend)**: [uv](https://github.com/astral-sh/uv) (>=0.9.0)
- **Package manager (frontend)**: pnpm (10.x)
- **Framework (backend)**: Django 5.2 + Django REST Framework + Celery + Channels (WebSockets)
- **Framework (frontend)**: Angular 21
- **Database**: PostgreSQL, MariaDB, or SQLite
- **Task queue**: Celery with Redis
- **API schema**: OpenAPI via drf-spectacular (available at `/api/schema/`)

## Repository Structure

```
├── src/                      # Python backend (Django project root)
│   ├── documents/            # Main Django app: models, views, serializers, tasks
│   ├── paperless/            # Core Django project: settings, urls, ASGI/WSGI, parsers
│   │   ├── settings/         # Django settings (package, loads from paperless.conf)
│   │   └── parsers/          # Built-in document parsers (tesseract, tika, mail, text, remote)
│   ├── paperless_mail/       # Mail processing Django app
│   ├── paperless_ai/         # AI/LLM integration (classification, chat, embeddings, indexing)
│   ├── manage.py             # Django management entry point (run from repo root via `uv run`)
│   └── locale/               # Backend translation files (.po)
├── src-ui/                   # Angular frontend
│   ├── src/app/
│   │   ├── components/       # Angular components (admin, chat, dashboard, document-*, manage, etc.)
│   │   ├── services/         # Angular services (REST, config, WebSocket, toast, etc.)
│   │   │   └── rest/         # REST API service classes extending AbstractPaperlessService
│   │   ├── data/             # TypeScript data models/interfaces
│   │   ├── pipes/            # Angular pipes
│   │   ├── interceptors/     # HTTP interceptors (auth, CSRF, API version)
│   │   ├── guards/           # Route guards (permissions, dirty forms)
│   │   ├── directives/       # Angular directives (permissions, sortable)
│   │   └── utils/            # Utility functions
│   ├── e2e/                  # Playwright E2E tests
│   ├── locale/               # Frontend translation files (.xlf)
│   └── messages.xlf          # Source strings for i18n extraction
├── docs/                     # Documentation (built with Zensical)
├── docker/                   # Docker build files and compose configs
├── scripts/                  # Service scripts and helpers
├── .devcontainer/            # VS Code devcontainer config
├── pyproject.toml            # Python project config, ruff, pytest, mypy, coverage settings
├── Dockerfile                # Multi-stage Docker build
└── paperless.conf.example    # Example configuration file
```

## Essential Commands

### Backend Setup

```bash
# Install all dependencies (dev, testing, lint, docs)
uv sync --group dev

# Install testing dependencies only
uv sync --group testing

# Install pre-commit hooks
uv run prek install
```

### Backend Development

```bash
# Run Django dev server (from repo root)
uv run src/manage.py runserver

# Run document consumer
uv run src/manage.py document_consumer

# Run Celery worker
uv run celery --app paperless worker -l DEBUG

# Create superuser
uv run src/manage.py createsuperuser

# Run migrations
uv run src/manage.py migrate
```

### Backend Testing

```bash
# Run all backend tests
uv run pytest

# Run a specific test file
uv run pytest src/documents/tests/test_api_documents.py

# Run with specific marker
uv run pytest -m "not live"

# Run a single test class/method
uv run pytest src/documents/tests/test_api_documents.py::TestDocumentApi::testDocuments
```

**pytest configuration** (from `pyproject.toml`):
- `DJANGO_SETTINGS_MODULE = "paperless.settings"`
- Parallel execution: `--numprocesses=auto --dist=loadscope`
- Coverage enabled by default (HTML + XML reports)
- Test paths: `src/documents/tests/`, `src/paperless/tests/`, `src/paperless_mail/tests/`, `src/paperless_ai/tests/`

### Backend Linting & Formatting

```bash
# Run pre-commit hooks (ruff check, ruff format, prettier, codespell, etc.)
uv run prek run --all-files

# Run ruff directly
uv run ruff check src/
uv run ruff format src/

# Type checking
uv run mypy src/ | uv run mypy-baseline filter   # mypy with baseline
uv run pyrefly check src/                          # pyrefly (alternative)
```

### Frontend Setup

```bash
cd src-ui
pnpm install
```

### Frontend Development

```bash
cd src-ui
pnpm ng serve              # Dev server at http://localhost:4200 (proxies to :8000)
pnpm ng build --configuration production
```

### Frontend Testing

```bash
cd src-ui
pnpm run test              # Jest unit tests (no-watch, with coverage)
pnpm ng test               # Jest unit tests (watch mode)
pnpm exec playwright test  # Playwright E2E tests
pnpm run lint              # ESLint
```

### Frontend i18n

```bash
cd src-ui
pnpm ng extract-i18n       # Extract new/changed strings to messages.xlf
```

### Documentation

```bash
uv run zensical build       # Build docs
uv run zensical serve       # Serve docs with live reload at http://127.0.0.1:8000
```

### Docker

```bash
docker build --file Dockerfile --tag paperless:local .
```

## Code Conventions

### Python (Backend)

- **Line length**: 88 (ruff default)
- **Formatter**: ruff (with `fix = true, show-fixes = true`)
- **Import style**: `force-single-line = true` (isort via ruff) — each import on its own line
- **Linting**: ruff with extended rule set (COM, DJ, FBT, I, PLC, PLE, PTH, RUF, SIM, T20, TC, UP, W)
- **String quotes**: Ruff defaults (double quotes preferred)
- **Type checking**: `from __future__ import annotations` used in many files; `TYPE_CHECKING` guard for type-only imports
- **Translations**: Use `gettext_lazy as _` for model field `verbose_name`/`help_text`; use `gettext as _` in serializers/views
- **Audit log**: Conditional import pattern: `if settings.AUDIT_LOG_ENABLED: from auditlog...`

Key ruff ignores:
- `DJ001` (nullable charfield) — ignored project-wide
- `PLC0415` (import outside top-level) — ignored for conditional imports
- `RUF012` (mutable class attrs) — ignored
- `SIM105` (try-except-pass) — ignored
- `E501` relaxed in migrations and tests
- `SIM117` relaxed in tests

### TypeScript/Angular (Frontend)

- **Formatter**: Prettier (via pre-commit hooks), with `prettier-plugin-organize-imports`
- **Component selector prefix**: `pngx` (attribute for directives, element for components)
  - Directives: `@Directive({ selector: '[pngxIfPermissions]' })`
  - Components: `@Component({ selector: 'pngx-document-detail' })`
- **Standalone**: Components use Angular 21 patterns with explicit imports
- **Services**: Extend `AbstractPaperlessService<T>` for REST CRUD operations
- **Data models**: Plain TypeScript interfaces in `src/app/data/`
- **State management**: RxJS BehaviorSubjects and Observables (no NgRx)
- **Testing**: Jest with `jest-preset-angular`; specs colocated as `*.spec.ts`
- **E2E**: Playwright with HAR-based request mocking

### General

- **Line endings**: LF enforced via pre-commit (`mixed-line-ending --fix=lf`)
- **Pre-commit hooks**: Run via `prek` (compatible with pre-commit config); includes ruff check+format, prettier, codespell, shellcheck, hadolint, yamlfmt, pyproject-fmt
- **Commit message style**: No strict convention enforced by tooling

## Testing Patterns

### Backend Tests

- **Test framework**: pytest + pytest-django (`APITestCase` for API tests)
- **Test mixins** (in `src/documents/tests/utils.py`):
  - `DirectoriesMixin` — Sets up temp directories with `override_settings` for all paths
  - `FileSystemAssertsMixin` — `assertIsFile()`, `assertIsNotFile()`, `assertFilesEqual()`, `assertFileCountInDir()`
  - `ConsumerProgressMixin` — Mocks Redis progress reporting
  - `DocumentConsumeDelayMixin` — Mocks `consume_file.delay` Celery task
  - `SampleDirMixin` — Provides path to test sample files
  - `GetConsumerMixin` — Context manager for creating consumer instances in tests
  - `TestMigrations` — Base class for migration tests using `TransactionTestCase`
- **Factories**: `factory-boy` with `DjangoModelFactory` in `src/documents/tests/factories.py`
- **Fixtures** (in `src/documents/tests/conftest.py`):
  - `paperless_dirs` — Temp directory structure for document storage
  - `sample_doc` — Creates a Document with valid files and checksums
  - `rest_api_client` / `authenticated_rest_api_client` — DRF APIClient fixtures
  - `faker_session_locale` / `faker_seed` — Fixed faker state for reproducibility
- **Test markers**: `live`, `nginx`, `gotenberg`, `tika`, `greenmail`, `date_parsing`, `management`
- **Concurrency**: Tests run in parallel via `pytest-xdist` with `--dist=loadscope`

### Frontend Tests

- **Unit tests**: Jest via `@angular-builders/jest` (ESM mode)
- **E2E tests**: Playwright against dev server (Chromium only in CI)
- **Mock pattern**: `__mocks__/` directory for mocking pdfjs-dist

## Important Patterns & Gotchas

### Configuration Loading

Settings are loaded from `paperless.conf` (searched in multiple locations: `PAPERLESS_CONFIGURATION_PATH` env var, `../paperless.conf`, `/etc/paperless.conf`, `/usr/local/etc/paperless.conf`). Copy `paperless.conf.example` to `paperless.conf` to get started.

### Django Apps & URL Structure

- API routes: All under `/api/` prefix
- Router-registered ViewSets: correspondents, document_types, documents, tags, storage_paths, saved_views, tasks, users, groups, mail_accounts, mail_rules, workflows, custom_fields, config, share_links, share_link_bundles, processed_mail
- Custom API endpoints: `/api/documents/post_document/`, `/api/documents/bulk_edit/`, `/api/documents/bulk_download/`, `/api/search/`, `/api/statistics/`, `/api/ui_settings/`, `/api/profile/`, `/api/status/`, `/api/trash/`, `/api/documents/chat/`
- WebSocket: `/ws/status/` via Django Channels (Redis backend in production)

### Async Tasks (Celery)

Document consumption, classification, and other long-running tasks use Celery with Redis as the broker. The task queue service, scheduler, and consumer are separate processes managed by s6-overlay in Docker.

### Permissions System

Uses `django-guardian` for object-level permissions. Models like `Document`, `Tag`, `Correspondent` extend `ModelWithOwner` which provides an `owner` ForeignKey. The `permissions.py` module and `permissions.guard.ts` handle enforcement.

### Soft Delete

Documents use `django-soft-delete` (`SoftDeleteModel`). Deleted documents go to trash before permanent removal. The trash API is at `/api/trash/`.

### Plugin System

Paperless-ngx supports third-party parsers via Python entry points:
- Document parsers: `paperless_ngx.parsers` entry point group
- Date parsers: `paperless_ngx.date_parsers` entry point group
- See `src/documents/plugins/` and `src/documents/plugins/date_parsing/`

### Frontend-Backend Communication

- REST API at `/api/` with CSRF protection
- WebSocket for real-time status updates via `/ws/status/`
- Frontend dev server (port 4200) proxies to backend (port 8000) by default
- Auth: Session-based (Django) with token auth support; social auth via django-allauth

### Source Language

Both frontend and backend use `en_US` as the source language. Translation files are managed via Crowdin.

### CI Pipeline

Three separate CI workflows:
- **Backend** (`ci-backend.yml`): Tests on Python 3.11–3.14 matrix + typing check (mypy + pyrefly)
- **Frontend** (`ci-frontend.yml`): Install → Lint → Unit tests (4 shards) → E2E (2 shards) → Bundle analysis
- **Lint** (`ci-lint.yml`): Runs `prek` (pre-commit) across all files

### Database Support

PostgreSQL (via psycopg3), MariaDB (via mysqlclient), and SQLite are all supported. Docker compose files for each combination are in `docker/compose/`.

### Backend Test Environment Variables

Tests set these via `pyproject.toml` `[tool.pytest_env]`:
- `PAPERLESS_DISABLE_DBHANDLER = "true"`
- `PAPERLESS_CACHE_BACKEND = "django.core.cache.backends.locmem.LocMemCache"`
- `PAPERLESS_CHANNELS_BACKEND = "channels.layers.InMemoryChannelLayer"`
