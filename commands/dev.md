You are a dev executor. Execute spec files by dispatching to the appropriate agent.

## Argument Handling

Check `$ARGUMENTS`:

- **Empty or blank**: scan all `.md` files in `specs/infra/`, `specs/dba/`, `specs/backend/`, `specs/frontend/` and execute every pending one
- **Has value**: treat as a spec file path or slug to execute only that spec. Resolve in order:
  1. Exact path match (e.g. `specs/backend/currency.md`)
  2. Slug match across all domains (e.g. `currency` matches `specs/*/currency.md`)
  3. If no match found: **print error** `Error: spec not found for "$ARGUMENTS"` and **STOP**

## Process

1. **Infrastructure Auto-Discovery** (MANDATORY, before anything else)
2. **Scan** target spec files based on argument resolution above
3. **Identify** files with `status: pending` in frontmatter
4. **Check** files with `status: done` for new/unchecked work:
   - If new `- [ ]` items were appended to `## Acceptance Criteria` (an increment — see "Incremental Specs & Consolidation" below), treat as pending and execute **only the unchecked items**. Never re-run or strip already-checked (`- [x]`) criteria.
   - If free-form content was appended after `## Execution Result` instead, first fold it into new `## Acceptance Criteria` bullet items, then execute those.
5. **Execute** in strict dependency order: Infra → DBA → Backend → Frontend
6. **Check off** (`- [x]`) every Acceptance Criteria item the agent implemented and verified working, immediately after executing it. Leave unmet items as `- [ ]` and state why in the Execution Result (e.g. deferred, blocked).
7. **Update** each spec's status to `done` once every Acceptance Criteria item is checked, or explicitly noted as deferred/blocked

## Infrastructure Auto-Discovery (runs before any spec execution)

Scan ALL pending specs (DBA, Backend, Frontend) to detect which infrastructure services are needed. Then ensure those services exist in `docker-compose.yml`, `env.md`, and `specs/infra/`.

### Step 1: Detect required services

Read all pending spec files and identify infrastructure dependencies. Common patterns:

| Keyword / Technology        | Service needed |
|-----------------------------|---------------|
| MySQL, JDBC, datasource     | mysql         |
| Redis, cache, session store | redis         |
| MongoDB, document store     | mongodb       |
| RabbitMQ, AMQP, message queue | rabbitmq   |
| Kafka, event stream         | kafka         |
| Elasticsearch, search       | elasticsearch |
| MinIO, S3, object storage   | minio         |

### Step 2: Check `env.md` for each required service

For each detected service, check if `env.md` already has a section for it.

- **If the section exists**: use those values (host, port, credentials, etc.)
- **If the section is missing**: append a new section to `env.md` with default values:

Default credentials for all new services: **Username: `app`, Password: `1234`**

Default templates:

```markdown
## Redis
- Host: 127.0.0.1
- Port: 6379
- Password: 1234

## MongoDB
- Host: 127.0.0.1
- Port: 27017
- Database: <same database name already used by this project's other services in env.md>
- Username: app
- Password: 1234

## RabbitMQ
- Host: 127.0.0.1
- Port: 5672
- Management Port: 15672
- Username: app
- Password: 1234
- Virtual Host: /

## Kafka
- Bootstrap Servers: 127.0.0.1:9092

## Elasticsearch
- Host: 127.0.0.1
- Port: 9200

## MinIO
- Host: 127.0.0.1
- Port: 9000
- Console Port: 9001
- Access Key: app
- Secret Key: 12341234
```

### Step 3: Check `docker-compose.yml` for each required service

For each detected service, check if `docker-compose.yml` already has the service defined.

- **If it exists**: skip
- **If it does not exist**: auto-generate a `specs/infra/<service>.md` spec file with `status: pending`

The generated infra spec should contain:
- Service name, image, port mappings (from env.md values)
- Credentials (from env.md)
- Health check
- Named volume for persistence
- Any init scripts directory mount

### Step 4: Verify service connectivity (DB, Redis, and any other detected service)

This pre-flight runs **before any spec (Infra/DBA/Backend/Frontend) is executed** — connectivity must be proven first, ahead of all other spec work.

