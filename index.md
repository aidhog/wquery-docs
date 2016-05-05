---
layout: default
---

This pages aims to make repeatable the experiments described in the paper
*Querying Wikidata: Comparing SPARQL, Relational and Graph Databases* (by
[Daniel Hernández](http://users.dcc.uchile.cl/~dhernand/),
[Aidan Hogan](http://users.dcc.uchile.cl/~ahogan/),
[Cristian Riveros](http://web.ing.puc.cl/~criveros/),
Carlos Rojas and Enzo Zerega).

# Additional resources

Resources that complement the submited paper are:

* This document (<https://dx.doi.org/10.6084/m9.figshare.3219217.v3>).
* The code and results (<https://bitbucket.org/danielhz/wikidata-experiments>).
* The data (currently available [here](http://datadumps.daniel.degu.cl/wikidata-20160104.json.gz)).

# The data

All experiments are done using the dump of Wikidata published on
January 04, 2017. The original dump was downloaded from the
[dumps folder](https://dumps.wikimedia.org/other/wikidata/) published by
the Wikimedia Foundation. However, the contains in this folder are frequently
updated and old dumps are discarded. Thus, to make this experiment repeatable
we publish temporally the data
[here](http://datadumps.daniel.degu.cl/wikidata-20160104.json.gz).
Currently we are loading the data into https://figshare.com/.

# Code and results

All the code used to run the experiments and the results is tracked in the
git repository
[wikidata-experiments](https://bitbucket.org/danielhz/wikidata-experiments).
This code allows to:

* Translate the data to different models used in the experiments.
* Load the data into the respective databases.
* Clean and fix the data.
* Generate parameters for queries that are generated randomly.
* Run the experiments.

# Licence

All our code is published under the
[Creative Commons CC0 Licence](https://creativecommons.org/publicdomain/zero/1.0/), that is, the same licence
used for the data. All of the engines used in this experiments are
distributed under open licences. PostgreSQL uses the
[PostgreSQL Licence](https://opensource.org/licenses/postgresql),
Virtuoso Opensource uses the
[GPLv2 Licence](https://github.com/openlink/virtuoso-opensource/blob/develop/7/LICENSE),
Blazegraph uses the
[GPLv2 Licence](https://www.blazegraph.com/services/blazegraph-licensing/)
licence and Neo4j Community Edition uses the
[GPLv3 Licence](http://neo4j.com/licensing/).

# Experimental settings

**Machine:** All experiments were run on a single machine with 2× Intel Xeon
Six Core E5-2609 V3 CPUs, 32GB of RAM, and 2× 1TB Seagate 7200 RPM
32MB Cache SATA hard-disks in a RAID-1 configuration.

**System:** All experiments where run on a Debian 7 system. The partition
mounted in `/` (which has only 55GB) is not enough for the data used. Thus,
when necessary we use the partition mount in `/home` (which has 1.7TB) for
the data.

**User:** All experiments will be run for a user identified as `$USER` which
home is `/home/$USER`. Also, `$USER` is in the `sudo` group.

**Code:** Code of experiments that use the RDF data model is tracked in
[a git repository](https://bitbucket.org/danielhz/wikidata-experiments).
`$USER` runs the following command inside the `/home/$USER` folder
to get this code.

```
$ git clone https://bitbucket.org/danielhz/wikidata-experiments.git
```      

In that follows we call the folder `/home/$USER/wikidata-experiments` as
`$WD_HOME`.

**Default graph:** In Virtuoso and Blazegraph the default dataset is always,
assumed as the union of all named graphs. Thus, no specific configuration is
needed to get this behavior with the named graphs schema.

**RDF schemas:** We use four schemas to model Wikidata using RDF: n-ary
relations, named graphs, singleton properties and standard reification.
We call this schemas as `naryrel`, `ngraphs`, `sgprop` and `stdreif`,
respectively. Also, we use the environment name `$SCHEMA` for the current
schema in RDF experiments.

## Virtuoso

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

We use Blazegraph 2.1.0 Community Edition with Java 7. We use the Java
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
And the server.properties define the parameters of the execution and properties
of the storage. Both configuration files can be found in the code repository
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
maintenance_work_mem          = 1920MB
shared_buffers                = 7680MB
effective_cache_size          = 22GB
default_transaction_isolation = 'read uncommitted'
```

We use [pgtune](https://github.com/gregs1104/pgtune#readme), an script to
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

# Queries

## Quin queries
