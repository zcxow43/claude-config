You are the environment closer — the inverse of `/start`. Stop everything `/start` brought up, in reverse order, without deleting anything. This is **not** `/destroy`: no files are removed, no volumes are wiped, no spec status changes. Containers are only ever `stop`ped, never `down -v`'d.

## Process

1. **Backend and frontend first.** List active preview servers (`mcp__Claude_Browser__preview_list`). For each one whose `name` matches an entry in `docker/launch.json`'s `configurations` array (i.e. it was started by `/start`/`/init` via that file — not some unrelated preview session), stop it with `mcp__Claude_Browser__preview_stop` using its `serverId`. Leave alone any active preview server whose name doesn't match a `docker/launch.json` entry — that's not this command's to touch.
2. **Then container services.** Read `env.md`'s `# Container` sections and `docker/docker-compose.yml`'s service keys, same intersection rule as `/infra`/`/start`. Run `docker compose -f docker/docker-compose.yml stop` — this stops the containers but keeps them (and their volumes/data) intact for a fast `/start` next time. Never `down`, never `-v`, never remove anything.
3. **Verify**:
   - Apps: confirm each stopped port no longer answers (e.g. `curl` fails to connect).
   - Containers: confirm each is `Exited`, not `Running` (`docker ps -a --filter name=<container>`).
4. Print a summary (see **Output** below).

## Output

```
🛑 Close complete:
   - App: backend ✅ stopped
   - App: frontend ✅ stopped
   - Container: mysql ✅ stopped (data preserved)
```

If something was already stopped, report that instead of erroring:

```
🛑 Close complete:
   - App: backend — already stopped
   - Container: mysql ✅ stopped (data preserved)
```

## Rules

- Do NOT ask for confirmation — execute immediately.
- Never `docker compose down` or pass `-v` — that deletes containers/volumes, which is `/destroy`'s job, not this command's. `/close` only pauses things; `/destroy` deletes them.
- Never stop a container that isn't a named service key in `docker/docker-compose.yml`, and never stop a preview server whose name doesn't match a `docker/launch.json` entry — same scoping rule `/start`/`/infra` use for starting, applied in reverse.
- If nothing is running, report that and stop — don't treat it as an error.
- If `docker/docker-compose.yml` or `docker/launch.json` is missing, skip that half of the process with a note rather than erroring.
