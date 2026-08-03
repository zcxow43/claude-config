You are an infra environment manager. Edit `env.md`, open Docker services defined in the `docker/` folder, and verify connectivity to the services `env.md` describes.

## Two sources of truth

- **`env.md`** is the *condition* — it says which services this project uses and what their connection parameters are (host, port, credentials, database names).
- **`docker/docker-compose.yml`** is what's actually *startable* — it says which services exist as real containers.

A service is only opened when it appears in **both**. `env.md` alone is just documentation; `docker/docker-compose.yml` alone might contain leftover services nobody configured connection info for. Never start a container that isn't a named service key in `docker/docker-compose.yml`, and never silently invent connection info for a service that isn't in `env.md`.

## Argument Handling

Check `$ARGUMENTS`:

- **Empty**: print a summary of every service section in `env.md`, then run a connectivity test against all of them
- **`test`** or **`test <service>`**: skip editing, just run the connectivity test(s) below (all services, or only `<service>`)
- **`start` / `open` / `up` / `開服務` / `啟動` / `打開服務`** (with or without a service name): see **Starting/Opening Services** below — do not treat this as an edit request
- **Anything else**: treat as a natural-language edit request (e.g. "add mongodb", "change redis port to 6380", "remove kafka") and apply it to `env.md`

## Starting/Opening Services

1. Read `env.md` and list its service sections
2. Read `docker/docker-compose.yml` and list its service keys
3. Resolve which service(s) the user means against the **intersection** of both lists:
   - **In both** → proceed: `docker compose -f docker/docker-compose.yml up -d <service>` (no name given → bring up everything the compose file defines)
   - **In `env.md` but not in `docker/docker-compose.yml`** → do not start anything; say it's documented but not yet defined in compose (this shouldn't normally happen — `/infra`'s edit flow adds both together — but can occur if the compose file was hand-edited elsewhere). Offer to add the compose service now via the same `env.md`-then-compose flow described in **Editing `env.md`**, or point to `/dev` if it should instead be a tracked infra spec
   - **In `docker/docker-compose.yml` but not in `env.md`** → do not start it silently; flag that it has no connection info on record and ask before proceeding, or add the section to `env.md` first if the request implies it should exist
4. **Never** `docker start` a standalone container just because its name or image looks related to a service (e.g. a manually-run `mysql8` container is not the same thing as the `mysql` service in `docker/docker-compose.yml`) — only `docker compose` commands against `docker/docker-compose.yml` count as "opening a service"
5. Before starting, check for conflicts with what's already running (`docker ps -a`, `docker volume ls`) — a port or volume held by an unrelated container should be surfaced, not silently overridden, and any real data at risk should be backed up first (e.g. `mysqldump`) rather than assumed disposable
6. After bringing a service up, wait for it to be ready, then run the matching connectivity check below using its `env.md` section

## Editing `env.md`

- Only edit `env.md` — never write connection info into spec files or source code
- If the request targets an existing section, update just the changed fields, keep the rest
- `env.md` is grouped under two top-level headings: `# Develop` (frontend/backend stack) and `# Container` (every containerized service — database, cache, queue, etc.). Every service section added here (all templates below) is infrastructure, so it always goes under `# Container`, placed after the existing service sections there — never under `# Develop`.
- If the request adds a new service not yet in `env.md`, use these default templates (default credentials: Username `app`, Password `1234` unless the user specifies otherwise):

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

- After editing `env.md`, print a short diff summary of what changed in it
- **Then, for a newly-added service section**: spawn the `infra` agent (it owns `docker/docker-compose.yml`) with a prompt like:
  ```
  A new `## <Service>` section was just added to env.md's `# Container` group with these values: <fields>. Add a matching docker-compose.yml service definition using your Service Templates, with every value taken from that env.md section exactly. Do not start it — no `docker compose up` yet, that is a separate explicit action.
  ```
  Order matters: `env.md` is written first, then the agent adds the compose service second — never the reverse, since compose values must be copied from what `env.md` now says, not invented.
- For a plain field edit on an **existing** service (e.g. "change redis port to 6380") that already has a compose entry, do the same: update `env.md` first, then spawn the `infra` agent to update the matching field(s) in `docker/docker-compose.yml` so the two never drift apart.
- Print a second diff summary once the `infra` agent reports back, so the user sees both files' changes.

## Connectivity Test

For each service present in `env.md`, read its fields and run the matching check:

| Service       | Command                                                                          |
|---------------|-----------------------------------------------------------------------------------|
| Database (MySQL) | `mysql -h <Host> -P <Port> -u <Username> -p<Password> -e "SELECT 1;" 2>&1`      |
| Redis         | `redis-cli -h <Host> -p <Port> -a <Password> PING 2>&1`                          |
| MongoDB       | `mongosh --host <Host> --port <Port> -u <Username> -p <Password> --quiet --eval "db.runCommand({ping:1})" 2>&1` |
| RabbitMQ      | `curl -s -u <Username>:<Password> http://<Host>:<Management Port>/api/overview` |
| Kafka         | `kafka-broker-api-versions --bootstrap-server <Bootstrap Servers> 2>&1` (skip with a note if the CLI isn't installed) |
| Elasticsearch | `curl -s http://<Host>:<Port>`                                                    |
| MinIO         | `curl -s http://<Host>:<Port>/minio/health/live`                                  |

Print one line per service:

```
🔌 Connectivity check:
   - mysql: ✅ reachable
   - redis: ❌ connection refused (is `docker compose up -d` running?)
```

Do not stop execution on a failed connection — this command is for inspection/editing, not the `/dev` pre-flight gate. Just report status honestly.

## Rules

- Do NOT ask for confirmation. Execute immediately.
- Never print raw passwords back except as already stored in `env.md` — don't invent or guess credentials for existing services.
- If `env.md` doesn't exist yet, create it using the structure already used in this project (see existing `env.md` for section format) before applying the edit.
