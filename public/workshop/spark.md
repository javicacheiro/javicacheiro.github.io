Spark
=====

Run an interactive shell
------------------------
 
    spark-shell --master yarn --num-executors 4 --executor-cores 6
    pyspark --master yarn --num-executors 4 --executor-cores 6

    --num-executors NUM         Number of executors to launch (Default: 2).
    --executor-cores NUM        Number of cores per executor. (Default: 1 in YARN mode)
    --driver-cores NUM          Number of cores used by the driver, only in cluster mode (Default: 1).
    --executor-memory MEM       Memory per executor (e.g. 1000M, 2G) (Default: 1G).
    --queue QUEUE_NAME          The YARN queue to submit to (Default: "default").
    --proxy-user NAME           User to impersonate when submitting the application.


YARN: The --num-executors option to the Spark YARN client controls how many executors it will allocate on the cluster (spark.executor.instances as configuration property), while --executor-memory (spark.executor.memory configuration property) and --executor-cores (spark.executor.cores configuration property) control the resources per executor. For more information, see the YARN Spark Properties


Submit job to queue
-------------------

Python:

    spark-submit --master yarn --name testWC test.py
    spark-submit --master yarn --deploy-mode cluster --name testWC test.py

Scala/Java:

    spark-submit --master yarn                       --name testPi --class org.apache.spark.examples.SparkPi /usr/hdp/2.4.2.0-258/spark/lib/spark-examples-1.6.1.2.4.2.0-258-hadoop2.7.1.2.4.2.0-258.jar 10
    spark-submit --master yarn --deploy-mode cluster --name testPi --class org.apache.spark.examples.SparkPi /usr/hdp/2.4.2.0-258/spark/lib/spark-examples-1.6.1.2.4.2.0-258-hadoop2.7.1.2.4.2.0-258.jar 10


Dynamic Resource Allocation
---------------------------

Spark provides a mechanism to dynamically adjust the resources your application occupies based on the workload. This means that your application may give resources back to the cluster if they are no longer used and request them again later when there is demand. This feature is particularly useful if multiple applications share resources in your Spark cluster.

This feature is disabled by default and available on all coarse-grained cluster managers, i.e. standalone mode, YARN mode, and Mesos coarse-grained mode.


Dataframes
----------

```
from pyspark.sql import Row
models = sc.textFile("data/models.csv")
modelsRows = models.map(lambda l: l.split(",")).map(lambda x: Row(sku=x[0], model=x[1], price=float(x[2])))
modelsDF = modelsRows.toDF()

modelsDF.select("sku").show()
modelsDF.select(modelsDF.sku).show()
modelsDF.select(modelsDF['sku']).show()

modelsDF.registerTempTable("models")
sqlContext.sql(''select * from models limit 10'')')

```


```
>>> d = [{'name': 'Alice', 'age': 1}]
>>> sqlContext.createDataFrame(d).collect()
[Row(age=1, name=u'Alice')]

>>> from pyspark.sql import Row
>>> Person = Row('name', 'age')
>>> person = rdd.map(lambda r: Person(*r))
>>> df2 = sqlContext.createDataFrame(person)
>>> df2.collect()
[Row(name=u'Alice', age=1)]

>>> from pyspark.sql.types import *
>>> schema = StructType([
...    StructField("name", StringType(), True),
...    StructField("age", IntegerType(), True)])
>>> df3 = sqlContext.createDataFrame(rdd, schema)
>>> df3.collect()
[Row(name=u'Alice', age=1)]

```

Loading data
------------
Load a local file (DOES NOT WORK!!):

    sc.textFile("file:/mnt/gluster/drv0/home/cesga/jlopez/example.json")

Read from HDFS:

    sc.textFile("hdfs://c13-19.node.int.cesga.es:8020/user/jlopez/derby.log")

Overriding configuration directory
----------------------------------
To specify a different configuration directory other than the default “SPARK_HOME/conf”, you can set SPARK_CONF_DIR. Spark will use the configuration files (spark-defaults.conf, spark-env.sh, log4j.properties, etc) from this directory.

Example:

    export SPARK_CONF_DIR=/home/cesga/jlopez/conf/

    export SPARK_CONF_DIR=/home/usc/fa/jlc/conf

SparkR
-------

  >sparkR
  sqlContext <- sparkRSQL.init(sc)
  df <- createDataFrame(sqlContext, faithful)
  head(df)

You can use it also from Jupyter notebook. Just start Jupyter using:

  start_jupyter

And select: new -> R

References: 
  http://hortonworks.com/hadoop-tutorial/a-lap-around-apache-spark/
  https://spark.apache.org/docs/1.6.1/sparkr.html 
