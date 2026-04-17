# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

CISO Assistant is a **Governance, Risk, and Compliance (GRC)** platform with a Django REST Framework backend, SvelteKit frontend, and 91+ built-in security frameworks (ISO 27001, NIST CSF, CIS, GDPR, etc.). Licensed under AGPL v3 (Community Edition).

## Development Commands

### Backend (run from `backend/` directory)

```bash
# Install dependencies
poetry install

# Run migrations
poetry run python manage.py migrate

# Load framework libraries into DB
poetry run python manage.py storelibraries

# Dev server (port 8173)
poetry run python manage.py runserver localhost:8173

# Run all tests
poetry run pytest --reuse-db

# Run a single test file
poetry run pytest app_tests/api/test_models.py --reuse-db

# Run a specific test
poetry run pytest app_tests/api/test_models.py::test_function_name --reuse-db

# Format check (ruff)
ruff format --check .

# Type checking
poetry run mypy
```

### Frontend (run from `frontend/` directory)

```bash
# Install dependencies
pnpm i --frozen-lockfile
pnpm run prepare

# Dev server
pnpm run dev

# Build
pnpm run build

# Unit tests (watch mode)
pnpm run test

# Unit tests (CI, single run)
pnpm run test:ci

# E2E tests (requires Docker)
pnpm run test:e2e

# Run specific Playwright test
pnpm exec playwright test tests/path/to/test.spec.ts

# Format check
pnpm exec prettier --check .

# Format fix
pnpm run format

# Svelte type check
pnpm run check
```

### Docker (full stack)

```bash
./docker-compose.sh   # Quick start: backend + frontend + Caddy on port 8443
```

### Pre-commit hooks

```bash
pre-commit run --all-files   # ruff format, yaml check, prettier
```

## Architecture

### Tech Stack
- **Backend**: Django 6.0+ / DRF / Python 3.14 / Poetry
- **Frontend**: SvelteKit / Svelte 5 / TypeScript / Tailwind CSS 4 / pnpm
- **Task Queue**: Huey (async background jobs)
- **Database**: SQLite (dev) or PostgreSQL (production)
- **i18n**: Paraglide (25 languages)
- **Auth**: django-allauth + Knox tokens, SSO via OIDC/SAML

### Multi-Tenant Folder Model

All domain objects are scoped to a **folder hierarchy**:

```
ROOT_FOLDER (Global)
  ├── DOMAIN folders (organizational units)
  │   └── ENCLAVE folders (sub-units)
  └── Global referential objects (frameworks, libraries, risk matrices)
```

- `FolderMixin` on models provides a `folder` FK that scopes objects to a domain
- Referential objects (frameworks, threats, reference controls) live in ROOT and are shared
- Domain-scoped objects: risk assessments, compliance assessments, applied controls, assets, evidence, etc.

### Permission System (RBAC)

Azure IAM-inspired role-based access control:

```
User/UserGroup → RoleAssignment → Role (with permission codenames)
                                → perimeter_folders (which domains)
                                → is_recursive (applies to subfolders?)
```

- **`RoleAssignment.is_access_allowed(user, permission, folder)`** - checks permission in folder scope
- **`RoleAssignment.get_accessible_object_ids(folder, user, model)`** - returns (viewable, changeable, deletable) ID tuples
- Built-in roles: ADMIN, DOMAIN_MANAGER, ANALYST, APPROVER, READER
- Permission enforcement happens in both `BaseModelViewSet.get_queryset()` and `BaseModelSerializer.create()/update()`
- Snapshot caching (`iam/cache_builders.py`) for folders, roles, and assignments avoids DB queries on every permission check

### Backend Patterns

**ViewSets**: `BaseModelViewSet` (in `core/views.py`) is the base for all API endpoints. It filters querysets by the user's accessible object IDs based on their role assignments.

**Serializers**: `SerializerFactory` dynamically selects `{Model}ReadSerializer` for list/retrieve and `{Model}WriteSerializer` for create/update. `BaseModelSerializer` enforces permission checks and blocks modification of imported referential objects (those with URNs).

**Model Mixins**: Extensive use of composition:
- `AbstractBaseModel` - UUID id, created_at, updated_at
- `NameDescriptionMixin` - name, description
- `FolderMixin` - folder FK scoping
- `ReferentialObjectMixin` - URN, provider, translations (for library-imported objects)
- `ETADueDateMixin` - eta, due_date fields

**API routing**: All models registered via DRF DefaultRouter in `core/urls.py` → standard REST endpoints at `/api/{model}/`.

### Frontend Patterns

**Route structure**: `routes/(app)/(internal)/[model=urlmodel]/` provides generic list/detail/edit views. The `[model=urlmodel]` param matches kebab-case model names.

**Server hooks** (`hooks.server.ts`): Validates Knox auth token from cookie, injects `Authorization`, `X-CSRFToken`, `Accept-Language`, and `X-Focus-Folder-Id` headers on all `/api/*` requests.

**Feature flags**: Loaded from `/api/settings/feature-flags/` and available to all routes via layout server load.

### Key Backend Apps

| App | Purpose |
|-----|---------|
| `core` | Main GRC models, views, serializers (risk/compliance assessments, controls, assets) |
| `iam` | Folder hierarchy, User, Role, RoleAssignment, permission resolution |
| `library` | Framework/control library import and management |
| `ebios_rm` | EBIOS RM methodology support |
| `tprm` | Third-party risk management |
| `serdes` | Serialization/deserialization for import/export |
| `global_settings` | System-wide configuration |

### Framework Libraries

91+ security frameworks defined as Excel files in `tools/excel/`. Converted to YAML via `tools/convert_library_v2.py`, then loaded into the database via `manage.py storelibraries`.

## Conventions

### Commit Messages

Follows [Conventional Commits](https://www.conventionalcommits.org). Squash merges are standard.

```
<type>(<scope>): <description>
```

Types: `fix`, `feat`, `chore`, `refactor`, `perf`, `docs`, `test`, `ci`, `build`. Use `!` for breaking changes (e.g., `feat!:`). All lowercase.

### Code Style

- **Backend**: Ruff formatter (excludes migrations). No trailing whitespace.
- **Frontend**: Prettier with tabs, single quotes, no trailing commas, 100 char width. Svelte files use the `svelte` parser.
- **Python tests**: Files named `test_*.py` or `*_test.py` in `backend/app_tests/api/`.

### Environment Variables

Backend: `POSTGRES_NAME` (triggers PostgreSQL mode), `DJANGO_DEBUG`, `CISO_ASSISTANT_URL`, `EMAIL_HOST`, `LOG_LEVEL`.
Frontend: `PUBLIC_BACKEND_API_URL` (default `http://localhost:8000/api`).
