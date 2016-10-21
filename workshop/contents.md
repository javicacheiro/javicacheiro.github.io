
Temario
=======
1. Introducción al servicio Big Data
1.1. Conceptos básicos
1.2. Descripción del hardware
1.3. Arquitectura de la plataforma

2. Utilización del servicio Big Data del CESGA
2.1. HDFS
2.2. Ejecución y monitorización de trabajos en YARN
2.3. Herramientas disponibles

3. Ejemplo de Uso
3.1. Ingesta de Datos con Sqoop
3.2. Transformacion de los datos con Spark
3.3. Analisis de datos con Hive

Contents
========
### Intro
- Motivation: MPI is dead
- Hardware description
- Software description: available tools
    - Hadoop distributions: HDP, CDH, MapR

### CESGA HDP Platform
- Available tools overview
- GlusterFS
- Anaconda Python
- HUE, jupyter?
- How to connect
- Big data portal: Proxy service
- How to launch a Jupyter notebook
- [x] YARN
- [x] MapReduce from Java: mr-wordcount, reuse from original presentation
- Spark
- Status monitoring: Ambari?

### Usage
- Data ingest:
    - HDFS
    - Sqoop
    - Flume

- Data transformation
    - Spark
    - MapReduce

- Data analysis
    - Hive



Cloudera Spark Certified Developer
==================================

Data Ingest
-----------

The skills to transfer data between external systems and your cluster. This includes the following:

    Import data from a MySQL database into HDFS using Sqoop
    Export data to a MySQL database from HDFS using Sqoop
    Change the delimiter and file format of data during import using Sqoop
    Ingest real-time and near-real time (NRT) streaming data into HDFS using Flume
    Load data into and out of HDFS using the Hadoop File System (FS) commands

Transform, Stage, Store
-----------------------

Convert a set of data values in a given format stored in HDFS into new data values and/or a new data format and write them into HDFS. This includes writing Spark applications in both Scala and Python (see note above on exam question format for more information on using either Scale or Python):

    Load data from HDFS and store results back to HDFS using Spark
    Join disparate datasets together using Spark
    Calculate aggregate statistics (e.g., average or sum) using Spark
    Filter data into a smaller dataset using Spark
    Write a query that produces ranked or sorted data using Spark

Data Analysis
-------------

Use Data Definition Language (DDL) to create tables in the Hive metastore for use by Hive and Impala.

    Read and/or create a table in the Hive metastore in a given schema
    Extract an Avro schema from a set of datafiles using avro-tools
    Create a table in the Hive metastore using the Avro file format and an external schema file
    Improve query performance by creating partitioned tables in the Hive metastore
    Evolve an Avro schema by changing JSON files


Reference documentation
=======================

### Hortonworks Documentation

HDP 2.4.2

[Introducing HDP to Java Developers](http://hortonworks.com/hadoop-tutorial/introducing-apache-hadoop-developers)
[Hortonworks Documentation](http://docs.hortonworks.com/index.html)
[HDP 2.4.2 Documentation](http://docs.hortonworks.com/HDPDocuments/HDP2/HDP-2.4.2/index.html)
[HDFS](https://docs.hortonworks.com/HDPDocuments/HDP2/HDP-2.4.2/bk_hdfs_admin_tools/content/index.html)
[Hive & Sqoop](https://docs.hortonworks.com/HDPDocuments/HDP2/HDP-2.4.2/bk_dataintegration/content/index.html)
[Spark](http://docs.hortonworks.com/HDPDocuments/HDP2/HDP-2.4.2/bk_spark-guide/content/ch_introduction-spark.html)

### Cloudera Documentation

    file:/usr/lib/spark/docs/_site/index.html
    http://www.cloudera.com/documentation/enterprise/5-3-x.html
    http://www.cloudera.com/documentation/enterprise/5-8-x.html

### MapR Documentation

### Microsoft Documentation

### Apache Documentation

