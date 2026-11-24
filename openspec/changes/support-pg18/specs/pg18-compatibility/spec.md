# Spec Delta

## Purpose

Defines how pg_dbms_stats builds, installs, upgrades, and presents locked planner statistics on PostgreSQL 18, so that plans stay stable on that server major version.

## ADDED Requirements

### Requirement: Builds and loads on PostgreSQL 18
The extension SHALL compile with PGXS against PostgreSQL 18 headers with no errors and no new compiler warnings. It SHALL load through `shared_preload_libraries` on a PostgreSQL 18 server. Loading SHALL register the `pg_dbms_stats.use_locked_stats` GUC (default `on`, settable per user) and reserve the `pg_dbms_stats` GUC prefix. Attempts to set an unknown `pg_dbms_stats.*` parameter SHALL then be rejected.

#### Scenario: PGXS build
- **WHEN** `make USE_PGXS=1` runs with a PostgreSQL 18 `pg_config` on `PATH`
- **THEN** the build produces `pg_dbms_stats.so` and both `-18` doc samples with no errors and no compiler warnings beyond those the PG14 build produced

#### Scenario: Preloaded on server start
- **WHEN** a PostgreSQL 18 server starts with `shared_preload_libraries = 'pg_dbms_stats'`
- **THEN** the server starts normally and `SHOW pg_dbms_stats.use_locked_stats` returns `on`

#### Scenario: Unknown parameter under reserved prefix
- **WHEN** a session runs `SET pg_dbms_stats.no_such_param = 1`
- **THEN** the server rejects the name as invalid under the reserved `pg_dbms_stats` prefix and creates no placeholder variable

### Requirement: Versioned release for PostgreSQL 18
The extension SHALL ship as version `18.0`. `CREATE EXTENSION pg_dbms_stats` SHALL install version `18.0` into the `dbms_stats` schema. Packaging (the RPM spec, source tarball names, and installed doc samples) SHALL target PostgreSQL 18.

#### Scenario: Fresh install
- **WHEN** a superuser runs `CREATE EXTENSION pg_dbms_stats` on PostgreSQL 18
- **THEN** `pg_extension.extversion` for `pg_dbms_stats` is `18.0` and all `dbms_stats` objects exist

#### Scenario: RPM build
- **WHEN** `make rpm18` runs on a host with PostgreSQL 18 installed under `/usr/pgsql-18`
- **THEN** an RPM named for PostgreSQL 18 is produced that installs `pg_dbms_stats.so`, `pg_dbms_stats--18.0.sql`, the upgrade script, the control file, and `export_{plain,effective}_stats-18.sql.sample`

### Requirement: Upgrade from 14.0
The extension SHALL provide an update path from `14.0` to `18.0`. Existing locked and backup statistics SHALL be preserved across the update.

#### Scenario: Update after pg_upgrade
- **WHEN** a database carrying pg_dbms_stats `14.0`, with locked and backed-up statistics, is moved to PostgreSQL 18 with `pg_upgrade`, and then `ALTER EXTENSION pg_dbms_stats UPDATE` is run
- **THEN** `extversion` becomes `18.0`, every row in `dbms_stats.relation_stats_locked`, `column_stats_locked`, `backup_history`, `relation_stats_backup`, and `column_stats_backup` is unchanged, and the objects in the `dbms_stats` schema match a fresh `18.0` install

### Requirement: Locked relation statistics are used by the planner
When `pg_dbms_stats.use_locked_stats` is on, the planner SHALL use the locked `relpages`, `reltuples`, `relallvisible`, and `curpages` for any table, materialized view, foreign table, or index that has a row in `dbms_stats.relation_stats_locked`. When the parameter is off, or no locked row exists, the planner SHALL use the real statistics. System catalogs and objects in the `pg_catalog`, `pg_toast`, `information_schema`, and `dbms_stats` schemas SHALL always use real statistics.

#### Scenario: Locked table size survives ANALYZE
- **WHEN** a table's statistics are locked with `dbms_stats.lock()`, rows are then added, `ANALYZE` runs, and a query on that table is planned
- **THEN** the row estimate in `EXPLAIN` comes from the locked `reltuples`/`relpages` density and the locked `curpages`, not from the post-ANALYZE values

#### Scenario: Feature disabled
- **WHEN** `SET pg_dbms_stats.use_locked_stats = off` is run and the same query is planned
- **THEN** the estimates match those of a server without pg_dbms_stats loaded

