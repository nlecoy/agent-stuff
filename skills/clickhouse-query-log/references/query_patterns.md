# Query patterns

## Contents
- Assumed columns
- Safe defaults
- Reusable filter snippets (project key, database scope, table scope)
- KPI summary for one window
- Compare two windows
- Slow queries by normalized query hash
- Abusive users
- Request spikes by user over time
- Investigation for one normalized query hash
- Top users for one normalized query hash
- Recent examples for one normalized query hash
- Notes on cluster safety

Use these SQL patterns as starting points. Adapt column names only when needed. Always preserve bounded time filters and low-cardinality outputs.

**Always replace `system.query_log` with `clusterAllReplicas('{cluster_name}', system.query_log)`** in every query. Substitute `{cluster_name}` with the cluster name confirmed by the user. This collects data from all shards and replicas, not just the node receiving the connection.

## Assumed columns

These templates assume common `system.query_log` columns such as:

- `event_time`
- `type`
- `query_duration_ms`
- `read_rows`
- `read_bytes`
- `result_rows`
- `memory_usage`
- `exception_code`
- `user`
- `client_name`
- `query_kind`
- `normalized_query_hash`
- `query`

If a column is unavailable, simplify the query and note the limitation.

## Safe defaults

- Default window: last 24 hours
- Default population for performance KPIs: `type = 'QueryFinish'`
- Default ranked result size: 20
- Default project filter: exact match when structured; otherwise conservative extraction from `query`

## Reusable filter snippets

### Structured project key

```sql
AND project_key = 'project_XYZ'
```

### Project key extracted from query text

```sql
AND positionCaseInsensitive(query, 'project_key') > 0
AND match(query, '(?i)project_key\\s*=\\s*''project_XYZ''')
```

Use text extraction only when there is no structured column and note that it is a best-effort filter.

### Database scope (structured array column)

```sql
AND has(databases, 'my_database')
```

Use when the user wants activity scoped to a specific database. `databases` is an `Array(String)` column in `system.query_log`. Note in the output that some query types (mutations, system queries) may not populate this column.

### Table scope (structured array column)

```sql
AND has(tables, 'my_database.my_table')
```

Table names are fully qualified (`database.table`). Use when the user targets a specific table. Apply the same caveat as for `databases`.

## KPI summary for one window

```sql
SELECT
    count() AS query_count,
    uniqExact(user) AS distinct_users,
    countIf(exception_code != 0) AS error_count,
    round(100.0 * countIf(exception_code != 0) / count(), 2) AS error_rate_pct,
    quantile(0.50)(query_duration_ms) AS p50_ms,
    quantile(0.95)(query_duration_ms) AS p95_ms,
    quantile(0.99)(query_duration_ms) AS p99_ms,
    round(avg(query_duration_ms), 2) AS avg_ms,
    sum(read_rows) AS total_read_rows,
    sum(read_bytes) AS total_read_bytes,
    sum(result_rows) AS total_result_rows,
    quantile(0.95)(memory_usage) AS p95_memory,
    max(memory_usage) AS max_memory
FROM clusterAllReplicas('{cluster_name}', system.query_log)
PREWHERE event_time >= toDateTime('{start}')
    AND event_time < toDateTime('{end}')
WHERE type = 'QueryFinish'
  {project_filter};
```

## Compare two windows

```sql
WITH
window_a AS (
    SELECT
        count() AS query_count,
        countIf(exception_code != 0) AS error_count,
        quantile(0.95)(query_duration_ms) AS p95_ms,
        sum(read_bytes) AS total_read_bytes,
        uniqExact(user) AS distinct_users
    FROM clusterAllReplicas('{cluster_name}', system.query_log)
    PREWHERE event_time >= toDateTime('{a_start}')
        AND event_time < toDateTime('{a_end}')
    WHERE type = 'QueryFinish'
      {project_filter}
),
window_b AS (
    SELECT
        count() AS query_count,
        countIf(exception_code != 0) AS error_count,
        quantile(0.95)(query_duration_ms) AS p95_ms,
        sum(read_bytes) AS total_read_bytes,
        uniqExact(user) AS distinct_users
    FROM clusterAllReplicas('{cluster_name}', system.query_log)
    PREWHERE event_time >= toDateTime('{b_start}')
        AND event_time < toDateTime('{b_end}')
    WHERE type = 'QueryFinish'
      {project_filter}
)
SELECT
    a.query_count AS query_count_a,
    b.query_count AS query_count_b,
    b.query_count - a.query_count AS query_count_delta,
    round(100.0 * (b.query_count - a.query_count) / nullIf(a.query_count, 0), 2) AS query_count_delta_pct,
    a.error_count AS error_count_a,
    b.error_count AS error_count_b,
    a.p95_ms AS p95_ms_a,
    b.p95_ms AS p95_ms_b,
    b.p95_ms - a.p95_ms AS p95_ms_delta,
    a.total_read_bytes AS total_read_bytes_a,
    b.total_read_bytes AS total_read_bytes_b,
    b.total_read_bytes - a.total_read_bytes AS total_read_bytes_delta,
    a.distinct_users AS distinct_users_a,
    b.distinct_users AS distinct_users_b
FROM window_a a
CROSS JOIN window_b b;
```

