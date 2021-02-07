## 4. Running queries using Promscale

Promscale offers the combined power of PromQL and SQL, enabling you to ask any 
question, create any dashboard, and achieve greater visibility into the systems 
you monitor.

In the configuration used in Part 3 above, Prometheus will scrape the Node Exporter every 10s and metrics will be stored in both Prometheus and TimescaleDB, via Promscale. 

This section will illustrate how to run simple and complex SQL queries against Promscale, as well as queries in PromQL.

### 4.1 SQL Queries in Promscale [](sql-queries)

You can query Promscale in SQL from your favorite favorite SQL tool or using psql: 

```bash
docker exec -it timescaledb psql postgres postgres
```

The above command first enters our timescaledb docker container (from Step 3.1 above) and creates an interactive terminal to it. Then it opens up`psql`, a terminal-based front end to PostgreSQL (More information on psql -- [psql-docs][].

Once inside, we can now run SQL queries and explore the metrics collected by Prometheus and Promscale

#### 4.1.1 Querying a metric:

Queries on metrics are performed by querying the view named after the metric you're interested in.

In the example below, we will query a metric named `go_dc_duration` for its samples in the past 5 minutes. This metric is a measurement for how long garbage collection is taking in Golang applications:

``` sql
SELECT * from go_gc_duration_seconds
WHERE time > now() - INTERVAL '5 minutes';
```

Here is a sample output for the query above (your output will differ):

```sql
            time            |    value    | series_id |      labels       | instance_id | job_id | quantile_id 
----------------------------+-------------+-----------+-------------------+-------------+--------+-------------
 2021-01-27 18:43:42.389+00 |           0 |       495 | {208,43,51,212}   |          43 |     51 |         212
 2021-01-27 18:43:42.389+00 |           0 |       497 | {208,43,51,213}   |          43 |     51 |         213
 2021-01-27 18:43:42.389+00 |           0 |       498 | {208,43,51,214}   |          43 |     51 |         214
 2021-01-27 18:43:42.389+00 |           0 |       499 | {208,43,51,215}   |          43 |     51 |         215
 2021-01-27 18:43:42.389+00 |           0 |       500 | {208,43,51,216}   |          43 |     51 |         216
```

Each row returned contains a number of different fields:
* The most important fields are `time`, `value` and `labels`.
* Each row has a `series_id` field, which uniquely identifies its measurements label set. This enables efficient aggregation by series. 
* Each row has a field named `labels`. This field contains an array of foreign key to label key-value pairs making up the label set.
* While the `labels` array is the entire label set, there are also seperate fields for each label key in the label set, for easy access. These fields end with the suffix `_id` .

#### 4.1.2 Querying values for label keys

As explained in the last bullet point above, each label key is expanded out into its own column storing foreign key identifiers to their value, which allows us to JOIN, aggregate and filter by label keys and values. 

To get back the text represented by a label id, use the `val(field_id)` function. This opens up nifty possibilities such as aggregating across all series with a particular label key.

For example, take this example, where we find the median value for the `go_gc_duration_seconds` metric, grouped by the job associated with it:

``` sql
SELECT 
    val(job_id) as job, 
    percentile_cont(0.5) within group (order by value) AS median
FROM 
    go_gc_duration_seconds 
WHERE 
    time > now() - INTERVAL '5 minutes'
GROUP BY job_id;
```

Sample Output:
```sql
      job      |  median
---------------+-----------
 prometheus    |  6.01e-05
 node-exporter | 0.0002631
```

#### 4.1.3 Querying label sets for a metric

As explained in Section 2, the `labels` field in any metric row represents the full set of labels associated with the measurement and is represented as an array of identifiers. 

To return the entire labelset in JSON, we can apply the `jsonb()` function, as in the example below:

``` sql
SELECT 
    time, value, jsonb(labels) as labels
FROM 
    go_gc_duration_seconds
WHERE 
    time > now() - INTERVAL '5 minutes';
```

Sample Output:

```sql
            time            |    value    |                                                        labels                                                        
----------------------------+-------------+----------------------------------------------------------------------------------------------------------------------
 2021-01-27 18:43:48.236+00 | 0.000275625 | {"job": "prometheus", "__name__": "go_gc_duration_seconds", "instance": "localhost:9090", "quantile": "0.5"}
 2021-01-27 18:43:48.236+00 | 0.000165632 | {"job": "prometheus", "__name__": "go_gc_duration_seconds", "instance": "localhost:9090", "quantile": "0.25"}
 2021-01-27 18:43:48.236+00 | 0.000320684 | {"job": "prometheus", "__name__": "go_gc_duration_seconds", "instance": "localhost:9090", "quantile": "0.75"}
 2021-01-27 18:43:52.389+00 |  1.9633e-05 | {"job": "node-exporter", "__name__": "go_gc_duration_seconds", "instance": "node_exporter:9100", "quantile": "0"}
 2021-01-27 18:43:52.389+00 |  1.9633e-05 | {"job": "node-exporter", "__name__": "go_gc_duration_seconds", "instance": "node_exporter:9100", "quantile": "1"}
 2021-01-27 18:43:52.389+00 |  1.9633e-05 | {"job": "node-exporter", "__name__": "go_gc_duration_seconds", "instance": "node_exporter:9100", "quantile": "0.5"}
```
This query returns the label set for the metric `go_gc_duration` in JSON format. It can then be read or further interacted with. 

#### 4.1.4 Advanced Query: Percentiles aggregated over time and series

The query below calculates the 99th percentile over both time and series (`app_id`) for the metric named `go_gc_duration_seconds`. This metric is a measurement for how long garbage collection is taking in Go applications:

``` sql
SELECT 
    val(instance_id) as app,
    percentile_cont(0.99) within group(order by value) p99
FROM 
    go_gc_duration_seconds
WHERE 
    value != 'NaN' AND val(quantile_id) = '1' AND instance_id > 0
GROUP BY instance_id 
ORDER BY p99 desc;
```

Sample Output:
```sql
        app         |     p99     
--------------------+-------------
 node_exporter:9100 | 0.002790063
 localhost:9090     |  0.00097977
```

The query above is uniquely enabled by Promscale, as it aggregates over both time and series and returns an accurate calculation of the percentile. Using only a PromQL query, it is not possible to accurately calculate percentiles when aggregating over both time and series.

The query above is just one example of the kind of analytics Promscale can help you perform on your Prometheus monitoring data.

#### 4.1.5 Filtering by labels

To simplify filtering by labels, we created operators corresponding to the selectors in PromQL. 

Those operators are used in a `WHERE` clause of the form `labels ? (<label_key> <operator> <pattern>)`

The four operators are:
* == matches tag values that are equal to the pattern
* !== matches tag value that are not equal to the pattern
* ==~ matches tag values that match the pattern regex
* !=~ matches tag values that are not equal to the pattern regex

These four matchers correspond to each of the four selectors in PromQL, though they have slightly different spellings to avoid clashing with other PostgreSQL operators. They can be combined together using any boolean logic with any arbitrary WHERE clauses.

For example, if you want only those metrics from the job with name `node-exporter`, you can filter by labels to include only those samples:

``` sql
SELECT 
    time, value, jsonb(labels) as labels
FROM 
    go_gc_duration_seconds
WHERE 
    labels ? ('job' == 'node-exporter')
    AND time > now() - INTERVAL '5 minutes';
```

Sample output:
```sql
 time                       |   value   |              labels
----------------------------+-----------+----------------------------------------------------------------------------------------------------------
 2021-01-28 02:01:18.066+00 |  3.05e-05 | {"job": "node-exporter", "__name__": "go_gc_duration_seconds", "instance": "node_exporter:9100", "quantile": "0"}
 2021-01-28 02:01:28.066+00 |  3.05e-05 | {"job": "node-exporter", "__name__": "go_gc_duration_seconds", "instance": "node_exporter:9100", "quantile": "0"}
 2021-01-28 02:01:38.032+00 |  3.05e-05 | {"job": "node-exporter", "__name__": "go_gc_duration_seconds", "instance": "node_exporter:9100", "quantile": "0"}
```

#### 4.1.6 Querying Number of Datapoints in a Series

As shown in 4.1.1 above, each in a row metric's view  has a series_id uniquely identifying the measurement’s label set. This enables efficient aggregation by series. 

You can easily retrieve the labels array from a series_id using the labels(series_id) function. As in this query that shows how many data points we have in each series:

``` sql 
SELECT jsonb(labels(series_id)) as labels, count(*)
FROM go_gc_duration_seconds
GROUP BY series_id;
```

Sample output:
```sql
                                                       labels                                                        | count
----------------------------------------------------------------------------------------------------------------------+-------
 {"job": "node-exporter", "__name__": "go_gc_duration_seconds", "instance": "node_exporter:9100", "quantile": "0.75"} |   631
 {"job": "prometheus", "__name__": "go_gc_duration_seconds", "instance": "localhost:9090", "quantile": "0.75"}        |   631
 {"job": "node-exporter", "__name__": "go_gc_duration_seconds", "instance": "node_exporter:9100", "quantile": "1"}    |   631
 {"job": "node-exporter", "__name__": "go_gc_duration_seconds", "instance": "node_exporter:9100", "quantile": "0.5"}  |   631
 {"job": "prometheus", "__name__": "go_gc_duration_seconds", "instance": "localhost:9090", "quantile": "0.5"}         |   631
 {"job": "node-exporter", "__name__": "go_gc_duration_seconds", "instance": "node_exporter:9100", "quantile": "0"}    |   631
 {"job": "prometheus", "__name__": "go_gc_duration_seconds", "instance": "localhost:9090", "quantile": "1"}           |   631
 {"job": "node-exporter", "__name__": "go_gc_duration_seconds", "instance": "node_exporter:9100", "quantile": "0.25"} |   631
 {"job": "prometheus", "__name__": "go_gc_duration_seconds", "instance": "localhost:9090", "quantile": "0.25"}        |   631
 {"job": "prometheus", "__name__": "go_gc_duration_seconds", "instance": "localhost:9090", "quantile": "0"}           |   631
```


#### BONUS: Other complex queries

While the above examples are for metrics from Prometheus and node_exporter, a more complex example from [Dan Luu’s post: "A simple way to get more value from metrics"][Dan-Luu's-post-on-SQL-query], shows how you can discover Kubernetes containers that are over-provisioned by finding those containers whose 99th percentile memory utilization is low:

``` sql
WITH memory_allowed as (
  SELECT 
    labels(series_id) as labels, 
    value, 
    min(time) start_time, 
    max(time) as end_time 
  FROM container_spec_memory_limit_bytes total
  WHERE value != 0 and value != 'NaN'
  GROUP BY series_id, value
)
SELECT 
  val(memory_used.container_id) container, 
  percentile_cont(0.99) 
    within group(order by memory_used.value/memory_allowed.value) 
    AS percent_used_p99, 
  max(memory_allowed.value) max_memory_allowed
FROM container_memory_working_set_bytes AS memory_used 
INNER JOIN memory_allowed
      ON (memory_used.time >= memory_allowed.start_time AND 
          memory_used.time <= memory_allowed.end_time AND
          eq(memory_used.labels,memory_allowed.labels)) 
WHERE memory_used.value != 'NaN'   
GROUP BY container 
ORDER BY percent_used_p99 ASC 
LIMIT 100;
```

A sample output for the above query is as follows:
```sql
 container                      |   percent_used_p99    |   total
--------------------------------+-----------------------+---------------
cluster-overprovisioner-system  | 6.961822509765625e-05 |  4294967296
sealed-secrets-controller       |   0.00790748596191406 |  1073741824
dumpster                        |    0.0135690307617187 |   268435456
```
While the above example requires the installation of `cAdvisor`, it's just an example of the sorts of sophisticated analysis enabled by Promscale's support to query your data in SQL.

### 4.2 PromQL Queries in Promscale [](promql-querying-using-promscale)
Promscale can also be used as a Prometheus data source for tools like [Grafana][grafana-homepage] and [PromLens][promlens-homepage]. 

We'll deomstrate this by connecting Promscale as a Prometheus data source in [Grafana][grafana-homepage], a popular open-source visualization tool.

First, let's install Grafana via [their official Docker image][grafana-docker]:

```bash
docker run -d \
  -p 3000:3000 \
  --network promscale-timescaledb \
  --name=grafana \
  -e "GF_INSTALL_PLUGINS=grafana-clock-panel,grafana-simple-json-datasource" \
  grafana/grafana
```

Next, navigate to `localhost:3000` in your browser and enter `admin` for both the Grafana username and password, and set a new password. Then navigate to `Configuration > Data Sources > Add data source > Prometheus`.

Finally, configure the data source settings, setting the `Name` as `Promscale` and setting the `URL` as`http://<PROMSCALE-IP-ADDR>:9201`, taking care to replace `<PROMSCALE-IP-ADDR>` with the IP address of your Promscale instance. All other settings can be kept as default unless desired.

To find the Promscale IP address, run the command `docker inspect promscale` (where `promscale` is the name of our container) and find the IP address under `NetworkSettings > Networks > IPAddress`.

Alternatively, we can set the `URL` as `http://promscale:9201`, where `promscale` is the name of our container. This method works as we've created all our containers in the same docker network (using the flag `-- network promscale-timescaledb` during our installs).

After configuring the Promscale as a datasource in Grafana, all that's left is to  create a sample panel using `Promscale` as the datasource.  The query powering the panel will be written in PromQL. The sample query below shows the average rate of change in the past 5 minutes for the metric `go_memstats_alloc_bytes`, which measures the Go's memory allocation on the heap from the kernel:
```bash
rate(go_memstats_alloc_bytes{instance="localhost:9090"}[5m])
```

#### Sample output:

![](https://i.imgur.com/vZPGsDu.png)

[psql-docs]: https://www.postgresql.org/docs/13/app-psql.html
[Dan-Luu's-post-on-SQL-query]: https://danluu.com/metrics-analytics/
[grafana-homepage]:https://grafana.com
[promlens-homepage]: https://promlens.com
[grafana-docker]: https://grafana.com/docs/grafana/latest/installation/docker/#install-official-and-community-grafana-plugins