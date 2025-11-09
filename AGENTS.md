# AGENTS.md

## Project Overview

Dify is an open-source platform for developing LLM applications with an intuitive interface combining agentic AI workflows, RAG pipelines, agent capabilities, and model management.

The codebase is split into:

- **Backend API** (`/api`): Python Flask application organized with Domain-Driven Design
- **Frontend Web** (`/web`): Next.js 15 application using TypeScript and React 19
- **Docker deployment** (`/docker`): Containerized deployment configurations

## Backend Workflow

- Run backend CLI commands through `uv run --project api <command>`.

- Before submission, all backend modifications must pass local checks: `make lint`, `make type-check`, and `uv run --project api --dev dev/pytest/pytest_unit_tests.sh`.

- Use Makefile targets for linting and formatting; `make lint` and `make type-check` cover the required checks.

- Integration tests are CI-only and are not expected to run in the local environment.

## Frontend Workflow

```bash
cd web
pnpm lint
pnpm lint:fix
pnpm test
```

## Testing & Quality Practices

- Follow TDD: red → green → refactor.
- Use `pytest` for backend tests with Arrange-Act-Assert structure.
- Enforce strong typing; avoid `Any` and prefer explicit type annotations.
- Write self-documenting code; only add comments that explain intent.

## Language Style

- **Python**: Keep type hints on functions and attributes, and implement relevant special methods (e.g., `__repr__`, `__str__`).
- **TypeScript**: Use the strict config, lean on ESLint + Prettier workflows, and avoid `any` types.

## General Practices

- Prefer editing existing files; add new documentation only when requested.
- Inject dependencies through constructors and preserve clean architecture boundaries.
- Handle errors with domain-specific exceptions at the correct layer.

## Project Conventions

- Backend architecture adheres to DDD and Clean Architecture principles.
- Async work runs through Celery with Redis as the broker.
- Frontend user-facing strings must use `web/i18n/en-US/`; avoid hardcoded text.

## DSL Application Development

### File Organization

- DSL applications are stored in `/dsl-applications/`
- Each application has its own subdirectory: `/dsl-applications/app-name/`
- Required files: `app-name.yml` (DSL config) and `README.md` (documentation)

### DSL File Standards

**Required Configuration:**
```yaml
version: 0.4.0              # Fixed for Dify 1.9.2
kind: app
mode: advanced-chat         # or workflow
```

**Variable Reference Format:**
- System variables: `{{#sys.query#}}`
- Conversation variables: `{{#conversation.var_name#}}`
- Node outputs: `{{#node_id.field#}}`

**Node Layout:**
- Horizontal spacing: ~300px between nodes
- Starting position: `x: 80, y: 300`
- Zoom level: `0.8`

**Conversation Variables:**
- **`id` field MUST use UUID format** (e.g., `ab77671b-7e2c-40ce-82cd-5ddedf0bd308`)
- Cannot use plain strings like `"stage"` as IDs - will cause database error
- Match `value` type with `value_type` (string: `''`, number: `0`)
- Use descriptive `name` fields

### DSL Import via API

**Required Authentication:**
```python
headers = {
    'Content-Type': 'application/json',
    'x-csrf-token': csrf_token  # lowercase, from user
}
cookies = {
    'access_token': access_token,
    'csrf_token': csrf_token
}
```

**Import Endpoint:** `http://localhost:8080/console/api/apps/imports`

**Payload:**
```python
{
    'mode': 'yaml-content',
    'yaml_content': yaml_content
}
```

### Critical Node Formats

**Assigner Node (Version 2 - REQUIRED):**

```yaml
- data:
    desc: Update stage to next phase
    items:
    - input_type: variable
      operation: over-write
      value:
      - source_node_id
      - output_field
      variable_selector:
      - conversation
      - target_variable
      write_mode: over-write
    type: assigner
    version: '2'
  height: 86
  id: assigner_node_id
```

**Common Error:** Using old format without `version: '2'` and `items` will cause `conversation_id not found` error.

### Validation Checklist

- All node IDs are unique
- Edge `source` and `target` reference existing nodes
- Model provider is configured in Dify instance
- **Conversation variable `id` fields use UUID format (not plain strings)**
- Conversation variable `value` types match their `value_type`
- **🔴 CRITICAL: All Assigner nodes MUST use `version: '2'` with `items` array structure**
