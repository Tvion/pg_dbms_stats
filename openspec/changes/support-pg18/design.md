# Design

## Context

See proposal.md for the motivation. The current tree is a PG14-only release (`14.0`). These findings come from comparing it against the `REL_18_STABLE` sources:

- **Hooks and APIs are unchanged.** The PG18 signatures of `planner_hook_type`, `get_relation_info_hook_type`, `get_relation_stats_hook_type`, `get_index_stats_hook_type`, and `get_attavgwidth_hook_type` match PG14. The same holds for `VariableStatData` (still ending in `acl_ok`), `DoCopy(ParseState*, const CopyStmt*, int, int, uint64*)`, `get_sort_group_operators`, `pq_puttextmessage`, `SetUserIdAndSecContext`, and `pg_class_aclcheck`/`pg_attribute_aclcheck`. `EmitWarningsOnPlaceholders` and `oid_hash` survive only as compatibility macros.
- **The catalogs are compatible.** The `pg_statistic` column list is identical, so `column_stats_locked`, `column_stats_backup`, `dbms_stats.merge(column_stats_locked, pg_statistic)`, and the binary export format all stay valid. `pg_class` gains `relallfrozen` (PG18), which the planner does not read.
- **The copied core code has drifted** (`pg_dbms_stats.c`, the `dbms_stats_*estimate*` and `dbms_stats_get_rel_data_width` functions):
  - `estimate_rel_size` (PG15+) dispatches on `RELKIND_HAS_TABLE_AM`. Sequences fall into the "use pg_class as-is" branch instead of being hardcoded to 1 page and 1 tuple.
  - `table_block_relation_estimate_size` (PG17+) computes never-vacuumed density as `(usable_bytes * fillfactor / 100) / tuple_width` and wraps it in `clamp_row_est()`.
  - `get_rel_data_width` (PG17+) accumulates the width in `int64` and returns `clamp_width_est()`.
- **`pg_regress` (PG15+)** no longer converts `input/*.source` and `output/*.source`. It exports `PG_ABS_SRCDIR`/`PG_ABS_BUILDDIR` instead. `ut_fdw_init` and `ut_imp_exp` depend on `@abs_srcdir@` substitution.
- **Cumulative statistics (PG15+)** live in shared memory. A backend's own counters become visible only after a flush, and reads are snapshotted (`stats_fetch_consistency = cache`). `ut-common.sql` uses `pg_sleep(0.7)` before reading `lockd_io`, which relies on the old collector's timing.
- **No PG18 toolchain** is installed on the workstation. Only a psql 16 client exists, and there is no `pg_config`.

## Goals / Non-Goals

**Goals:**
- Keep the code a minimal, reviewable delta against the existing tree, in the same style as the 13 → 14 port (commits `a62cb2e`, `e041b7c`).
- Make the copied estimation code behave the same as PG18 core, so locked statistics never yield different estimates than core would from the same numbers.
- Make the regression suite deterministic on PG18.

**Non-Goals:**
- Keeping PG14 support in the same tree with `#if PG_VERSION_NUM` branches.
- Refactoring the hook architecture, or replacing SPI lookups with something else.
- Fixing unrelated latent issues found during review, beyond noting them. One example: the MCV/histogram type check in `dbms_stats_merge_internal` compares the slot index with `STATISTIC_KIND_*` instead of comparing `stakindN`.

## Decisions

### D1. Single-target release `18.0`, with no multi-version `#if`s
Follow the existing convention: `DBMSSTATSVER = 18.0`, `PGVERS = 18`, `default_version = '18.0'`, and rename `pg_dbms_stats--14.0.sql` → `pg_dbms_stats--18.0.sql` with `git mv` to keep history.
*Alternative*: keep one tree that builds on 14–18 using `PG_VERSION_NUM` guards. This was rejected because the project has never done it, the copied planner code would need per-version variants, and expected outputs would differ per version.

### D2. Re-copy the estimation functions from PG18 rather than patch them
Replace the bodies of `dbms_stats_estimate_rel_size`, `dbms_stats_table_relation_estimate_size`, and `dbms_stats_get_rel_data_width` with the PG18 originals. Then re-apply the same, already documented customizations: the `dbms_stats_` prefix, the extra `curpages` parameter, and calling `RelationGetNumberOfBlocks` only when `curpages == InvalidBlockNumber`. Update the header comments to say "version 18". Keep the explicit heap overhead constants (`HEAP_OVERHEAD_BYTES_PER_TUPLE`, `HEAP_USABLE_BYTES_PER_PAGE`), which match `heapam_estimate_rel_size`.

