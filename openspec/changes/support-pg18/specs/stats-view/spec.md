# Spec Delta

## Purpose

Defines the `dbms_stats.stats` view. It presents the effective statistics (locked where present, real otherwise) in the same shape as PostgreSQL's `pg_stats`, so that tools and users can inspect what the planner sees.

## ADDED Requirements

### Requirement: Column set matches pg_stats
`dbms_stats.stats` SHALL expose the same column names, in the same order and with compatible types, as PostgreSQL 18's `pg_catalog.pg_stats`. The columns are: `schemaname`, `tablename`, `attname`, `inherited`, `null_frac`, `avg_width`, `n_distinct`, `most_common_vals`, `most_common_freqs`, `histogram_bounds`, `correlation`, `most_common_elems`, `most_common_elem_freqs`, `elem_count_histogram`, `range_length_histogram`, `range_empty_frac`, and `range_bounds_histogram`.

#### Scenario: Column list comparison
- **WHEN** the column names of `dbms_stats.stats` and `pg_catalog.pg_stats` are listed in `attnum` order
- **THEN** the two lists are identical

### Requirement: Values come from effective statistics
Each row of `dbms_stats.stats` SHALL report the effective statistics for one column: the merge of `dbms_stats.column_stats_locked` and `pg_statistic`. Slot-derived columns SHALL be taken from the slot whose `stakind` matches the kind used by `pg_stats`: 1 MCV, 2 histogram, 3 correlation, 4 MCELEM, 5 DECHIST, 6 range length histogram, and 7 range bounds histogram.

#### Scenario: Range column with locked stats
- **WHEN** a table with an `int4range` column is analyzed, its statistics are locked, and `dbms_stats.stats` is queried for that column
- **THEN** `range_length_histogram`, `range_empty_frac`, and `range_bounds_histogram` are non-NULL and equal the locked slot values

#### Scenario: Unlocked column matches pg_stats
- **WHEN** a column with no locked statistics is queried in both `dbms_stats.stats` and `pg_stats`
- **THEN** the two rows have equal values in every column

### Requirement: Row visibility matches pg_stats
`dbms_stats.stats` SHALL be a security-barrier view. It SHALL show a column's row only when the current user has SELECT privilege on that column, and only when row-level security is disabled or not active for the table. Dropped columns SHALL NOT be shown.

#### Scenario: Column without privilege
- **WHEN** a user without SELECT on column `c` queries `dbms_stats.stats`
- **THEN** no row for `c` is returned
