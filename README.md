## pktbstpsql
###Chapter 1. Installing PostgreSQL
####Deciding on a version number
version number a.b.c  
If a or b changed,a migration process is needed.


####Memory and kernel issues
######Adjusting kernel parameters for Mac OS X
```
sysctl -w kern.sysv.shmmax
sysctl -w kern.sysv.shmmin
sysctl -w kern.sysv.shmmni
sysctl -w kern.sysv.shmseg
sysctl -w kern.sysv.shmall
```
######Fixing other kernel-related limitations
If run a large-scale system, raise the maximum number of open files allowed.
```
vim /etc/security/limits.conf
```
edit
```
postgres    hard    nofile    1024
postgres    soft    nofile    1024
```

####Adding checksums to a database instance
```
initdb -k
```

####Preventing encoding-related issues
```
CREATE DATABASE name ENCODING =
```


####Avoiding template pollution
######Killing the postmaster
When start PostgreSQL we are starting a process called postmaster.  
Whenever a new connection comes in, this postmaster forks and creates a backend process (BE).  
This process is in charge of handling exactly one connection.   

###Chapter 2. Creating Data Structures
####Grouping columns the right way

###Chapter 6. Writing Proper Procedures
####Choosing the right language
```
CREATE LANGUAGE
```

####Managing procedures and transactions
create a function concat strings
```
CREATE FUNCTION ksum(text, text) RETURNS text AS ' SELECT $1 || $2 ' LANGUAGE 'sql';
```
invoke
```
SELECT ksum('1', '2');  -- 12
```

######Understanding transactions and procedures
```
CREATE FUNCTION fail(int) RETURNS int AS $$
 DECLARE
  one = ALIAS FOR $1;
 BEGIN
  RETURN one/0;
 EXCEPTION WHEN division_by_zero THEN
  RETURN 0;
 END;
$$ LANGUARE 'plpgsql';
```

####Procedures and indexing
(tbc)

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

equivalent query
```
relfilenode: (SELECT relname FROM pg_class WHERE relfilenode = 'this oid';)
reltablespace: (SELECT spcname FROM pg_tablespace WHERE oid = 'this oid';)
reldatabase: (SELECT datname FROM pg_database WHERE oid = 'this oid';)
```

###Chapter 8. Fixing Backups and Replication
####Using pg_dump
review: basic command
```
pg_dump tbname > /dump.sql
psql tbname < /dump.sql
```
blobs using blob option:
```
-b (--blobs) 
```
######Handling passwords .pgpass
format
```
hostname:port:database:username:password
```
######Creating custom format dumps
```
pg_dump tbname -Fc > /dump.fc
pg_restore --list /dump.fc
```
we can restore subset of dump:
```
pg_restore -t t_tbname /dump.fc
```
(tbc)
####Managing point-in-time recovery
#####How PITR works
Take a snapshot of the data and archive the transaction log from then on.


######Preparing PostgreSQL for PITR
create folder
```
mkdir /archive
```
in postgresql.conf edit
```
wal_level = archive
archive_mode = on 
archive_command = 'cp %p /archive/%f '
```
######Taking base backups
(tbc)


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