The top-level `switch` becomes the PG18 shape: `RELKIND_HAS_TABLE_AM` → table path, `RELKIND_INDEX` → index path, else → pg_class as-is. This needs `access/tableam.h` for the macro, `utils/rel.h` for `RelationGetFillFactor`/`HEAP_DEFAULT_FILLFACTOR`, and `optimizer/optimizer.h` for `clamp_row_est`/`clamp_width_est`.
*Alternative*: call core's `estimate_rel_size()` after temporarily overriding `rd_rel` fields. This was rejected because core always calls the smgr for the current page count, so it cannot honor locked `curpages`.

### D3. Version-gate and API cleanups
- `EmitWarningsOnPlaceholders("pg_dbms_stats")` → `MarkGUCPrefixReserved("pg_dbms_stats")`.
- Remove `_PG_fini` and its prototype. PG15+ never unloads shared libraries, so the function is dead code. Hooks are still chained to the `prev_*` pointers.
- Remove the `set_acl_ok` minor-version detection, which is always true on PG ≥ 10. Set `vardata->acl_ok` unconditionally.
- Drop the dead `#if PG_VERSION_NUM >= 90300` / `>= 100000` / `#elif` branches in `import.c` and `dump.c`, keeping only the modern path.
- `ctl.hash = oid_hash` with `HASH_FUNCTION` → `HASH_BLOBS` with no custom hash, the idiomatic replacement for the `oid_hash` macro.
- Keep `PG_MODULE_MAGIC`. `PG_MODULE_MAGIC_EXT(.name, .version)` is optional, and adding it is a cosmetic choice that would put the version string in two places.
- Under `UNIT_TEST`, `import.c` calls `SPI_restore_connection()`, which was removed in PG10. The unit-test build is already broken on PG14, so this stays out of scope and is recorded as a known issue.

### D4. `dbms_stats.stats` gains the PG17 range columns
Append `range_length_histogram` (stakind 6, `stavalues`), `range_empty_frac` (stakind 6, `stanumbers[1]`), and `range_bounds_histogram` (stakind 7, `stavalues`) to the view, as in PG18 `system_views.sql`. Update the comment to say "copied from version 18". The columns are appended, so `CREATE OR REPLACE VIEW` can apply them in the upgrade script.

### D5. Upgrade script `pg_dbms_stats--14.0--18.0.sql`
Its only content is `CREATE OR REPLACE VIEW dbms_stats.stats ...` with the new definition, because the tables, types, and functions are unchanged and C symbols resolve through `MODULE_PATHNAME`. This makes the path `pg_upgrade` → `ALTER EXTENSION pg_dbms_stats UPDATE` work. Add it to `DATA` and `TARSOURCES` in the `Makefile` and to `%files` in the RPM spec.
*Alternative*: no upgrade script, as in 13 → 14. This was rejected because `pg_upgrade` users would be stuck on catalog version `14.0` with a stale view.

### D6. Regression tests use `\getenv` instead of `.source` conversion
- Move `input/ut_fdw_init.source` → `sql/ut_fdw_init.sql`, `output/ut_fdw_init.source` → `expected/ut_fdw_init.out`, and the same for `ut_imp_exp`, keeping history with `git mv`. Delete the `input/*.source` and `output/` files, but keep `input/ut-fdw.csv`.
- At the top of each file, put `\getenv abs_srcdir PG_ABS_SRCDIR`, then build paths with `\set` concatenation (for example, `\set dump_file :abs_srcdir '/export_stats.dmp'`) and use `:'dump_file'` in `COPY ... FROM` and in `dbms_stats.import_database_stats(:'dump_file')`.
- psql does not interpolate variables inside `\!`. So `\!` lines use paths relative to the regress working directory (the source directory under PGXS), as the existing `sed doc/...` lines already do. Likewise, the `file_fdw` `filename` option points at `:'abs_srcdir' '/input/ut-fdw.csv'`, so no `\! cp` is needed.
- Replace the hardcoded `export_plain_stats-14` / `export_effective_stats-14` names in `ut_imp_exp` with `-18`.
- In the `Makefile`, remove the generated `sql/ut_*`/`expected/ut_*` entries from `EXTRA_CLEAN`, and in `TARSOURCES` replace `input/*.source output/*.source` with the new `sql/ut_*.sql expected/ut_*.out` and `input/ut-fdw.csv`.
*Alternative*: generate the files with a Makefile `sed` rule. This was rejected because PG core itself moved to `\getenv`, and generated test files complicate `git status` and tarballs.

