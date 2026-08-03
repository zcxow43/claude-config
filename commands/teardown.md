You are a project demolisher — the counterpart to `/init`. Delete the generated project skeleton (`develop/`, `docker/`) and reset every spec back to not-executed, so `/init` then `/dev` can rebuild the whole project from scratch and reproduce the same result.

## Process

1. **Stop and wipe containers first, before deleting `docker/`** — if `docker/docker-compose.yml` exists and Docker is running, run `docker compose -f docker/docker-compose.yml down -v` (removes containers **and** volumes, so no orphaned container or stale data volume is left behind once the compose file itself is gone). Skip silently if Docker isn't installed/running or the file doesn't exist.
2. **Delete `develop/`** entirely (backend + frontend — Maven project, Vite project, all build output).
3. **Delete `docker/`** entirely (compose file, Dockerfiles, any service config).
4. **Reset every spec file** under `specs/infra/`, `specs/dba/`, `specs/backend/`, `specs/frontend/`:
   - Set frontmatter `status: pending`
   - Reset every checked `- [x]` item under `## Acceptance Criteria` back to `- [ ]`
   - **Never delete or rewrite `## Execution Result`** (past history stays — see `dev.md`'s consolidation rules) — instead append:
     ```
     ### Teardown — <today's date>
     Build artifacts wiped (`develop/`, `docker/`) and this spec's Acceptance Criteria reset to unexecuted. The Execution Result above describes a prior build that no longer exists on disk — /dev will re-execute this spec from scratch on the next run.
     ```
5. **Leave `env.md` and `specs/**/*.md`'s prose (Overview/Requirements/Table Definitions/API contracts) untouched** — those describe *what* to build, not what's currently built; only `status`, the checkboxes, and the appended teardown note change.
6. Print a summary:
   ```
   🪓 Teardown complete:
      - develop/ removed
      - docker/ removed (containers + volumes stopped and wiped)
      - N spec files reset to pending
   Run /init then /dev to rebuild.
   ```

## Rules

- Do NOT ask for confirmation — execute immediately.
- This is destructive and not reversible except via git (`git status`/`git diff` will show everything as deleted/modified — nothing is committed by this command). If the working tree has uncommitted work outside of what this command touches, do not discard it — only touch `develop/`, `docker/`, and spec status/checkboxes as described above.
- If `develop/` or `docker/` doesn't exist, skip that deletion step silently (already torn down).
- If no spec files have `status: done` or checked items, still normalize any `status: pending` files' checkboxes if any are checked, but report "nothing to reset" if truly nothing changed.
