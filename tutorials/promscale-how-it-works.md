## 2. How Promscale Works [](how-to)

### 2.1 Promscale Architecture [](promscale-architecture)
Unlike some other long-term data stores for Prometheus, the basic Promscale architecture consists of only three components: Prometheus, Promscale, and TimescaleDB.

The diagram below explains the high level architecture of Promscale, including how it reads and writes to Prometheus, and how it can be queried by additional components, like PromLens, Grafana and any other SQL tool.

<img src="https://s3.amazonaws.com/assets.timescale.com/images/misc/promscale-architecture-final-2021.png" alt="Promscale Architecture Diagram" width="800"/>


#### Ingesting metrics

* Once installed alongside Prometheus, Promscale automatically generates an optimized schema which allows you to efficiently store and questy your metrics using SQL.
* Prometheus will write data to the Connector using the Prometheus`remote_write` interface.
* The Connector will then write data to TimescaleDB. 

#### Querying metrics

* PromQL queries can be directed to the Connector, or to the Prometheus instance, which will read data from the Connector using the `remote_read` interface. The Connector will, in turn, fetch data from TimescaleDB.
* SQL queries are handled by TimescaleDB directly.

As can be seen, this architecture has relatively few components, enabling simple operations.

#### Promscale PostgreSQL Extension

Promscale has a dependency on the [Promscale PostgreSQL extension][promscale-extension], which contains support functions to improve the performance of Promscale. While Promscale is able to run without the additional extension installed, adding this extension will get better performance from Promscale. 

#### Deploying Promscale

Promscale can be deployed in any environment running Prometheus, alongside any Prometheus instance. We provide Helm charts for easier deployments to Kubernetes environments (see Section 3 of this tutorial for more on installation and deployment).

### 2.2 Promscale Schema [](promscale-schema)

To achieve high ingestion, query performance and optimal storage we have designed a schema which takes care of writing the data in the most optimal format for storage and querying in TimescaleDB. Promscale translates data from the [Prometheus data model][prometheus-native-format] into a relational schema that is optimized for TimescaleDB, is stored efficiently, and is easy to query.

The basic schema uses a normalized design where the time-series data is stored in compressed hypertables. These tables have a foreign-key to series tables (stored as vanilla PostgreSQL tables), where each series consists of a unique set of labels.

In particular, this schema decouples individual metrics, allowing for the collection of metrics with vastly different cardinalities and retention periods. At the same time, Promscale exposes simple, user-friendly views so that you do not have to understand this optimized schema (see 2.3 for more on views).

> :TIP: Promscale automatically creates and manages database tables. So, while understanding the schema can be beneficial (and interesting), it is not required to use Promscale. Skip to Section 2.3 for information how to interact with Promscale using SQL views and to Section 4 to learn using hands on examples.


#### 2.2.1 Metrics Storage Schema
Each metric is stored in a separate hypertable. 

A hypertable is a TimescaleDB abstraction that represents a single logical SQL table that is automatically physically partitioned into chunks, which are physical tables that are stored in different files in the filesystem. Hypertables are partitioned into chunks by the value of certain columns. In this case, we will partition out tables by time (with a default chunk size of 8 hours).

**Compression**

The first chunk will be decompressed to serve as a high-speed query cache. Older chunks are stored as compressed chunks. We configure compression with the segment_by column set to the series_id and the orderby column set to time DESC. These settings control how data is split into blocks of compressed data. Each block can be accessed and decompressed independently.

The settings we have chosen mean that a block of compressed data is always associated with a single series_id and that the data is sorted by time before being split into blocks; thus each block is associated with a fairly narrow time range.  As a result, in compressed form, accesses by series_id and time range are optimized.

**Example**

The hypertables for each metric use the following schema (using `cpu_usage` as an example metric):

The `cpu_usage` table schema:
```sql
CREATE TABLE cpu_usage (
	time 		TIMESTAMPTZ,
	value 	DOUBLE PRECISION,
	series_id 	BIGINT,
)
CREATE INDEX ON cpu_usage (series_id, time) INCLUDE (value)
```

