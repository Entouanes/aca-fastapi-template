# Copilot Instructions for ACA FastAPI Template

## Project Architecture

This is a **FastAPI container app** designed for deployment to **Azure Container Apps (ACA)**. The stack:
- **Backend**: FastAPI app in `src/api/main.py` serving `/generate_name` endpoint
- **Server**: Gunicorn with Uvicorn workers (config: `src/gunicorn.conf.py`, port 3100)
- **Deployment**: Azure Developer CLI (`azd`) + Bicep infrastructure (`infra/`)
- **Dependencies**: Managed by `uv` package manager (not pip/poetry)

Key architectural decisions:
- Port 3100 is used for Docker/production (not FastAPI default 8000)
- Gunicorn workers formula: `(cpu_count * 2) + 1` for production load handling
- Container Registry + Container Apps Environment pattern for Azure deployment

## Critical Developer Workflows

### Local Development
```bash
# Setup (first time)
uv sync --extra dev
uv run pre-commit install

# Run dev server (uses port 8000)
uv run fastapi dev src/api/main.py

# Run tests (requires 100% coverage)
uv run pytest

# Docker local testing (port 3100)
docker build --tag fastapi-app --load ./src
docker run --publish 3100:3100 fastapi-app
```

### Azure Deployment
```bash
# One-time setup
azd auth login

# Deploy everything (provision + deploy)
azd up

# Update code only
azd deploy

# Teardown
azd down
```

## Testing Requirements

- **100% code coverage required** (enforced by `pytest.ini_options.fail_under = 100`)
- Test files: `src/api/api_test.py` and `src/gunicorn_test.py`
- Use `TestClient` from `fastapi.testclient` for API testing
- Control randomness with `random.seed()` for deterministic tests
- Run with: `uv run pytest`

## Code Quality Standards

**Pre-commit hooks check:**
- **Formatter**: Black (line-length: 120)
- **Linter**: Ruff (rules: E, F, I, UP)
- **Target**: Python 3.13+

When adding new code:
1. Follow 120-character line limit
2. Import sorting with Ruff
3. All functions must have test coverage
4. Pre-commit runs automatically on `git commit`

## Bicep Infrastructure Patterns

**Module hierarchy:**
- `main.bicep` (subscription scope) → creates resource group and orchestrates modules
- `api.bicep` → defines the FastAPI container app
- `core/host/container-apps.bicep` → shared Container Apps infrastructure
- `core/monitor/loganalytics.bicep` → observability

**Key conventions:**
- Resource naming: `${name}-${resourceToken}` pattern for uniqueness
- Tags: All resources tagged with `azd-env-name`
- Container port: 3100 (matches Gunicorn config)
- Outputs: Used by `azd` for deployment (e.g., `SERVICE_API_IMAGE_NAME`)

When modifying infrastructure:
- Update `main.parameters.json` for new parameters
- Use `module` syntax for reusable components
- Maintain output variables for `azd` integration

## Project-Specific Conventions

**Directory structure:**
- `src/api/` → FastAPI app code (not just `src/`)
- `infra/` → All Bicep files (not `infrastructure/` or `iac/`)
- Test files alongside source: `api_test.py` next to `main.py`

**Dependency management:**
- Use `uv` commands (not `pip install`)
- Two requirement files: `requirements.txt` (prod) + `pyproject.toml` (dev)
- Docker uses `requirements.txt`, local dev uses `pyproject.toml`

**API design:**
- Query parameters for filtering (e.g., `?starts_with=N`)
- Case-insensitive parameter handling
- Always return JSON objects (not raw strings)

## Common Gotchas

1. **Port confusion**: Dev server uses 8000, Docker/production uses 3100
2. **Docker on Apple M1/M2**: Cannot run inside Dev Container—use Codespaces or run natively
3. **Coverage failures**: Add tests immediately when adding new functions
4. **azd deployment errors**: Try different Azure region if provision fails
5. **Container registry costs**: ACR has fixed daily cost—run `azd down` when not in use

## Azure Resources Created

- **Container Apps Environment**: Hosts the FastAPI container
- **Container Registry**: Stores Docker images
- **Log Analytics Workspace**: Observability and diagnostics
- **Managed Identity**: For secure registry access

Resources are in a single resource group: `${name}-rg`
