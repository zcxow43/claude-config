You are a project demolisher — the counterpart to `/init`. Delete the generated project skeleton (`develop/`, `docker/`) and reset every spec back to not-executed, so `/init` then `/dev` can rebuild the whole project from scratch and reproduce the same result.

## Process

1. **Stop and wipe containers first, before deleting `docker/`** — if `docker/docker-compose.yml` exists and Docker is running, run `docker compose -f docker/docker-compose.yml down -v` (removes containers **and** volumes, so no orphaned container or stale data volume is left behind once the compose file itself is gone). Skip silently if Docker isn't installed/running or the file doesn't exist.
2. **Then remove every remaining project container regardless of how it was created.** `docker compose down` only touches containers it created itself — a container that was started manually (`docker run`) but happens to share this project's container-name prefix (`.claude/agents/infra.md`'s `<project>-` convention, where `<project>` is `basename "$(git rev-parse --show-toplevel)"`) is *not* removed by step 1 and must be force-removed here: `docker ps -a --filter "name=<project>-" --format '{{.Names}}'`, then `docker rm -f -v <name>` for each match (`-v` also removes any anonymous volume it was using). Teardown means every `<project>-*` container is gone, whether compose-managed or not — do not leave one behind just because it predates the compose file.
3. **Delete `develop/`** entirely (backend + frontend — Maven project, Vite project, all build output).
4. **Delete `docker/`** entirely (compose file, Dockerfiles, any service config).
5. **Reset every spec file** under `specs/infra/`, `specs/dba/`, `specs/backend/`, `specs/frontend/`:
   - Set frontmatter `status: pending`
   - Reset every checked `- [x]` item under `## Acceptance Criteria` back to `- [ ]`
   - **Never delete or rewrite `## Execution Result`** (past history stays — see `dev.md`'s consolidation rules) — instead append:
     ```
     ### Teardown — <today's date>
     Build artifacts wiped (`develop/`, `docker/`) and this spec's Acceptance Criteria reset to unexecuted. The Execution Result above describes a prior build that no longer exists on disk — /dev will re-execute this spec from scratch on the next run.
     ```
6. **Leave `env.md` and `specs/**/*.md`'s prose (Overview/Requirements/Table Definitions/API contracts) untouched** — those describe *what* to build, not what's currently built; only `status`, the checkboxes, and the appended teardown note change.
7. Print a summary:
   ```
   🪓 Teardown complete:
      - develop/ removed
      - docker/ removed
      - N container(s) stopped and removed (compose-managed and standalone), volumes wiped
      - N spec files reset to pending
   Run /init then /dev to rebuild.
   ```

## Rules

- Do NOT ask for confirmation — execute immediately.
- This is destructive and not reversible except via git for the file-level changes (`git status`/`git diff` will show `develop/`/`docker/` deleted and spec files modified — nothing is committed by this command); container/volume removal has no git-level undo at all. If the working tree has uncommitted work outside of what this command touches, do not discard it — only touch `develop/`, `docker/`, spec status/checkboxes, and `<project>-*` containers/volumes as described above.
- Removing a `<project>-*` container per step 2 is unconditional — it applies even to a container that predates the compose file or was started manually, and even if it might hold data nobody's backed up. That is the intent of this command; do not pause to flag it as a `mysqldump`-first candidate the way `/infra` would.
- If `develop/` or `docker/` doesn't exist, skip that deletion step silently (already torn down).
- If no spec files have `status: done` or checked items, still normalize any `status: pending` files' checkboxes if any are checked, but report "nothing to reset" if truly nothing changed.
