# syntax=docker/dockerfile:1
# Isolated build and test image for pg_dbms_stats on PostgreSQL 18.
FROM postgres:18
SHELL ["/bin/bash", "-c"]

RUN apt-get update \
 && apt-get install -y --no-install-recommends build-essential postgresql-server-dev-18 \
 && rm -rf /var/lib/apt/lists/*

COPY . /src/pg_dbms_stats
RUN chown -R postgres:postgres /src

WORKDIR /src/pg_dbms_stats
USER postgres
RUN set -o pipefail; make USE_PGXS=1 clean && make USE_PGXS=1 2>&1 | tee /tmp/build.log
USER root
RUN make USE_PGXS=1 install

COPY <<'SCRIPT' /usr/local/bin/run-tests.sh
#!/bin/bash
set -u
export PGDATA=/tmp/pgdata
initdb -D "$PGDATA" --encoding=UTF8 >/dev/null
echo "shared_preload_libraries = 'pg_dbms_stats'" >> "$PGDATA/postgresql.conf"
pg_ctl -w -D "$PGDATA" -l /tmp/pg.log start >/dev/null
cd /src/pg_dbms_stats
make USE_PGXS=1 installcheck "$@"
rc=$?
echo "=== installcheck exit: $rc ==="
[ -f regression.diffs ] && head -c 20000 regression.diffs
pg_ctl -D "$PGDATA" stop >/dev/null
exit $rc
SCRIPT
RUN chmod +x /usr/local/bin/run-tests.sh
USER postgres
CMD ["/usr/local/bin/run-tests.sh"]
