# agent-stuff

This repository contains skills and extensions that I use in some form with projects. Note that I usually fine-tune these for projects so they might not work without modification for you.

It is released on npm as `mitsupi` for use with the [Pi](https://buildwithpi.ai/) package loader.

## Skills

All skill files are in the [`skills`](skills) folder:

* [`/clickhouse-query-log`](skills/clickhouse-query-log) - Claude Skill for analyzing ClickHouse system.query_log across multiple clusters with KPI summaries, slow-query and abusive-user investigations, and database/table-scoped analysis
* [`/commit`](skills/commit) - Claude Skill for creating git commits using concise Conventional Commits-style subjects
* [`/update-changelog`](skills/update-changelog) - Claude Skill for updating changelogs with notable user-facing changes
