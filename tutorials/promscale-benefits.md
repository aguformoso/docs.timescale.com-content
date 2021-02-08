## Why use Promscale & TimescaleDB to store Prometheus metrics [](why-promscale)

This section details five reasons to use Timescale and Promscale to store Prometheus metrics:

1. [Long term data storage](#lts-data-storage)
2. [Operational ease](#operational-ease)
3. [Queries in SQL and PromQL](#querying-using-sql-promql)
4. [Per Metric Retention](#metric-retention)
5. [Ability to push data from custom applications](#push-custom-time-series-data)

### 1.1 Long term data storage [](lts-data-storage)

In order to keep Prometheus simple and easy to operate, its creators intentionally left out many scaling features one would normally expect. Data in Prometheus is stored locally within the instance and is not replicated. Having both compute and data storage on one node may make it easier to operate, but also makes it harder to scale and ensure high availability.

As a result, Prometheus is not designed to be a long-term metrics store. From the [documentation][prometheus-storage-docs]:

>Prometheus is not arbitrarily scalable or durable in the face of disk or node outages and should thus be treated as more of an ephemeral sliding window of recent data.

On the other hand, TimescaleDB can easily handle petabytes of data, and supports high availability and replication, making it a good fit for long term data storage. In addition, it provides advanced capabilities and features, such as full SQL, JOINs and data retention and aggregation policies, all of which are not available in Prometheus.

Using Promscale as a long term store for Prometheus metrics works as follows: 
* All metrics that are scraped from targets are first written to the Prometheus local storage. 
* Metrics are then written to Promscale via the Prometheus `remote-write` endpoint. 
* This means that all of your metrics are ingested in parallel to TimescaleDB using Promscale, making any Prometheus disk failure less painful.

TimescaleDB can also store other types of data (metrics from other systems, time-series data, relational data, metadata), allowing you to consolidate monitoring data from different sources into one database and simplify your stack. You can also join different types of data and add context to your monitoring data for richer analysis, using SQL `JOIN`s.

Moreover, the recent release of [TimescaleDB 2.0][multinode-blog] introduces multi-node functionality, making it easier to scale horizontally and store data at petabyte scale.

### 1.2 Operational ease [](operational-ease)


Promscale is an exception. Promscale operates as a single stateless service. This means that once you configure the `remote-write` & `remote-read` endpoints in your prometheus configuration file then all samples  forwarded to Promscale. Promscale then handles the ingestion of samples into TimescaleDB. Promscale exposes all Prometheus compatible APIs, allowing you to query Promscale using PromQL queries. 

Moreover, the only way to scale Prometheus is by [federation][prometheus-federation]. However, there are cases where federation is not a good fit: for example, when copying large amounts of data from multiple Prometheus instances to be handled by a single machine. This can result in poor performance, decreased reliability (an additional point of failure), and loss of some data. These are all problems you can avoid by using an operationally simple and mature platform like Promscale combined with TimescaleDB.

Furthermore, with Promscale, it's simple to get a global view of all metrics data, using both PromQL and SQL queries.



### 1.3 Queries in SQL and PromQL [](querying-using-sql-promql)

By allowing a user to use SQL, in addition to PromQL, Promscale empowers the user to ask complex analytical queries from their metrics data, and thus extract more meaningful insights.

PromQL is the Prometheus native query language. It’s a powerful and expressive query language that allows you to easily slice and dice metrics data and use a variety of monitoring specific [functions][promql-functions].

However, there may be times where you need greater query flexibility and expressiveness than what PromQL provides. For example, trying to cross-correlate metrics with events and incidents that occurred, running more granular queries for active troubleshooting, or applying machine learning or other forms of deeper analysis on metrics.

TimescaleDB's full SQL support is of great help in these cases. It enables you to apply the full breadth of SQL to your Prometheus data, joining your metrics data with any other data you might have, and run more powerful queries.

We detail examples of such SQL queries in Part 4 of this tutorial. 

### 1.4 Per Metric Retention [](metric-retention)

Promscale maintains isolation between metrics. This allows you to set retention periods, downsampling and compression settings on a per metric basis, giving you more control over your metrics data. 

Per metric retention policies, downsampling, aggregation and compression helps you store only series you care about for the long term. This helps you better trade off the value brought by keeping metrics for the long term with the storage costs involved, allowing you to keep metrics that matter for as long as you need and discarding the rest to save on costs.

This isolation extends to query performance, wherein queries for one metric are not affected by the cardinality or query and write load of other metrics. This provides better performance on smaller metrics and in general provides a level of safety within your metric storage system.

### 1.5 Ability to push data from custom applications [](push-custom-time-series-data)

The Promscale write endpoints also accept data from your custom applications (i.e data outside of Prometheus) in JSON format. 

All you have to do is parse your existing time series data to object format below:

```bash
{
    "labels":{"__name__": "cpu_usage", "namespace":"dev", "node": "brain"},
    "samples":[
        [1577836800000, 100],
        [1577836801000, 99],
        [1577836802000, 98],
        ...
    ]
}
```

Then perform a write request to Promscale:

```bash
curl --header "Content-Type: application/json" \
--request POST \
--data '{"labels":{"__name__":"foo"},"samples":[[1577836800000, 100]]}' \
"http://localhost:9201/write"
```

For more details on writing custom time-series data to Promscale can be found in this document: [Writing to Promscale][writing-to-promscale]

Next Step: [How Promscale Works][how-promscale-works].

[prometheus-storage-docs]: https://prometheus.io/docs/prometheus/latest/storage/
[prometheus-federation]: https://prometheus.io/docs/prometheus/latest/federation/
[promql-functions]: https://prometheus.io/docs/prometheus/latest/querying/functions/
[multinode-blog]: https://blog.timescale.com/blog/timescaledb-2-0-a-multi-node-petabyte-scale-completely-free-relational-database-for-time-series/
[writing-to-promscale]: https://github.com/timescale/promscale/blob/master/docs/writing_to_promscale.md
[how-promscale-works]: /tutorials/getting-started-with-promscale/promscale-how-it-works