1. Read `env.md` — extract connection details for **every** detected service that needs a live connection (e.g. Database Host/Port/Database/Username/Password, Redis Host/Port/Password)
2. If `env.md` is missing or a required service's fields are incomplete: **STOP all execution** and report the error
3. Test connectivity for each detected service using its env.md values:
   - MySQL/database: `mysql -h <Host> -P <Port> -u <Username> -p<Password> -e "SELECT 1;" 2>&1`
   - Redis: `redis-cli -h <Host> -p <Port> -a <Password> PING 2>&1`
   - Other services (MongoDB, RabbitMQ, Kafka, Elasticsearch, MinIO, etc.): use the equivalent lightweight CLI/ping check if available
4. If **any** connection fails: **STOP all execution immediately** and report: "<Service> is not reachable. Start it with `docker compose up -d` and retry."
5. If the target database does not exist, create it (use root credentials from docker-compose.yml)
6. On success, print `✅ <Service> pre-flight passed` for each service and continue

If any service pre-flight fails, do NOT execute any specs (Infra, DBA, Backend, or Frontend) — terminate immediately.

### Step 5: Print discovery summary

```
🔍 Infrastructure discovery:
   - mysql: ✅ exists in docker-compose.yml
   - redis: ⚠️  missing → generated specs/infra/redis.md
   - env.md: ✅ updated with Redis section
   ✅ mysql pre-flight passed
   ✅ redis pre-flight passed
```

## Execution Order

1. **Infra** specs first — containers must be running before anything else
2. **DBA** specs second — database schema must exist before backend uses it
3. **Backend** specs third — APIs must exist before frontend calls them
4. **Frontend** specs last — UI connects to backend APIs

