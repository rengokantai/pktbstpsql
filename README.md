## pktbstpsql
###Chapter 7. PostgreSQL Monitoring
####Checking the overall database behavior
######Checking pg_stat_activity
The application_name field can be set freely by the end user.
```
SET application_name TO 'a';
SHOW application_name;  
```
######Checking database-level information
```
\d pg_stat_database
```
some useful fields:
```
xact_commit  --  number of successful transactions
xact_rollback --  number of aborted transactions
```
Clue about IO:(not enabled by default)
```
 blk_write_time 
 blk_read_time
 ```
check
```
SHOW track_io_timing;
```
Unless track_io_timing has been activated in postgresql.conf, there is no data available.
####Detecting I/O bottlenecks
```
\d pg_stat_bgwriter 
```
####Checking for conflicts
```
\d pg_stat_database_conflicts
```
####Chasing down slow queries
Enable this query in postgresql.conf
```
shared_preload_libraries = 'pg_stat_statements'
```
use
```
CREATE EXTENSION pg_stat_statements;
```
get 2 slow queries
```
SELECT query, total_time, calls FROM   pg_stat_statements ORDER BY 2 DESC;
```
reset data
```
SELECT pg_stat_statements_reset();
```
####Inspecting internal information
######Looking inside a table
PostgreSQL uses a mechanism called Multi-Version Concurrency Control (MVCC).  
Hence, space on the disk might be wasted if too many versions of too many rows exist.(table bloat)  

use pgstattuple
```
CREATE EXTENSION pgstattuple;
SELECT * FROM pgstattuple('pg_class');
```
The dead_* columns with information about the number and size of dead rows in the table.  
Dead rows can easily be turned into free space by running VACUUM.   

######Run pgstattuple for many tables at the same time  
```
SELECT relname, (pgstattuple(oid)).* FROM   pg_class WHERE   relkind = 'r' ORDER BY table_len DESC;
```
This query has to read tables entirely so using it too often is not a good idea because it causes heavy I/O.  

######Inspecting the I/O cache
```
CREATE EXTENSION pg_buffercache;
```

###Chapter 9. Handling Hardware and Software Disasters
####Checksums 
To enable data checksums
```
initdb -k
```
Check whether a database instance has data checksums enabled
```
pg_controldata /ke | grep -i checks
```
There is no possibility to turn checksums on after initdb.

####Zeroing out damaged pages
```
SET zero_damaged_pages TO on;
```
####Dealing with index corruption
```
\h REINDEX
```
Note that REINDEX will not perform a concurrent build.  
To build the index without interfering with production you should drop the index and reissue the ```CREATE INDEX CONCURRENTLY``` command.

####Dumping individual pages
######Extracting the page header
```
SELECT * FROM page_header(get_raw_page('pg_class', 'main', 0));
```
Inspect individual tuples (rows).
```
SELECT * FROM heap_page_items(get_raw_page('pg_class', 'main', 0));
```

####Resetting the transaction log
Note: It almost always leads to some data loss, and it does not guarantee that your data will still be fully consistent.
```
pg_resetxlog --help
```
If pg_control file is not valid. To see which data is in the control
```
pg_controldata /data
```
If pg_resetxlog cannot find a valid control file, it is necessary to use -f.

###Chapter 10. A Standard Approach to Troubleshooting
####Getting an overview of the problem
```
pg_stat_activity
```
The VACUUM command can only clean up dead rows if there is no transaction around anymore that is capable of seeing the data.  
But it cannot solve all problems.(Old, idle transactions can delay the cleanup of rows)
####Attacking low performance
######Reviewing indexes
pg_stat_activity vs pg_stat_user_tables
```
SELECT relname, seq_scan, seq_tup_read, idx_scan AS idx, seq_tup_read / seq_scan AS ratio FROM  pg_stat_user_tables;
```

######Fixing UPDATE commands
To change the FILLFACTOR to 70 percent
```
ALTER TABLE foo SET (FILLFACTOR=70);
```
Change FILLFACTOR of an index
```
ALTER INDEX foo_index SET (FILLFACTOR=70);
```

######Detecting slow queries
```
SELECT (SELECT datname FROM   pg_database WHERE   dbid = oid), query, calls, total_time FROM   pg_stat_statements AS x ORDER BY total_time DESC;
```
[pg_stat_statements page](https://www.postgresql.org/docs/current/static/pgstatstatements.html)
[pg_database page](https://www.postgresql.org/docs/current/static/catalog-pg-database.html)
important attributes in pg_stat_statement
```
temp_blks_read  
temp_blks_written  
blk_read_time  
blk_write_time
```
