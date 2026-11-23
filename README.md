# pg_dbms_stats

pg_dbms_statsは、PostgreSQLが実行計画作成に利用する統計情報を管理し、間接的に実行計画を制御できるようにします。

特定の状態の統計情報を保存しておき、実行計画作成時に最新の統計情報ではなく保存しておいた統計情報を見せることができます。 これにより、運用中に実行計画が急に変化して、クエリの性能が不安定化するリスクを抑えることが可能です。

[日本語のマニュアルはこちら](/doc/pg_dbms_stats-ja.md)

[English version here](/doc/pg_dbms_stats-en.md)

このリリース（18.0）は PostgreSQL 18 向けです。`dbms_stats.stats` は PostgreSQL 18 の `pg_stats` と同じ列を持ち、`range_length_histogram`、`range_empty_frac`、`range_bounds_histogram` を含みます。

PostgreSQL 14 から `pg_upgrade` した場合は、各データベースで `ALTER EXTENSION pg_dbms_stats UPDATE;` を実行してください。ロック済みの統計情報はそのまま残ります。PostgreSQL 9.6（1.3.7）からも同じ手順で 18.0 に更新できます。ただし更新が終わるまで `shared_preload_libraries` に `pg_dbms_stats` を入れないでください。詳細はマニュアルの「PostgreSQL 9.6（pg_dbms_stats 1.3.7）から PostgreSQL 18 への移行」を参照してください。`pg_dump` はロック済み統計情報を出力しないので、ダンプで移す場合は `doc/export_effective_stats-18.sql.sample` で書き出し、移設先で `dbms_stats.import_database_stats()` を使って読み込みます。

行数の見積もりは PostgreSQL 18 に合わせています。シーケンスは `pg_class` の値を使い、未バキューム（`reltuples < 0`）のテーブルは fillfactor を考慮します。

-----
Copyright (c) 2012-2022, NIPPON TELEGRAPH AND TELEPHONE CORPORATION
