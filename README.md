## pktbstpsql
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
