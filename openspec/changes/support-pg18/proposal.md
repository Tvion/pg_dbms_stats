# Proposal

## Why

pg_dbms_stats 14.0 only builds, installs, and passes its regression suite on PostgreSQL 14, so users cannot run it on PostgreSQL 18. PG 15–18 changed things the extension depends on. It carries copies of core size-estimation code that now differ from core. The regression suite uses `pg_regress` `.source` conversion, which was removed in PG15. The `pg_stats` view that `dbms_stats.stats` mirrors gained columns in PG17.

## What Changes

- Release pg_dbms_stats **18.0**, targeting PostgreSQL 18 only. This follows the repository's one-release-per-major convention (13 → 1.5.0, 14 → 14.0). **BREAKING**: the 18.0 build no longer targets PG14 (the 14.0 release remains for PG14 users).
- Re-sync the core code copied into `pg_dbms_stats.c` (`estimate_rel_size`, `table_block_relation_estimate_size`, `get_rel_data_width`) with PG18. Locked statistics must keep producing the same estimates core would produce from the same pg_class values. This covers sequences using pg_class values (PG15), the fillfactor-aware density for never-vacuumed tables (PG17), and width clamping (PG17).
- Clean up deprecated or no-op APIs. Replace `EmitWarningsOnPlaceholders` with `MarkGUCPrefixReserved`, and drop the `_PG_fini` hook restore, which PG15+ never calls. Remove obsolete pre-PG10 `PG_VERSION_NUM` branches and the pre-9.x `set_acl_ok` version gate.
- Extend `dbms_stats.stats` with the `range_length_histogram`, `range_empty_frac`, and `range_bounds_histogram` columns, so it matches PG18's `pg_stats` again.
- Ship `pg_dbms_stats--18.0.sql`, plus a `pg_dbms_stats--14.0--18.0.sql` upgrade script so that databases moved with `pg_upgrade` can run `ALTER EXTENSION pg_dbms_stats UPDATE`.
- Rename the version-specific artifacts: the doc samples `export_*_stats-18.sql.sample`, the RPM spec (`/usr/pgsql-18`, `postgresql18-*`), the `Makefile` `DBMSSTATSVER`/`PGVERS`, the control file, and the README/docs.
- Port the regression suite to PG18's `pg_regress`. Replace `input/*.source` and `output/*.source` with plain `sql/` and `expected/` files that use `\getenv abs_srcdir PG_ABS_SRCDIR`. Regenerate the expected outputs against PG18 and review the diffs for behavior changes, not just accept them.

## Capabilities

### New Capabilities

- `pg18-compatibility`: covers the build, load, packaging, upgrade path, and planner-statistics behavior of pg_dbms_stats on PostgreSQL 18. This includes parity with core size estimation and a passing regression suite.
- `stats-view`: covers the contents of the `dbms_stats.stats` view, a `pg_stats`-shaped view over the effective (locked-or-real) statistics.

### Modified Capabilities

None. `openspec/specs/` has no existing capabilities.

## Impact

- **C sources**: `pg_dbms_stats.c` (size-estimation copies, `_PG_init`/`_PG_fini`, GUC registration) and `import.c`/`dump.c` (version `#if` cleanup only). The hook signatures (`planner_hook`, `get_relation_info_hook`, `get_relation_stats_hook`, `get_index_stats_hook`, `get_attavgwidth_hook`), `VariableStatData.acl_ok`, `DoCopy`, and the `pg_statistic` layout are unchanged in PG18, so no hook rewrites are expected.
- **SQL**: `pg_dbms_stats--14.0.sql` becomes `pg_dbms_stats--18.0.sql`, with the `dbms_stats.stats` view extended. A new upgrade script is added. The on-disk table layouts (`relation_stats_locked`, `column_stats_locked`, and the backup tables) are unchanged.
- **Build/packaging**: `Makefile`, `pg_dbms_stats.control`, and `SPECS/pg_dbms_stats.spec`. The doc samples are renamed to `-18`.
- **Tests**: `input/`, `output/`, `sql/`, `expected/`, and the Makefile `REGRESS`/`EXTRA_CLEAN`/`TARSOURCES` lists.
- **Dev environment**: PostgreSQL 18 server and dev headers (`pg_config`, `postgresql-server-dev-18`, `file_fdw`) are required to build and run `make installcheck`. They are not installed on the current workstation.
- **Out of scope**: locking the new `pg_class.relallfrozen` (the PG18 planner does not read it), integrating with PG18's `pg_restore_*_stats()` functions, and supporting multiple PG majors from one source tree.
