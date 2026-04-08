---
name: clickhouse-query-log
description: analyze clickhouse system.query_log data through a dedicated read-only user to extract kpis, compare two date ranges, investigate slow queries, abusive users, and activity for a normalized query hash. use when a user asks for operational analysis of query logs, latency trends, error rates, request volume, heavy users, project scoped activity, database or table level activity, or safe sql against system database tables with strong query guardrails. supports multiple clickhouse clusters — always asks the user which cluster to target.
---

# ClickHouse Query Log

## Overview

Analyze `system.query_log` safely and consistently. Access happens through a dedicated ClickHouse user with read-only permissions on `system` tables only.

1. Ask the user which cluster to target before running any query (see **Cluster selection** below).
2. Translate the user's operational question into a safe bounded query or set of queries.
3. Enforce guardrails that protect the cluster.
4. Compute the requested KPI or comparison.
5. Identify slow queries, abusive users, and query-hash level behavior.
6. Summarize findings in plain language with the exact scope and filters used.

Read `references/query_patterns.md` before writing SQL.

## Cluster selection

Clusters are sharded and replicated. Two pieces of information are required before running any query:

1. **Host URL** — the ClickHouse endpoint to connect to (e.g. `localhost:8123`).
2. **Cluster name** — the internal cluster name used in `clusterAllReplicas()` (e.g. `mycluster`). Ask explicitly if not provided; do not guess or default.

**Always ask the user for both values before running any query.** Record them and use them for every query in the session. State both in the **Scope** section of every output.

**Always query `clusterAllReplicas('{cluster_name}', system.query_log)` instead of `system.query_log` directly.** This ensures data is collected from every shard and every replica, not just the node receiving the connection.

## Operating assumptions

- Always connect as ClickHouse user `llm`. Never use any other user.
- Treat `system.query_log` as the primary source.
- Never run unbounded exploration queries.
- Always constrain by time window.
- Prefer project-scoped analysis. When the request is about application traffic, require a `project_key` filter or explicitly state that the result is not project-scoped.
- When the user asks who is sending too many requests, analyze the `user` dimension and, if available in the query text, infer whether the traffic is associated with a specific `project_key`.
- For query-level investigations, prefer `normalized_query_hash` over raw query text.
- Aggregate early. Return small result sets.

## Mandatory safety rules

Apply these rules on every query unless the user explicitly asks for a different safe bound and it is still narrow enough.

1. Always include a time predicate on `event_time` or `event_date`.
2. Default to the last 24 hours if the user gave no time window.
3. Never scan more than 7 days in a single raw query-log query unless the task is explicitly historical and the SQL is heavily aggregated.
4. Always filter to `type = 'QueryFinish'` when computing latency, rows, bytes, memory, and request-volume KPIs, unless failed queries are the topic.
5. When analyzing failures, use `exception_code != 0` or equivalent failure logic and keep the same time bound.
6. Always add `LIMIT` for ranked outputs.
7. Prefer `PREWHERE event_time >= ... AND event_time < ...` when practical.
8. Select only the columns needed for the task.
9. Avoid returning raw query text by default. Use `normalized_query_hash`, dimensions, and summarized metrics first. Only show query text when it is essential and safe.
10. If the user request is missing both a time window and a project scope, proceed with a short, safe default query and clearly state the default scope used.
11. Refuse or narrow any request that would require a broad full-table scan, highly cardinal raw dump, or unnecessary query text extraction.

## Workflow

1. **Select cluster** — confirm the host URL and cluster name with the user before running any query. Both are required.
2. Identify the task type:
   - KPI summary for one window
   - comparison of two windows
   - slow-query investigation
   - abusive-user investigation
   - normalized-query-hash investigation
   - database-scoped or table-scoped investigation
3. Determine filters:
   - required time range
   - `project_key` scope when relevant
   - database scope (`databases` array) when the user targets a specific database
   - table scope (`tables` array) when the user targets a specific table
   - optional filters such as `user`, `query_kind`, `client_name`, `normalized_query_hash`
