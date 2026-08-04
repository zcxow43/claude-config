You are a project bootstrapper. Read `env.md` and create the base project skeleton it declares — no spec files required. This runs once, before any `/spec`/`/dev` feature work exists, to stand up an empty-but-buildable project matching `env.md`.

## Docker Pre-flight (MANDATORY, before anything else)

1. Check Docker is installed: `docker --version`. If this fails (command not found), **ABORT** with:
   ```
   ❌ ABORT: Docker is not installed.
   Install Docker Desktop (or the Docker Engine CLI) before running /init.
   ```
2. Check the Docker daemon is running: `docker info`. If this fails (daemon not reachable), **ABORT** with:
   ```
   ❌ ABORT: Docker is installed but not running.
   Start Docker Desktop (or the Docker daemon) before running /init.
   ```
3. On success, print `✅ Docker pre-flight passed (<version>)` and continue.

Do NOT scaffold anything or touch `docker-compose.yml` if this pre-flight fails.

## Process

1. **Docker Pre-flight** (above) — stop on any failure.
2. **Read `env.md`** completely. It has two top-level groups:
   - `# Develop` — one `## <Name>` subsection per app (currently `## Frontend`, `## Backend`)
   - `# Container` — one `## <Name>` subsection per containerized service (currently `## Database`; may grow to include Redis, MongoDB, etc.)
3. **For each `# Develop` subsection with a scaffold agent** (`## Frontend`, `## Backend`), check whether its directory (`develop/frontend/`, `develop/backend/`) already exists:
   - **Exists** → skip it entirely. Never touch or overwrite existing app code, and never overwrite an existing `docker/start-<name>.sh`.
   - **Missing** → spawn the matching agent (`backend` for `## Backend`, `frontend` for `## Frontend`) with the prompt below. The agent picks the run command itself, following its own conventions — `env.md` no longer states one.
4. **For the `# Container` group**, spawn the `infra` agent once with the prompt below to reconcile `docker/docker-compose.yml` against every service section under `# Container`, then bring the services up and verify health.
5. **For `## Demo`** (no scaffold agent — `demo/` is permanent and checked into git, unlike `develop/`): if `docker/start-demo.sh` is missing (e.g. after `/teardown` deleted `docker/`), recreate it directly — `exec node demo/server.js` from the project root, `chmod +x`. Skip if it already exists.
6. **Reconcile `.claude/launch.json`** against every `# Develop` subsection with a scaffolded app or script (Frontend, Backend, Demo, etc.): `runtimeExecutable` is always `sh`, `runtimeArgs` is `["docker/start-<name>.sh"]` (that script must already exist from step 3/5 — never invent its command here), and `port` is read from that app's own config, never from `env.md` (which doesn't state ports) — backend's `develop/backend/src/main/resources/application.yml` `server.port`, frontend's `develop/frontend/vite.config.ts` `server.port` (or `5173`, Vite's default, if unset), demo's hardcoded `port` constant in `demo/server.js`. This file is git-ignored inside `.claude` (it's project-specific, not shared tooling — see `.claude/.gitignore`), so a fresh checkout of either repo won't have it on disk at all; do not assume it exists. If `.claude/launch.json` is missing, create it fresh with one entry per subsection. If it exists, add/update only the entries that don't already match exactly.
7. Print a summary table (see **Output** below).

## Agent Prompts

### Backend / Frontend scaffold (only when the directory is missing)
```
Bootstrap-only task: no feature spec exists yet. Scaffold an empty, buildable project skeleton under `develop/<backend|frontend>/` matching the `## <Backend|Frontend>` section of `env.md` exactly (language, framework, build tool, etc. — read `env.md` yourself for the current fields). No feature code, no controllers/pages beyond whatever a minimal "hello world" health-check endpoint/page requires to prove the skeleton builds and runs. Follow every convention in your own agent definition for anything `env.md` doesn't state (base package, port, Lombok, Maven artifact name = parent directory name, semantic version 0.0.1, etc.). Verify it builds (`mvn compile` / `npm run build`) before finishing. Then write an executable `docker/start-<backend|frontend>.sh` (`chmod +x`) whose body `cd`s to the project root and `exec`s the exact command to run your app in dev mode (e.g. `mvn -f develop/backend/pom.xml spring-boot:run`, or `npm --prefix develop/frontend run dev`) — this script is the single source of truth for how your app starts.
```

### Container reconciliation
```
Bootstrap-only task: reconcile `docker/docker-compose.yml` against every service section under the `# Container` heading in `env.md` — one compose service per section, using your Service Templates and every value (host/port/credentials/database name) from that section. Do not touch any service already correctly defined. After updating the compose file, run `docker compose -f docker/docker-compose.yml up -d` and verify every service passes its health check.
```

## Output

After both steps finish, print:

| Group    | Item     | Action                          |
|----------|----------|----------------------------------|
| Develop  | backend  | scaffolded (+ start-backend.sh) / skipped (exists) |
| Develop  | frontend | scaffolded (+ start-frontend.sh) / skipped (exists) |
| Develop  | demo     | start-demo.sh recreated / skipped (exists) |
| Container| database | compose service ensured, ✅ healthy |
| Launch   | launch.json | created / updated / already matched env.md |

## Rules

- Do NOT ask for confirmation — execute immediately.
- Idempotent: re-running `/init` after the project already exists should skip everything and report all-skipped, not overwrite anything.
- This command never executes `specs/**/*.md` — that is `/dev`'s job. `/init` only builds the skeleton `/dev` will later fill in.
