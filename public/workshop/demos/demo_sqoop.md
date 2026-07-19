# List tables:

    sqoop list-tables --username sqoop -P --connect jdbc:mysql://10.118.67.106/slurm_acct_db

# Clean

    hdfs dfs -rm -r -f demo/ft2_job_table*

# Import one table

    sqoop import --username sqoop -P \
        --connect jdbc:mysql://10.118.67.106/slurm_acct_db \
        --table ft2_job_table \
        --target-dir /user/jlopez/demo/ft2_job_table_1 \
        --num-mappers 1

    hdfs dfs -cat /user/jlopez/demo/ft2_job_table_1/part-m-00000 | less

# Import a table using ^a as field delimiter

    sqoop import --username sqoop -P \
        --connect jdbc:mysql://10.118.67.106/slurm_acct_db \
        --table ft2_job_table \
        --target-dir /user/jlopez/demo/ft2_job_table_2 \
        --num-mappers 1 \
        --fields-terminated-by '\001'

    hdfs dfs -cat /user/jlopez/demo/ft2_job_table_2/part-m-00000 | less

# Create only the table structure into hive

    sqoop create-hive-table --username sqoop -P \
        --connect jdbc:mysql://10.118.67.106/slurm_acct_db \
        --table ft2_job_table \
        --hive-table ft2_job_table_EMPTY

    hive> describe ft2_job_table_EMPTY;
    hive> describe formatted ft2_job_table_EMPTY;

# Import the table directly into Hive

    sqoop import --username sqoop -P \
        --connect jdbc:mysql://10.118.67.106/slurm_acct_db \
        --table ft2_job_table \
        --num-mappers 1 \
        --hive-import


    hive> describe ft2_job_table;
    hive> describe formatted ft2_job_table;
    hive> select * from ft2_job_table limit 1;
    hive> select count(*) from ft2_job_table;