```sql
 Column   |           Type           | Modifiers
-----------+--------------------------+-----------
time      | TIMESTAMPTZ              |
value     | DOUBLE PRECISION         |
series_id | BIGINT                   |
```

In the above table, `series_id` is foreign key to the `series` table described below.

#### 2.2.2 Series Storage Schema

Conceptually, each row in the series table stores a set of key-value pairs. 

In Prometheus such a series is represented as a one-level JSON string. For example: `{ “key1”:”value1”, “key2”:”value2”}`. But the strings representing keys and values are often long and repeating. So, to save space, we store a series as an array of integer “foreign keys” to a normalized labels table. 

The definition of these two tables is shown below:

```sql
CREATE TABLE _prom_catalog.series (
    id serial,
    metric_id int,
    labels int[],
    UNIQUE(labels) INCLUDE (id)
);
CREATE INDEX series_labels_id ON _prom_catalog.series USING GIN (labels);

CREATE TABLE _prom_catalog.label (
    id serial,
    key TEXT,
    value text,
    PRIMARY KEY (id) INCLUDE (key, value), 
    UNIQUE (key, value) INCLUDE (id)
);
```

### 2.3 Promscale Views [](promscale-views)

The good news is that in order to use Promscale well, you do not need to understand the schema design. Users interact with Prometheus data in Promscale through views. These views are automatically created and are used to interact with metrics and labels.

Each metric and label has its own view. You can see a list of all metrics by querying the view named `metric`. Similarly, you can see a list of all labels by querying the view named `label`. These views are found in the `prom_info` schema.

#### Metrics Information View

Querying the `metric` view returns all metrics collected by Prometheus: 
```SQL
SELECT * 
FROM prom_info.metric;
```

Here is one row of a sample output for the query above:
```
id                | 16
metric_name       | process_cpu_seconds_total
table_name        | process_cpu_seconds_total
retention_period  | 90 days
chunk_interval    | 08:01:06.824386
label_keys        | {__name__,instance,job}
size              | 824 kB
compression_ratio | 71.60883280757097791800
total_chunks      | 11
compressed_chunks | 10
```

Each row in the `metric` view, contains fields with the metric `id`, as well as information about the metric, such as its name, table name, retention period, compression status, chunk interval etc.

Promscale maintains isolation between metrics. This allows users to set retention periods, downsampling and compression settings on a per metric basis, giving users more control over their metrics data. 


#### Labels Information View

Querying the `label` view returns all labels associated with metrics collected by Prometheus: 
```SQL
SELECT * 
FROM prom_info.label;
```

Here is one row of a sample output for the query above:
```
key               | collector
value_column_name | collector
id_column_name    | collector_id
values            | {arp,bcache,bonding,btrfs,conntrack,cpu,cpufreq,diskstats,edac,entropy,filefd,filesystem,hwmon,infiniband,ipvs,loadavg,mdadm,meminfo,netclass,netdev,netstat,nfs,nfsd,powersupplyclass,pressure,rapl,schedstat,sockstat,softnet,stat,textfile,thermal_zone,time,timex,udp_queues,uname,vmstat,xfs,zfs}
num_values        | 39
```


Each label row contains information about a particular label, such as the label key, the label's value column name, the label's id column name, the list of all values taken by the label and the total number of values for that label.

For examples of querying a specific metric view, see [querying using SQL][querying-using-sql] of this tutorial.

Next Step: [Installing Promscale][installing-promscale]. 

[prometheus-native-format]: https://prometheus.io/docs/instrumenting/exposition_formats/
[promscale-extension]: https://github.com/timescale/promscale_extension#promscale-extension
[querying-using-sql]: /tutorials/getting-started-with-promscale/promscale-run-queries#sql-queries
[installing-promscale]: /tutorials/getting-started-with-promscale/promscale-install