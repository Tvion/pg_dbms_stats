# Tasks

## 1. PostgreSQL 18 development environment

- [ ] 1.1 Install `postgresql-18`, `postgresql-server-dev-18`, and contrib (`file_fdw`) from PGDG apt in WSL. Verify that `/usr/lib/postgresql/18/bin/pg_config --version` prints `PostgreSQL 18.x` and that `file_fdw.control` exists under `$(pg_config --sharedir)/extension`.
- [ ] 1.2 Create a throwaway cluster with `shared_preload_libraries = 'pg_dbms_stats'` and a start/stop helper command (not committed). Verify that the cluster starts once the library is installed in group 2.

## 2. Version bump and build on PG18

- [ ] 2.1 Rename `pg_dbms_stats--14.0.sql` → `pg_dbms_stats--18.0.sql` and the doc samples `export_{plain,effective}_stats-14.sql.sample` → `-18` with `git mv`. Set `DBMSSTATSVER = 18.0` and `PGVERS = 18` in `Makefile`, and `default_version = '18.0'` in `pg_dbms_stats.control`. Verify that `git status` shows the files as renames.
- [ ] 2.2 Fix the PG18 compile and API items from design D3: `MarkGUCPrefixReserved`, remove `_PG_fini`, remove `set_acl_ok`, switch to `HASH_BLOBS`, and drop the dead `PG_VERSION_NUM` branches in `import.c` and `dump.c`. Verify that `make USE_PGXS=1 PG_CONFIG=/usr/lib/postgresql/18/bin/pg_config` builds with no errors or new warnings.
- [ ] 2.3 Install, start the cluster, and run `CREATE EXTENSION pg_dbms_stats`. Verify that `extversion = '18.0'`, that `SHOW pg_dbms_stats.use_locked_stats` returns `on`, and that `SET pg_dbms_stats.bogus = 1` is rejected.

## 3. Size-estimation parity with PG18 core

- [ ] 3.1 Re-copy `estimate_rel_size` (from `plancat.c`), `table_block_relation_estimate_size` (from `tableam.c`), and `get_rel_data_width` (from `plancat.c`) from `REL_18_STABLE` into `pg_dbms_stats.c` as described in design D2. This includes the `RELKIND_HAS_TABLE_AM` dispatch, the fillfactor density with `clamp_row_est`, `clamp_width_est`, and the "version 18" header comments. Verify that a side-by-side diff against the PG18 sources shows only the documented customizations (prefix, `curpages`, conditional smgr call).
- [ ] 3.2 Verify parity manually on the cluster. Run `EXPLAIN SELECT * FROM t` for (a) a never-analyzed `fillfactor=50` table, (b) an analyzed table, (c) an index scan, and (d) a sequence-backed query, each with a locked row that copies the real pg_class values and `curpages` = actual pages. Confirm that the row estimates equal those with `pg_dbms_stats.use_locked_stats = off`.
- [ ] 3.3 Add regression cases to `sql/ut.sql`/`expected/ut.out` for the fillfactor-50 never-vacuumed table and for a locked sequence (spec scenarios "Never-vacuumed table with low fillfactor" and "Sequence"). Verify that they pass in group 5's installcheck.

## 4. SQL objects: stats view and upgrade path

- [ ] 4.1 Extend `dbms_stats.stats` in `pg_dbms_stats--18.0.sql` with `range_length_histogram`, `range_empty_frac`, and `range_bounds_histogram` (stakind 6/7), and update its "copied from version 18" comment. Verify that `SELECT attname FROM pg_attribute WHERE attrelid IN ('dbms_stats.stats'::regclass, 'pg_stats'::regclass) AND attnum > 0` gives identical ordered lists for both views.
- [ ] 4.2 Add `pg_dbms_stats--14.0--18.0.sql`, containing only `CREATE OR REPLACE VIEW dbms_stats.stats`, to `DATA` and `TARSOURCES` in the `Makefile`. Verify the update path with `CREATE EXTENSION pg_dbms_stats VERSION '14.0'` (by temporarily installing the 14.0 script on the PG18 test cluster), locking stats on a table, running `ALTER EXTENSION pg_dbms_stats UPDATE`, and checking that `extversion = '18.0'`, that the locked rows are unchanged, and that the `\dx+ pg_dbms_stats` objects match a fresh install.
- [ ] 4.3 Add regression cases for the stats-view spec: the column list matches `pg_stats`, a locked `int4range` column shows non-NULL range columns, and an unlocked column matches `pg_stats`. Verify that they pass in group 5's installcheck.

## 5. Regression suite on PG18

- [ ] 5.1 Convert `input/ut_fdw_init.source` → `sql/ut_fdw_init.sql`, `output/ut_fdw_init.source` → `expected/ut_fdw_init.out`, and the same for `ut_imp_exp`, using `git mv`, `\getenv abs_srcdir PG_ABS_SRCDIR`, `\set` path concatenation, and `file_fdw` pointing at `input/ut-fdw.csv` (design D6). Change the `-14` sample names to `-18`. Verify that `grep -r '@abs_srcdir@' sql expected` returns nothing.
- [ ] 5.2 Update the `Makefile` `EXTRA_CLEAN` and `TARSOURCES` for the moved test files, and update `.gitignore` for the renamed `-18` sample outputs. Verify that `make clean` leaves the committed `sql/ut_*.sql` and `expected/ut_*.out` in place and that `make sourcetar` includes them.
- [ ] 5.3 Replace the `pg_sleep(...)` waits before `lockd_io` reads in `sql/ut-common.sql` with `pg_stat_force_next_flush()` + `pg_stat_clear_snapshot()` (design D7). Verify that running `ut-common` five times in a row gives identical results.
- [ ] 5.4 Run `make installcheck` on PG18, regenerate `expected/*.out` from `results/`, and classify every `git diff expected/` hunk as intended, a documented PG behavior change, or a regression (design D8). Fix any regressions in code. Verify that `make installcheck` passes cleanly and that the classification notes are saved for the MR description.
- [ ] 5.5 Update `README.installcheck` for PG18: the PGDG package names (`postgresql18-contrib` / `postgresql-contrib`) and the PG18 `pg_config` path. Verify by following it verbatim on a fresh cluster.

## 6. Packaging and documentation

- [ ] 6.1 Update `SPECS/pg_dbms_stats.spec` for PG18: `/usr/pgsql-18`, `postgresql18-*` requirements, the summary and description text, `Version: 18.0`, the `%files` entries for `--18.0.sql`, the upgrade script, and the `-18` samples, and a `%changelog` entry. Verify with `rpmspec -P SPECS/pg_dbms_stats.spec` (or `make rpm18` on an RPM host) that it parses and lists the expected files.
- [ ] 6.2 Update `doc/pg_dbms_stats-en.md`, `doc/pg_dbms_stats-ja.md`, `doc/objects-*.md`, and `README.md`. Cover the supported version (PG18), the new `dbms_stats.stats` columns, the `pg_upgrade` + `ALTER EXTENSION ... UPDATE` path, the fact that `pg_dump` does not carry locked statistics (use export/import), and the estimate changes (fillfactor, sequences). Bump the copyright years in touched file headers. Verify that every documented command and file name exists in the tree.

## 7. Integration check

- [ ] 7.1 On a clean PG18 cluster, from a fresh `git clone`, run `make && make install && make installcheck`, the 14.0 → 18.0 upgrade check from 4.2, and the export/import round trip from the spec scenario. Verify that all pass, and open a GitLab MR (conventional commit `feat: support PostgreSQL 18`) with the D8 diff classification in the description.