**Within each domain, filename order is not a reliable ordering signal** — spec slugs are named after features, not creation date, so alphabetical order does not track build dependencies (e.g. `currency-pair-definition.md` sorts before `currency-pair.md`, but its migration (`V009`) can only run once `currency-pair.md`'s table (`V003`) already exists). Determine order per domain instead:

- **DBA**: order by the migration version number(s) each file *defines as its own*, not every `V0xx` it merely mentions — a spec's `## Migration SQL` heading/fenced block states exactly which `V0xx__description.sql` file(s) it is responsible for creating (e.g. `## Migration SQL — V010 (Delta: ...)`); a spec's prose may separately reference an *earlier* file's version purely as context ("next migration after `V009` is `V011`") — that referenced number belongs to the other file, not this one, and must not be picked up as this file's own. Extract only the version(s) each spec's own `## Migration SQL` section(s) define, then sort files by their lowest own version ascending. A file whose own migrations span a gap (e.g. one spec owns both `V001` and a later `V010` delta) still sorts at its lowest own number, and its later migration(s) must be applied only after every other file's migration with a smaller number has already been applied — do not apply a file's migrations out of order relative to its own accumulated history either.
- **Backend / Frontend**: order by each spec's `depends_on:` frontmatter field — a list of same-domain slugs that must reach `status: done` before this spec starts (e.g. `depends_on: [brand, currency, audit]`). Topologically sort pending specs on this field; specs with no `depends_on` (or whose listed deps are already `done`) have no ordering constraint and may run in any order, including in parallel per "Parallel Rules" below. If a spec's `depends_on` lists another spec that is itself still `pending`, wait for that one to finish first — do not run them out of order or in parallel.
- Every spec file — DBA, backend, and frontend alike — must declare enough of this ordering information (migration filename for DBA; `depends_on` for backend/frontend) for `/dev` to determine correct order without needing a human to already know the feature history. This is what makes "run every pending spec starting from a completely empty project" reproduce the same result as running them incrementally, spec by spec, over time.

## Dispatching

For each pending spec, spawn an Agent with the matching `subagent_type`:

- `specs/infra/*.md` → `subagent_type: "infra"`
- `specs/dba/*.md` → `subagent_type: "dba"`
- `specs/backend/*.md` → `subagent_type: "backend"`
- `specs/frontend/*.md` → `subagent_type: "frontend"`

### Working Directory Rules

Each domain has its own working directory. **Never** place backend or frontend code in the project root.

| Domain   | Working Directory     | Description                              |
|----------|------------------------|------------------------------------------|
| Infra    | project root           | docker-compose.yml, Dockerfiles, docker/ |
| DBA      | project root           | Migration SQL embedded in the spec, applied directly to the live DB |
| Backend  | `develop/backend/`     | Maven project (pom.xml, src/, etc.)      |
| Frontend | `develop/frontend/`    | npm/Vite project (package.json, src/, etc.) |

If `develop/backend/` or `develop/frontend/` doesn't exist yet, the assigned agent creates it and scaffolds the project skeleton there (Maven `pom.xml` for backend, Vite `package.json` for frontend) as the first step of executing that spec — every backend spec's output lands under `develop/backend/`, every frontend spec's output lands under `develop/frontend/`, regardless of whether the folder already existed.

Agent prompt format:
```
Execute the following spec. Read the existing codebase to understand conventions before making changes. Read `env.md` for tech stack details. Implement everything described. Do not ask questions — proceed with your best judgment.

IMPORTANT: <domain-specific working directory instruction>
- Infra: docker-compose.yml at project root, Dockerfiles and init scripts in `docker/`. After updating docker-compose.yml, run `docker compose up -d` and verify the new service is healthy.
- DBA: migration SQL lives only inside this spec's `## Migration SQL` section — apply it directly against the live database via the `mysql` CLI. Do not write a standalone `.sql` file anywhere (no `docker/mysql/initdb/`, no `develop/backend/src/main/resources/db/migration/`).
- Backend: ALL backend code goes in the `develop/backend/` subdirectory. This is the Maven project root. Never place pom.xml or src/ in the project root. If `develop/backend/` doesn't exist yet, create it and scaffold the Maven project (pom.xml, src/main/java, src/test/java) before implementing this spec.
- Frontend: ALL frontend code goes in the `develop/frontend/` subdirectory. This is the npm project root. Never place package.json or src/ in the project root. If `develop/frontend/` doesn't exist yet, create it and scaffold the Vite project before implementing this spec.

Spec file: <path>

<full spec content>
```

After the agent completes, update the spec file:
- Check off (`- [x]`) every Acceptance Criteria item the agent implemented and verified; leave unmet items unchecked with a reason in the Execution Result
- Change frontmatter to `status: done` only once every item is checked or explicitly noted as deferred/blocked
- The agent should have appended an `## Execution Result` section — for an increment on a previously-done spec, append a new `### Increment <n> — <date>` subsection under the existing one rather than overwriting it

## Parallel Rules

- Same domain, no interdependencies → MAY run in parallel
- Different domains → MUST run sequentially (Infra → DBA → Backend → Frontend)

## Progress

Print progress as you go:

```
[Infra]    specs/infra/mysql.md ... DONE
[DBA]      specs/dba/currency.md ... DONE
[Backend]  specs/backend/currency.md ... DONE
[Frontend] specs/frontend/currency.md ... DONE

All specs executed. 4/4 completed.
```

## Incremental Specs & Consolidation

Specs are living documents per feature, not one-shot files:

- **Adding scope to a `done` feature**: append new bullet items to that domain's existing `## Acceptance Criteria` list instead of creating a new spec file for a small addition. Set frontmatter back to `status: pending` — this command will execute only the newly appended, unchecked (`- [ ]`) items and leave already-checked (`- [x]`) ones untouched.
- **Execution Result history**: never overwrite or strip a prior `## Execution Result`. Append a new `### Increment <n> — <date>` subsection under it for each re-run, describing only what changed in that increment.
- **Consolidating fragmented specs**: if a domain folder accumulates multiple small files describing the same feature/entity, merge them into one spec per feature area the next time they're touched, rather than letting them multiply — combine their `## Requirements` and `## Acceptance Criteria` sections and keep each file's own Execution Result history intact under clearly dated subsections.

## Error Handling

- On failure: mark spec as `status: error`, include error in execution result
- Continue with remaining specs — do not stop on failure
- Report all errors in the final summary

## If No Pending Specs

Print: "No pending specs found." and stop.

Do NOT ask for confirmation at any step. Execute everything immediately.
