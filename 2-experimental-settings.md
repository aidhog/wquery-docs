---
layout: page
title: Experimental settings
top: true
---

**Machine:** All experiments were run on a single machine with 2× Intel Xeon
Six Core E5-2609 V3 CPUs, 32GB of RAM, and 2× 1TB Seagate 7200 RPM
32MB Cache SATA hard-disks in a RAID-1 configuration.

**System:** All experiments where run on a Debian 7 system using Ruby 2.3,
Python 2 and Bash scripts. The partition mounted in `/` (which has only 55GB)
is not enough for the data used. Thus, when necessary we use the partition
mounted in `/home` (which has 1.7TB) for the data.

**Default graph:** In Virtuoso and Blazegraph the default dataset is always,
assumed as the union of all named graphs. Thus, no specific configuration is
needed to get this behavior with the named graphs schema.

**RDF schemas:** We use four schemas to model Wikidata using RDF: n-ary
relations, named graphs, singleton properties and standard reification.
We call this schemas as `naryrel`, `ngraphs`, `sgprop` and `stdreif`,
respectively. Also, we use the environment name `$SCHEMA` for the current
schema in RDF experiments.

## Virtuoso

We used Virtuoso Open Source Edition (7.2.3-dev.3215-pthreads), compiled from
[the source](https://github.com/openlink/virtuoso-opensource/).

For each `$SCHEMA` there is a directory
`$WD_HOME/dbfiles/virtuoso/db-$SCHEMA-1`. Initially, this folder contains
only a file `virtuoso.ini` inside.

By default, Virtuoso stores the data in a folder in the `/` partition
(that has not enough space). Inside this folder we create a symbolic link to
the database folder of each `$SCHEMA`.

```
$ cd /usr/local/virtuoso-opensource/var/lib/virtuoso/
$ ln -s $WD_HOME/dbfiles/virtuoso/db-$SCHEMA-1 db-$SCHEMA-1
```

The following variables are set in `virtuoso.ini` file of each `$SCHEMA`.

```
NumberOfBuffers            = 2720000
MaxDirtyBuffers            = 2000000
MaxQueryCostEstimationTime = 0
MaxQueryExecutionTime      = 60
```

The values for the properties `NumberOfBuffers` and `MaxDirtyBuffers` are the
recommended in the configuration file for a machine with 32GB of memory
(as our machine has).

The property `MaxQueryCostEstimationTime` indicates the maximum estimated time
for a query. If the engine estimates that a query will long more that this
value, then it will not evaluate it. We set this property to 0, that means that
no estimation limits are considered applied, i.e., all queries are evaluated.

Finally, the `MaxQueryExecutionTime` is the timeout for query execution.
Queries that exceed this timeout are aborted in runtime.

## Blazegraph

We used Blazegraph 2.1.0 Community Edition with Java 7. We use the Java
implementation distributed by ORACLE
(Java(TM) SE Runtime Environment build 1.7.0_80-b15).

Some parameters are added into the command line to improve the resource
usage of the process.
We set the JVM heap to 6GB and we use the G1 garbage collector
off the Hostpot JVM (The JVM provides several garbage collectors).
The
[documentation](https://wiki.blazegraph.com/wiki/index.php/PerformanceOptimization)
recommends these parameters.

We use the `exec` primitive in a Ruby script to start Blazegraph
with the following parameters:

```
exec(['java', 'blazegraph'],
  '-Xmx6g',
  '-XX:+UseG1GC',
  '-Djetty.overrideWebXml=override.xml',
  '-Dbigdata.propertyFile=server.properties',
  '-jar',
  'blazegraph.jar')
```

The `override.xml` file define a timeout of 60 seconds for query execution.
And the `server.properties` file defines the parameters of the execution and
properties of the storage. Both configuration files can be found in the
[code repository](https://bitbucket.org/danielhz/wikidata-experiments/src/0389b993f34afaab2fff36411e76ce33ef86465b/dbfiles/blazegraph/?at=master)
of our experiments.

Blazegraph provides several data storages. We use triple stores for
experiments with n-ary relations, singleton properties and standard
reification. For named graphs we use quad stores.

## Neo4j

We used Neo4J-community-2.3.1 with Java 7, using the distribution
available in the system
(OpenJDK Runtime Environment (IcedTea 2.6.6)).

Two databases were created, one with all the data of the JSON dump and the
other without the Labels, Aliases and Descriptions of Wikidata entities, in
order to make the nodes lighter.

We created indexes on labels `:Item(id)`, `:Property(id)` and `:Entity(id)`.
To map from entity ids (e.g., Q42) and property ids (e.g., P1432) to their
respective nodes.

The following variables are set in `conf/neo4j-wrapper.conf` a 20GB heap:

```
wrapper.java.initmemory = 20480
wrapper.java.maxmemory  = 20480
```

Also we set the open file descriptor limit to 40.000 as is recommended in
the [Neo4j documentation](http://neo4j.com/docs/stable/performance-guide.html#_setting_the_number_of_open_files).

## PostgreSQL

We used PostgreSQL 9.1.20 with the folowing variables set in the
`postgres.conf`:

```
default_statistics_target     = 100
maintenance_work_mem          = 1920MB
shared_buffers                = 7680MB
wal_buffers                   = 16MB
effective_cache_size          = 22GB
work_mem                      = 160MB
default_transaction_isolation = 'read uncommitted'
statement_timeout             = 60010
```

We used [pgtune](https://github.com/gregs1104/pgtune#readme), an script to
automatically generate a configuration for PostgresSQL in our server.
This script is based on the recommendations in the
[PostgresSQL Wiki](https://wiki.postgresql.org/wiki/Tuning_Your_PostgreSQL_Server)
and set the values of properties `maintenance_work_mem`,
`shared_buffers` and `effective_cache_size`. Also, we set
the lowest of isolation where transactions are isolated only enough to ensure
that physically corrupt data is not read.

We set a B-tree index for primary keys. By default PostgreSQL do not set
any index for foreign keys because these are used for consistency and not
for performance. We create a secondary index for every foreign keys in the
model and for each attribute that stores either entities, properties or data
values (e.g. dates) from Wikidata.