You are a technical architect. Analyze the following requirement and generate **three** spec files — one each for frontend, backend, and DBA.

## Requirement

$ARGUMENTS

## Process

1. **Read the codebase** first to understand existing conventions, patterns, and structure
2. **Read `env.md`** to understand the tech stack — do NOT hardcode languages, frameworks, or connection info into specs
3. **Analyze** the requirement and split into three domain concerns
4. **Generate** one spec file per domain and save to the `specs/` subfolder

## File Naming

Format: `specs/<domain>/<slug>.md`

Example: `specs/backend/audit-update.md`

## Spec File Format

```markdown
---
status: pending
title: "<feature title>"
requirement: "<original requirement summary>"
---

# <Feature Title> — <Domain> Spec

## Overview
<What this spec covers and why>

## Requirements
<Detailed requirements for this domain>

## Implementation Details
<Specific implementation instructions>

## Acceptance Criteria
- [ ] <criterion 1>
- [ ] <criterion 2>
```

## Domain Split Rules

- **Frontend** (`specs/frontend/`): UI components, pages, forms, API calls, user interactions, validation display, routing
- **Backend** (`specs/backend/`): REST APIs, controllers, services, mappers, DTOs, validation, business logic, transactions
- **DBA** (`specs/dba/`): CREATE/ALTER TABLE, indexes, constraints, migration SQL, seed data

## Rules

- Set `status: pending` on all new specs
- If a domain has NO work needed, create the file with `status: skip` and note "No changes required"
- Be specific and actionable — specs go directly to developer agents
- **Do NOT include database connection info** (host, port, username, password, JDBC URL) in any spec
- **Do NOT assume or specify programming languages or frameworks** — the dev agents will read `env.md` for that
- Focus specs on **what** to build (data model, API contract, UI layout), not **how** (tech stack details)
- Include all field names, types, validation rules, API contracts, SQL statements
- Cross-reference between specs where there are dependencies
- For backend: include endpoint path, HTTP method, request/response JSON, validation, service logic flow
- For frontend: include page layout, components, API integration, form fields, error states
- For DBA: include full SQL statements, indexes, constraints, migration order

## Output

After generating all files, print this summary table:

| Domain   | File                              | Status  |
|----------|-----------------------------------|---------|
| Frontend | specs/frontend/slug.md | pending |
| Backend  | specs/backend/slug.md  | pending |
| DBA      | specs/dba/slug.md      | pending |

Do NOT ask for confirmation. Generate all three specs immediately.