## Slow queries by normalized query hash

```sql
SELECT
    normalized_query_hash,
    count() AS query_count,
    round(avg(query_duration_ms), 2) AS avg_ms,
    quantile(0.50)(query_duration_ms) AS p50_ms,
    quantile(0.95)(query_duration_ms) AS p95_ms,
    quantile(0.99)(query_duration_ms) AS p99_ms,
    max(query_duration_ms) AS max_ms,
    round(100.0 * countIf(exception_code != 0) / count(), 2) AS error_rate_pct,
    sum(read_bytes) AS total_read_bytes,
    sum(read_rows) AS total_read_rows,
    uniqExact(user) AS distinct_users,
    any(query_kind) AS sample_query_kind
FROM clusterAllReplicas('{cluster_name}', system.query_log)
PREWHERE event_time >= toDateTime('{start}')
    AND event_time < toDateTime('{end}')
WHERE type = 'QueryFinish'
  {project_filter}
GROUP BY normalized_query_hash
HAVING query_count >= {min_count}
ORDER BY p95_ms DESC
LIMIT 20;
```

## Abusive users

```sql
SELECT
    user,
    count() AS query_count,
    uniqExact(normalized_query_hash) AS distinct_hashes,
    round(avg(query_duration_ms), 2) AS avg_ms,
    quantile(0.95)(query_duration_ms) AS p95_ms,
    round(100.0 * countIf(exception_code != 0) / count(), 2) AS error_rate_pct,
    sum(read_bytes) AS total_read_bytes,
    sum(read_rows) AS total_read_rows,
    max(memory_usage) AS max_memory
FROM clusterAllReplicas('{cluster_name}', system.query_log)
PREWHERE event_time >= toDateTime('{start}')
    AND event_time < toDateTime('{end}')
WHERE type = 'QueryFinish'
  {project_filter}
GROUP BY user
ORDER BY query_count DESC, total_read_bytes DESC
LIMIT 20;
```

## Request spikes by user over time

```sql
SELECT
    toStartOfInterval(event_time, INTERVAL 15 MINUTE) AS bucket,
    user,
    count() AS query_count
FROM clusterAllReplicas('{cluster_name}', system.query_log)
PREWHERE event_time >= toDateTime('{start}')
    AND event_time < toDateTime('{end}')
WHERE {project_filter_no_and}
GROUP BY bucket, user
ORDER BY bucket ASC, query_count DESC
LIMIT 500;
```

## Investigation for one normalized query hash

```sql
SELECT
    toStartOfInterval(event_time, INTERVAL 15 MINUTE) AS bucket,
    count() AS query_count,
    uniqExact(user) AS distinct_users,
    round(avg(query_duration_ms), 2) AS avg_ms,
    quantile(0.95)(query_duration_ms) AS p95_ms,
    round(100.0 * countIf(exception_code != 0) / count(), 2) AS error_rate_pct,
    sum(read_bytes) AS total_read_bytes,
    sum(read_rows) AS total_read_rows
FROM clusterAllReplicas('{cluster_name}', system.query_log)
PREWHERE event_time >= toDateTime('{start}')
    AND event_time < toDateTime('{end}')
WHERE normalized_query_hash = {query_hash}
  {project_filter}
GROUP BY bucket
ORDER BY bucket ASC
LIMIT 500;
```

## Top users for one normalized query hash

```sql
SELECT
    user,
    count() AS query_count,
    round(avg(query_duration_ms), 2) AS avg_ms,
    quantile(0.95)(query_duration_ms) AS p95_ms,
    sum(read_bytes) AS total_read_bytes,
    round(100.0 * countIf(exception_code != 0) / count(), 2) AS error_rate_pct
FROM clusterAllReplicas('{cluster_name}', system.query_log)
PREWHERE event_time >= toDateTime('{start}')
    AND event_time < toDateTime('{end}')
WHERE normalized_query_hash = {query_hash}
  {project_filter}
GROUP BY user
ORDER BY query_count DESC, total_read_bytes DESC
LIMIT 20;
```

## Recent examples for one normalized query hash

Use only when needed for interpretation.

```sql
SELECT
    event_time,
    user,
    query_duration_ms,
    read_bytes,
    read_rows,
    exception_code,
    query_kind,
    query
FROM clusterAllReplicas('{cluster_name}', system.query_log)
PREWHERE event_time >= toDateTime('{start}')
    AND event_time < toDateTime('{end}')
WHERE normalized_query_hash = {query_hash}
  {project_filter}
ORDER BY event_time DESC
LIMIT 10;
```

## Notes on cluster safety

Prefer multiple small targeted queries over one huge query.

Good pattern:
1. get compact KPI summary,
2. rank top hashes or users,
3. drill into one offender,
4. optionally fetch a few recent examples.

Avoid:
- dumping raw query text for large windows,
- scanning without time bounds,
- grouping by highly-cardinal raw query text,
- broad historical scans before identifying a focused offender.
