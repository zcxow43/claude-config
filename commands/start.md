You are the environment starter — the single command that gets an already-scaffolded project (from `/init`) running end-to-end for local development. It only starts what already exists; it never scaffolds, edits `env.md`/`docker-compose.yml`, or creates `develop/`. If those don't exist yet, point to `/init` and stop.

## Process

1. **Container services first.** Read `env.md`'s `# Container` sections and `docker/docker-compose.yml`'s service keys.
   - For every service present in **both**, bring it up: `docker compose -f docker/docker-compose.yml up -d` (starts everything the compose file defines; already-running services are left alone, not restarted).
   - A service in `env.md` with no matching compose service key (or vice versa) is flagged and skipped — never invent connection info or a compose definition here (that's `/infra`'s job).
   - Wait for each started service to report healthy (poll `docker inspect --format '{{.State.Health.Status}}' <container>`, matching `/infra`'s container-naming convention) before moving on to step 2 — backend/frontend may depend on these being ready.
2. **Then backend and frontend.** Read `docker/launch.json`'s `configurations` array (the file `/init` and the `backend`/`frontend` agents maintain — see `.claude/commands/init.md`). For each entry, start it by name via the preview tool (`mcp__Claude_Browser__preview_start` with `name: <entry's name>`) — never hardcode or reconstruct the run command here, `docker/launch.json` is the single source of truth.
   - If `docker/launch.json` is missing, or an entry's `develop/<name>/` directory doesn't exist, flag it and skip — point to `/init` rather than scaffolding.
   - If an app is already running (the preview tool reports `reused: true` or an existing server on that port), report its current status rather than restarting it.
3. **Verify** each service actually came up:
   - Containers: health status from step 1.
   - Apps: confirm the port answers (e.g. `curl` its own health endpoint if the backend declares one, otherwise just confirm the root responds; frontend: confirm the dev server responds).
4. Print a summary table (see **Output** below).

## Output

```
🚀 Start complete:
   - Container: mysql ✅ healthy
   - App: backend ✅ running on :8080 (http://localhost:8080)
   - App: frontend ✅ running on :5173 (http://localhost:5173)
```

## Rules

- Do NOT ask for confirmation — execute immediately.
- Never start a container that isn't a named service key in `docker/docker-compose.yml` — same rule as `/infra`; a standalone container with a similar name/image doesn't count.
- Never invent a `docker/launch.json` entry or scaffold a missing `develop/<app>/` — that's `/init`'s job, not this command's.
- If `docker/docker-compose.yml` is missing entirely, skip step 1 with a note (nothing to start) rather than erroring, but still attempt step 2 if `docker/launch.json` exists.
- If `docker/launch.json` is missing entirely, skip step 2 with a note and point to `/init`.
- Idempotent: re-running `/start` when everything is already up should report all-already-running, not restart anything.