### Requirement: Size estimation parity with PostgreSQL 18 core
Given the locked or real pg_class values, the extension SHALL compute page, tuple, and all-visible-fraction estimates exactly as the PostgreSQL 18 planner would for the same values. The one exception is that the current page count SHALL come from the locked `curpages` when present. In particular:
- heap-like relations (tables, materialized views, TOAST) SHALL apply the 10-page minimum for never-vacuumed relations (`reltuples < 0`) without inheritance children;
- never-vacuumed heap-like relations SHALL derive density from the relation's fillfactor and clamp it to at least one row per page;
- indexes SHALL discount the metapage;
- sequences and foreign tables SHALL use the pg_class `relpages`/`reltuples` as-is with an all-visible fraction of 0;
- estimated tuple widths SHALL be clamped as in core.

#### Scenario: Never-vacuumed table with low fillfactor
- **WHEN** a table created `WITH (fillfactor = 50)` has a locked row with `reltuples = -1` and `curpages = 20`, and the estimate is compared with core's for the same pg_class values
- **THEN** the two tuple estimates are equal, and both reflect roughly half the density a fillfactor-100 table would have

#### Scenario: Sequence
- **WHEN** a sequence's size is estimated
- **THEN** pages and tuples equal the sequence's `pg_class.relpages` and `pg_class.reltuples`, and the all-visible fraction is 0

### Requirement: Locked column statistics are used by the planner
When `pg_dbms_stats.use_locked_stats` is on, selectivity estimation for a table column or an expression-index column SHALL use the row from `dbms_stats.column_stats_locked`, merged with `pg_statistic`. Locked non-NULL fields SHALL take precedence. If a locked row's histogram or MCV element type no longer matches the column type, the extension SHALL fall back to real statistics and log the mismatch. Statistics SHALL be shown to the planner only for columns the current user may SELECT, consistent with core's `acl_ok` handling, including access through views and expression indexes.

#### Scenario: Locked MCV drives selectivity
- **WHEN** a column's statistics are locked, the column's real data distribution then changes, `ANALYZE` runs, and an equality predicate on an MCV value is planned
- **THEN** the selectivity reflects the locked MCV frequency

#### Scenario: Access via view without table privilege
- **WHEN** a user with SELECT on a view, but not on its underlying table, plans a query through the view on a locked table
- **THEN** the locked statistics are applied, matching core's behavior for real statistics accessed through views

### Requirement: Cached plans are invalidated on lock changes
Inserting, updating, or deleting rows in `dbms_stats.relation_stats_locked` or `dbms_stats.column_stats_locked` SHALL invalidate cached plans that depend on the affected relation, including plans on the parent table when an index's statistics change. The extension's per-session statistics cache SHALL be refreshed before the next planning.

#### Scenario: Prepared statement replans after lock
- **WHEN** a prepared statement on table T has been executed, and then `dbms_stats.lock('T')` runs with different statistics
- **THEN** the next `EXECUTE` produces a plan based on the newly locked statistics

### Requirement: Export and import of statistics
`dbms_stats.import_*_stats()` SHALL accept binary dump files produced by the `export_plain_stats-18.sql.sample` and `export_effective_stats-18.sql.sample` scripts, from either a file or stdin, on PostgreSQL 18. It SHALL keep the existing validation: absolute paths only, no system schemas, and skipping columns whose type or typmod differ.

#### Scenario: Round-trip export and import
- **WHEN** statistics are exported with `export_plain_stats-18.sql.sample`, the locked statistics are cleared, and `dbms_stats.import_database_stats('<absolute path>')` runs
- **THEN** `dbms_stats.relation_stats_locked` and `column_stats_locked` contain the exported values for every exported non-system relation

### Requirement: Regression suite passes on PostgreSQL 18
The repository's regression suite SHALL run with `make installcheck` on PostgreSQL 18 and pass without relying on `pg_regress` `.source` file conversion. Each expected-output difference from the PG14 suite SHALL be explained by a documented PostgreSQL behavior change.

#### Scenario: installcheck
- **WHEN** the steps in `README.installcheck` are followed against a PostgreSQL 18 server and `make installcheck` runs
- **THEN** all tests (`init-common`, `ut_fdw_init`, `init`, `ut-common`, `ut`, `ut_imp_exp`) pass
