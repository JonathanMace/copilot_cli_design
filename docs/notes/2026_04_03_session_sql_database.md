---
name: Session SQL Database
description: Detailed notes on the per-session SQL database, how it is materialized, what is stored in it, and what is currently known about its role.
copilot_version: 1.0.17
---

# Session SQL Database

This note documents the Copilot CLI per-session SQL database as exposed through the `sql` tool and backed by `session.db` in the session-state directory.

## The Core Finding

The most important observation is a causal one:

1. The current session directory was inspected before any successful use of the `sql` tool
2. At that time, there was **no** `session.db` file
3. The `sql` tool was then used successfully against the `session` database
4. Immediately afterward, a new file named `session.db` appeared in the session directory
5. Querying `sqlite_master` then returned the default tables:
   - `todos`
   - `todo_deps`

This supports a strong conclusion:

**`session.db` is the on-disk backing store for the per-session SQLite database exposed by the `sql` tool.**

## Session Database vs Session Store

The `sql` tool exposes two logical databases:

1. **`session`** — writable, per-session
2. **`session_store`** — read-only, cross-session/global history store

Only the `session` database has direct evidence tying it to the current session directory's `session.db`.

There is no evidence from this inspection that `session_store` is stored in the session directory. In fact, its cross-session nature suggests it is backed elsewhere.

## Materialization Behavior

One especially useful detail is that `session.db` appears to be **lazy-created**.

Observed sequence:

- session starts
- no `session.db` file exists yet
- `sql` tool is used
- `session.db` appears

This suggests the per-session database is not necessarily created for every session up front. It may only be materialized when the `sql` tool is actually used.

That behavior is operationally important because it means:

- not every session directory will contain `session.db`
- the presence of `session.db` may indicate SQL usage rather than just session existence

## Default Tables

After first successful SQL use, the following tables were visible:

- `todos`
- `todo_deps`

These match the runtime instructions for the session database, which describe them as pre-existing default tables for todo tracking.

### `todos`

The runtime documentation describes `todos` as the primary task-tracking table, with fields such as:

- `id`
- `title`
- `description`
- `status`
- `created_at`
- `updated_at`

Statuses include:

- `pending`
- `in_progress`
- `done`
- `blocked`

### `todo_deps`

The runtime documentation describes `todo_deps` as the dependency table, linking one todo to another via:

- `todo_id`
- `depends_on`

This enables dependency-aware task tracking within a session.

## What the Session Database Is For

From the tool documentation and observed behavior, the per-session SQL database is intended for structured operational state during a session. This includes things like:

- todo tracking
- dependency tracking
- test case tracking
- batch processing queues
- session key-value state
- intermediate analysis results

The important distinction is that this database is for **structured machine-friendly session state**, not prose.

## What the Session Database Is Not

Current evidence suggests `session.db` is **not**:

- the primary storage for `plan.md`
- the main transcript store for the session
- the backing store for the repo-wide `store_memory` mechanism

Those conclusions rest on the following observations:

### Not `plan.md`

`plan.md` is a separate markdown file in the session root and is explicitly treated by the runtime as the human-readable planning artifact.

That makes it unlikely that the SQL database is the primary plan store, even though planning workflows may choose to mirror operational pieces of a plan into SQL.

### Not the transcript log

The transcript/event history clearly lives in `events.jsonl`, which records user messages, assistant messages, and tool executions in chronological order.

### Probably not `store_memory`

`store_memory` is framed as cross-session memory. By contrast, `session.db` is clearly session-local and physically stored under a single session directory. That makes it a poor fit for repo-wide or long-lived memory.

This last point is still an inference, but it is a strong one.

## Relationship to Other Session Artifacts

The session database fits into the larger session-state layout like this:

- `workspace.yaml` -> session metadata
- `events.jsonl` -> event history
- `plan.md` -> human-readable plan
- `files/`, `research/` -> durable artifacts
- `rewind-snapshots/` -> undo/rewind metadata and backups
- `session.db` -> structured per-session relational state

This gives the session two complementary storage modes:

1. **Markdown/filesystem storage** for human-facing notes and artifacts
2. **SQLite storage** for structured relational state

## Security and Access Constraints

One useful observation from experimentation:

- `PRAGMA database_list` was blocked by the SQL tool for security reasons

So although the database is SQLite-backed, the `sql` tool does not expose every SQLite command equally. Some low-level inspection primitives may be restricted even when ordinary `SELECT`, `INSERT`, `UPDATE`, and schema queries are allowed.

However, querying `sqlite_master` was allowed:

```sql
SELECT name FROM sqlite_master WHERE type = 'table' ORDER BY name;
```

This is useful for safe schema inspection.

## Practical Usage Model

The session SQL database is best understood as a scratch relational workspace that persists for the duration of the session and survives across turns.

It is useful when:

- task state needs to be queried repeatedly
- dependencies matter
- a workflow benefits from filtering, joins, grouping, or structured updates
- the agent needs durable intermediate state without inventing ad hoc text files

It is less appropriate when:

- the output is prose
- the user needs to edit/read the state directly as narrative text
- the artifact is meant to be committed into the repo

In those cases, `plan.md` or files under `files/` are usually the better fit.

## User-Message Reminder Integration

One additional runtime behavior became directly visible after the session SQL database had been initialized and the default tables existed.

User messages began arriving with a suffixed reminder block of this form:

```xml
<reminder>
<sql_tables>Available tables: todos, todo_deps</sql_tables>
</reminder>
```

Earlier in the session, before SQL initialization, the reminder instead reported:

```xml
<reminder>
<sql_tables>No tables currently exist. Default tables (todos, todo_deps) will be created automatically when you first use the SQL tool.</sql_tables>
</reminder>
```

This is useful for two reasons:

1. it confirms that SQL-table state is surfaced back into the prompt at user-message time
2. it shows that the runtime tracks session SQL state and re-injects a summary of that state as context

The strongest supported reading is that the reminder is not itself the database, but a prompt-time summary derived from current session SQL state.

Operationally, this means the agent can often see the high-level SQL-table state without having to query SQLite immediately.

## Remaining Open Questions

Even with the causal evidence above, a few details remain open:

1. Does `session.db` contain anything besides the `sql` tool's tables?
2. Are there hidden tables created by the runtime under certain workflows?
3. Does plan mode ever mirror plan data into SQL automatically, or only when the agent explicitly uses the `sql` tool?
4. Is there any retention or cleanup behavior tied to old `session.db` files?
5. Are there indexes or performance optimizations added automatically for large sessions?

## Strongest Safe Summary

The safest detailed summary is:

- `session.db` is the on-disk SQLite backing file for the per-session `sql` tool database
- it appears to be created lazily on first SQL use
- it initially exposes at least `todos` and `todo_deps`
- it is intended for structured session-local state, not prose planning or transcript logging
- it complements, rather than replaces, other session artifacts like `plan.md` and `events.jsonl`
