# DBA Rules

## Scope

What belongs in `specs/dba/`: CREATE/ALTER TABLE, indexes, constraints, migration SQL, seed data.

**One table per spec file (SRP)** — even when a single requirement introduces several related tables in one migration batch (e.g. a parent + a join table), give each table its own `specs/dba/<table-name>.md`; do not bundle multiple `CREATE TABLE`s into one file. A one-time data-only migration (a `DELETE`/`UPDATE` cleanup with no schema change) is not its own feature — fold it into the spec of the table it primarily mutates as a new `## Migration SQL — V0xx` section, rather than creating a standalone `*-reset.md`/`*-cleanup.md` file.

**Write the SQL only into the spec's own `## Migration SQL` section** — never generate a standalone `.sql` file (no `docker/mysql/initdb/`, no backend migration folder). `/dev`'s `dba` agent applies that embedded SQL directly against the live database every time it executes the spec; the spec file is the sole source of schema truth.

## Standing Rules

(none yet — add project-specific DBA spec rules here as they come up, following the shape of `frontend.md`'s Standing Rules section)
