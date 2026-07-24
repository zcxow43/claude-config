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
4. **Check** files with `status: done` — if new content was appended after `## Execution Result`, treat as pending (strip old result, re-process)
5. **Execute** in strict dependency order: Infra → DBA → Backend → Frontend
6. **Update** each spec's status to `done` after successful execution

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
- Database: wdd
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

Within each domain, execute in filename order (chronological by date prefix).

## Dispatching

For each pending spec, spawn an Agent with the matching `subagent_type`:

- `specs/infra/*.md` → `subagent_type: "infra"`
- `specs/dba/*.md` → `subagent_type: "dba"`
- `specs/backend/*.md` → `subagent_type: "backend"`
- `specs/frontend/*.md` → `subagent_type: "frontend"`

### Working Directory Rules

Each domain has its own working directory. **Never** place backend or frontend code in the project root.

| Domain   | Working Directory | Description                              |
|----------|-------------------|------------------------------------------|
| Infra    | project root      | docker-compose.yml, Dockerfiles, docker/ |
| DBA      | project root      | Migration SQL, docker init scripts       |
| Backend  | `backend/`        | Maven project (pom.xml, src/, etc.)      |
| Frontend | `frontend/`       | npm/Vite project (package.json, src/, etc.) |

Agent prompt format:
```
Execute the following spec. Read the existing codebase to understand conventions before making changes. Read `env.md` for tech stack details. Implement everything described. Do not ask questions — proceed with your best judgment.

IMPORTANT: <domain-specific working directory instruction>
- Infra: docker-compose.yml at project root, Dockerfiles and init scripts in `docker/`. After updating docker-compose.yml, run `docker compose up -d` and verify the new service is healthy.
- DBA: place migration SQL in `src/main/resources/db/migration/` under `backend/`, and Docker init scripts in `docker/mysql/initdb/`.
- Backend: ALL backend code goes in the `backend/` subdirectory. This is the Maven project root. Never place pom.xml or src/ in the project root.
- Frontend: ALL frontend code goes in the `frontend/` subdirectory. This is the npm project root. Never place package.json or src/ in the project root.

Spec file: <path>

<full spec content>
```

After the agent completes, update the spec file:
- Change frontmatter to `status: done`
- The agent should have appended an `## Execution Result` section

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

## Error Handling

- On failure: mark spec as `status: error`, include error in execution result
- Continue with remaining specs — do not stop on failure
- Report all errors in the final summary

## If No Pending Specs

Print: "No pending specs found." and stop.

Do NOT ask for confirmation at any step. Execute everything immediately.
