YARN
====

Introduction
------------
Hadoop V2 scheduler

ResourceManager -> NodeManager
AppMaster
Containers

CLI
---

    yarn application -list
    yarn logs -applicationId <applicationId>
    yarn application -kill <applicationId>

YARN RM UI
----------

    http://yarn.hdp.cesga.es:8088

Capacity Scheduler
------------------
List of available queues:
- default
- interactive
- hive

References
----------
http://hortonworks.com/blog/apache-hadoop-yarn-resourcemanager/
