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