4. Choose the smallest safe query template from `references/query_patterns.md`.
5. Execute only the minimum number of queries needed.
6. Validate that the result is bounded and interpretable.
7. Summarize:
   - cluster used
   - exact time window(s)
   - exact filter set
   - KPI definitions used
   - top findings
   - caveats such as missing `project_key` extraction or unavailable dimensions

## Database and table scope

`system.query_log` stores touched databases and tables as arrays:

- `databases Array(String)` — databases referenced by the query.
- `tables Array(String)` — fully-qualified table names (e.g., `mydb.orders`) referenced by the query.

When the user asks for activity on a specific database or table, add the appropriate array predicate. See filter snippets in `references/query_patterns.md`.

State in the output whether the scope came from the structured array column or from parsing raw query text, and note that `databases`/`tables` may be empty for certain query types (e.g., mutations, system queries).

## Interpreting project scope

The user said queries should always define something like `project_key = 'project_XYZ'`.

Use this rule:

- If `project_key` exists as a structured column in the environment, filter on that column.
- Otherwise, assume `project_key` must be extracted from the SQL text. In that case, use a conservative predicate such as `positionCaseInsensitive(query, 'project_key') > 0` plus a narrow match for the target project value when given.
- Be explicit in the output whether project scoping came from a real column or from parsing query text.
- Do not claim exact per-project attribution if the only evidence is ambiguous free-form text.

## Default KPI set

For a single time window, compute a compact KPI block with some or all of:

- total query count
- distinct users
- error count and error rate
- p50, p95, p99 `query_duration_ms`
- average `query_duration_ms`
- total and average `read_rows`
- total and average `read_bytes`
- total and average `result_rows`
- peak and p95 `memory_usage`
- top query kinds by volume

Adjust to available columns and the user's question.

## Comparison guidance

When comparing two date ranges:

- keep the same metric definitions on both sides,
- label windows clearly,
- compute absolute delta and percent delta,
- call out regressions in latency, error rate, scan volume, and user/request concentration,
- avoid mixing successful and failed-query populations unless the user asked for both.

## Slow-query guidance

For slow queries:

- default rank by p95 or max `query_duration_ms`, depending on whether the user cares about chronic slowness or worst outliers,
- group by `normalized_query_hash` first,
- include count, p50, p95, p99, max duration, error rate, bytes read, rows read, and representative dimensions,
- only include raw query text when needed to interpret the finding.

## Abusive-user guidance

For abusive users:

- rank by request volume first,
- then examine scan intensity and latency impact,
- compute query count, error rate, total read bytes, total read rows, avg duration, p95 duration, and distinct normalized hashes,
- if the request is project-specific, keep the same `project_key` filter,
- call a user "abusive" only from evidence such as extreme request concentration, extreme scan volume, consistently slow queries, or repeated failures.

## Normalized query hash guidance

When the user provides a `normalized_query_hash`, build a focused investigation:

- request count over time
- distinct users running it
- latency distribution
- error rate
- bytes and rows scanned
- top projects if project extraction is possible
- example recent executions only if necessary and safe

If the user asks for a hash without a time window, default to the last 24 hours.

## Output format

Use this structure unless the user asks for a different format.

# Query Log Analysis

## Scope
- Cluster: ...
- Time window: ...
- Comparison window: ... if applicable
- Filters: ...
- Database/table scope: ... if applicable
- Population: successful queries / failed queries / all queries

## KPI summary
- Metric: value
- Metric: value

## Key findings
- Finding with supporting numbers
- Finding with supporting numbers
- Finding with supporting numbers

## Top offenders
- User / normalized query hash / query kind: short evidence

## Caveats
- Missing dimension, query-text extraction assumption, or other limitation

## Next actions
- 1 to 3 concrete follow-ups

## Communication rules

- State defaults explicitly.
- Name the metrics exactly.
- Separate facts from inference.
- Prefer concise ranked lists over large dumps.
- If project attribution required parsing SQL text, say so.
- If the request is unsafe or too broad, narrow it and explain the safer scope you used.
