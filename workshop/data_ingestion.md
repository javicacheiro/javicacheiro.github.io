Data Ingest
-----------

Copy compressed data to GlusterFS HOME directory using rsync.
- rsync from EMC VNX COMPARTIDO fs6801 to c13-5 we get 50MB/s

Modify files in GlusterFS:
- Uncompress, rename, ...

Upload to HDFS:

HDFS command line

    hdfs dfs
        -help
        -setrep <rep> <path> : Set the replication level to a file or to all files in a dir
        -put
        -get
        -ls -h -R
        -mkdir
        -chown
        -rm -r -f
        -cat
        -tail
        -head
        -getmerge
        -cp
        -du
        -df
        -find <path> ... <expression> ...
        -appendToFile
        -getfacl
        -stat [format] <path> : format example '%n %o %r' -> name blocksize replication
        -expunge : Delete files from the trash that are older than the retention threshold


