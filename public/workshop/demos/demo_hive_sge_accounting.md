# Create Hive table
```
-- SGE Accounting format specification:
--   [1] http://gridscheduler.sourceforge.net/htmlman/htmlman5/accounting.html
--   [2] http://man7.org/linux/man-pages/man2/getrusage.2.html 

CREATE EXTERNAL TABLE jobs (
    qname string, hostname string, groupname string, owner string,
    jobname string, jobnumber bigint, account string, priority int,
    qsub_time bigint, start_time bigint, end_time bigint,
    failed int, exit_status int, ru_wallclock bigint,
    ru_utime double, ru_stime double,
    ru_maxrss bigint, ru_ixrss bigint, ru_ismrss bigint, ru_idrss bigint, ru_isrss bigint,
    ru_minflt bigint, ru_majflt bigint, ru_nswap bigint, ru_inblock bigint, ru_oublock bigint,
    ru_msgsnd bigint, ru_msgrcv bigint, ru_nsignals bigint, ru_nvcsw bigint, ru_nivcsw bigint,
    project string, department string, granted_pe string,
    slots int, na1 string, cpu double, mem double, na2 string,
    command_line_arguments string, na3 string, na4 string, maxvmem_bytes bigint
  )
ROW FORMAT DELIMITED
  FIELDS TERMINATED BY ':'
LOCATION '/user/jlopez/data/sge';
```

# Upload sample data
```
cd /home/cesga/jlopez/accounting/FT/accounting_sge/diario/uncompressed
hdfs dfs -put 20160101 data/sge
```

# Query data
```
hive> SELECT count(*) AS total FROM jobs;
4823
Time taken: 8.734 seconds, Fetched: 1 row(s)
hive> SELECT sum(ru_wallclock) AS etime FROM jobs WHERE owner='msotillo';
hive> SELECT owner,sum(ru_wallclock)/3600 AS etime FROM jobs GROUP BY owner;
hive> SELECT owner,sum(ru_wallclock)/3600 AS etime FROM jobs GROUP BY owner SORT BY etime DESC LIMIT 10;
```

# Now with the full dataset
Now we will use the full dataset 2008-2016: 5GB, 2840 files
```
DESCRIBE FORMATTED jobs;
ALTER TABLE jobs SET LOCATION 'hdfs://c13-19.node.int.cesga.es:8020/accounting/sge/FT';
```

Perform a query against the whole dataset:
```
hive> SELECT count(*) AS total FROM jobs;
12757226
Time taken: 28.1 seconds, Fetched: 1 row(s)

hive> SELECT owner,sum(ru_wallclock)/3600 AS etime FROM jobs GROUP BY owner SORT BY etime DESC LIMIT 10;
...
2772838 1.0215526849391111E10
ulctidrc        1852843.6469444444
csedamsp        999163.1966666667
csfamorv        757732.4427777778
ulcqidrr        740642.4316666666
uscfaapg        659184.7275
uscfadsa        433767.0275
uscfampl        399284.9147222222
cseemcca        351715.2975
ulctiefb        345775.88333333336
Time taken: 25.911 seconds, Fetched: 10 row(s)

```

# Outliers
We should clean data:
```
select count(*) as total from jobs where owner='2772838';
```

