\getenv abs_srcdir PG_ABS_SRCDIR
\set csvfile :abs_srcdir '/input/ut-fdw.csv'
CREATE EXTENSION file_fdw;
CREATE SERVER test_server
       FOREIGN DATA WRAPPER file_fdw;
CREATE FOREIGN TABLE s0.sft0(id integer)
       SERVER test_server
       OPTIONS (filename :'csvfile',
				format 'csv');