### D7. Deterministic stats-counter checks
Before each `SELECT * FROM lockd_io`, replace each `SELECT pg_sleep(0.7)` with `SELECT pg_stat_force_next_flush();` followed by `SELECT pg_stat_clear_snapshot();`. Because psql sends each statement separately, the flush happens when the backend goes idle after the first call, and the snapshot clear ensures a fresh read. Apply the same treatment after `pg_stat_reset()`.
*Alternative*: keep the sleeps but lengthen them. This was rejected because it is still racy and slows the suite.

### D8. Expected-output regeneration with review
Regenerate `expected/*.out` on PG18, and classify every hunk of `git diff` as (a) an intended change (paths, the version string `18.0`, the new view columns), (b) a documented PG15–18 behavior change (for example, error message wording, `EXPLAIN` formatting, sequence size estimates, or fillfactor-aware estimates), or (c) a regression, which must be fixed rather than accepted. Record the classification in the merge request description.

### D9. Development environment
Install PostgreSQL 18 from PGDG apt (`postgresql-18`, `postgresql-server-dev-18`, and `postgresql-contrib`, which provides `file_fdw`) inside WSL. Build with `PG_CONFIG=/usr/lib/postgresql/18/bin/pg_config`. Run a throwaway cluster with `shared_preload_libraries = 'pg_dbms_stats'` for `make installcheck`, following `README.installcheck`.

## Risks / Trade-offs

- [Plan-shape diffs from PG15–18 planner changes could hide a real regression in the large expected outputs] → Apply the D8 hunk classification. Also run a targeted check, outside the suite, that compares `EXPLAIN` row estimates with `use_locked_stats = on` versus `off` for unlocked tables. They must be identical.
- [The fillfactor and sequence behavior changes alter estimates for some locked relations compared with 14.0] → This is intended (parity with core, as specified). Document it in the release notes and ChangeLog.
- [Expanding `dbms_stats.stats` breaks dependent views or `SELECT *` consumers that expect 14 columns] → The columns are appended only. Note it in the release notes.
- [`HASH_BLOBS` changes the hash function, though not the semantics, for the per-session relation cache] → The cache is per-backend and never persisted, so there is no compatibility impact.
- [`pg_stat_force_next_flush` semantics differ under extended query protocol] → `pg_regress` uses psql's simple protocol, so flushes happen at idle as expected.
- [There is no PG18 toolchain locally] → D9 is the first implementation task, and nothing else can be validated before it.

## Migration Plan

1. Users on PG14 stay on pg_dbms_stats 14.0.
2. When moving to PG18: install the pg_dbms_stats 18.0 package on the new cluster, run `pg_upgrade`, and then run `ALTER EXTENSION pg_dbms_stats UPDATE;` in each database that has the extension. Locked and backup statistics are carried over unchanged.
3. Alternatively, run `pg_dump`/restore. Restoring creates the extension at the new default version `18.0`. However, the `dbms_stats` tables are not registered with `pg_extension_config_dump`, in 14.0 or in this change, so locked and backup statistics are **not** dumped. Users of this path must first export the statistics with `export_effective_stats-<ver>.sql.sample`, then re-import them with `dbms_stats.import_database_stats()` on PG18. The README/docs must state this.
4. Rollback: the 18.0 SQL changes only a view definition. To roll back, reinstall the 14.0 package on a PG14 cluster. PG18 binaries cannot be used with PG14.

## Open Questions

- Should `PG_MODULE_MAGIC_EXT(.name = "pg_dbms_stats", .version = "18.0")` be adopted for the `pg_get_loaded_modules()` reporting added in PG18? This is cosmetic and can be decided during review.
- Should the latent stakind check in `dbms_stats_merge_internal` (noted in Non-Goals) get its own follow-up issue?
