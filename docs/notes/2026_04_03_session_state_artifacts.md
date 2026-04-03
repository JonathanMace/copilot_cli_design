---
name: Session State Artifacts
description: Detailed inventory of Copilot CLI session-state files and directories, with direct observations from short and long-running sessions.
copilot_version: 1.0.17
---

# Session State Artifacts

This note documents the on-disk artifacts observed under `~/.copilot/session-state/{session-id}/` and what can be concluded about their purpose from direct inspection.

## Scope

The findings in this note come from inspecting:

- The current session directory
- Multiple older session directories
- Several long-running sessions with large `events.jsonl` files

Where possible, conclusions are grounded in file contents rather than naming alone. When a conclusion is based on naming or nearby evidence rather than explicit contents, it is labeled as an inference.

## Core Session Directory Layout

A typical session directory contains some or all of the following:

- `workspace.yaml`
- `events.jsonl`
- `checkpoints/`
- `files/`
- `research/`
- `rewind-snapshots/`
- `vscode.metadata.json`
- `inuse.<pid>.lock`

Longer-lived or more feature-rich sessions may additionally contain:

- `plan.md`
- `session.db`
- `heartbeat.log`
- `keepalive.bat`
- `keepalive.ps1`

## Artifact-by-Artifact Inventory

### `workspace.yaml`

**Status:** directly identified

This is the primary metadata file for the session. Observed fields include:

- `id`
- `cwd`
- `git_root`
- `repository`
- `host_type`
- `branch`
- `summary`
- `summary_count`
- `created_at`
- `updated_at`

This file appears to describe the session's workspace attachment and summary metadata rather than the conversation itself.

### `events.jsonl`

**Status:** directly identified

This is the session's main event log. It is newline-delimited JSON and contains low-level session events such as:

- `session.start`
- `user.message`
- `assistant.turn_start`
- `assistant.message`
- `tool.execution_start`
- `tool.execution_complete`

Each event includes structured metadata such as timestamps, interaction IDs, parent IDs, tool arguments, and tool results.

This file grows large in long-running sessions. Several examples were observed:

- ~541 lines
- ~1,566 lines
- ~2,702 lines
- ~10,178 lines
- ~11,910 lines
- ~93,180 lines

So `events.jsonl` appears to be the detailed historical record of the session's execution.

### `checkpoints/`

**Status:** partly direct, partly inferred

The `checkpoints` directory contains checkpoint-related materials.

The directly observed file inside it is:

- `checkpoints/index.md`

That file states:

- checkpoints are listed in chronological order
- checkpoint 1 is the oldest
- higher numbers are more recent
- the index tracks checkpoint number, title, and file

In some sessions the index was empty; in others the directory contained markdown checkpoint files.

**Conclusion:** `checkpoints/` stores user-visible checkpoint summaries/history, and `checkpoints/index.md` is the chronological index for those documents.

### `files/`

**Status:** directly identified

This directory is used for persistent session artifacts that are not part of the repository itself.

Evidence:

- Session context explicitly describes `files/` as persistent storage for session artifacts
- In longer sessions, the directory contained real markdown artifacts such as:
  - `submission-plan.md`
  - `repo-layout-audit.md`
  - `advisory-board-report.md`
  - `external-examiner-report.md`

These are clearly user-facing or agent-generated work products rather than runtime internals.

**Conclusion:** `files/` is the general-purpose artifact directory for session outputs that should persist across checkpoints but not be committed to the repo.

### `research/`

**Status:** directly identified

This directory stores research outputs. Evidence includes:

- numbered markdown files such as:
  - `001-corrected-modal-analysis-and-c.md`
  - `014-full-review-cycle-new-agents-a.md`
  - `028-p1-external-review-revision.md`
- an `index.md`

This is consistent with `/research` or research-style work producing structured outputs over time.

In small sessions the directory may exist but remain empty or unreadable as a directory artifact.

**Conclusion:** `research/` is a dedicated artifact area for research results, distinct from the more general `files/` directory.

### `rewind-snapshots/`

**Status:** directly identified at index level, partly inferred at payload level

Observed contents include:

- `rewind-snapshots/index.json`
- `rewind-snapshots/backups/`

The `index.json` file is explicit. It contains snapshot records with fields such as:

- `snapshotId`
- `eventId`
- `userMessage`
- `timestamp`
- `fileCount`
- `gitCommit`
- `gitBranch`
- `backupHashes`
- `files`

This strongly ties rewind snapshots to specific user messages and to the repo state at the time of capture.

The `backups/` subdirectory contains hash-like backup entries. Combined with the `backupHashes` field in `index.json`, the strongest supported conclusion is:

**Inference:** `rewind-snapshots/backups/` stores the backing payloads referenced by rewind snapshots, likely used by `/rewind` or related restoration behavior.

### `plan.md`

**Status:** directly identified

Some sessions contain a `plan.md` file at the session root.

Observed contents show that it is:

- human-readable
- markdown
- session-specific
- used to track multi-phase work
- updated over time

This aligns with the runtime instructions that say multi-phase work may create and maintain `plan.md` in the session folder.

**Conclusion:** `plan.md` is the human-readable planning artifact for a session, not a low-level runtime store.

### `session.db`

**Status:** directly identified through causal test

This file did not exist in the current session initially.

Then the `sql` tool was used successfully against the session database. Immediately afterward:

- `session.db` appeared in the session directory
- querying `sqlite_master` returned the default tables:
  - `todos`
  - `todo_deps`

This is the strongest direct result in the entire inspection:

**Conclusion:** `session.db` is the backing SQLite file for the per-session `sql` tool database.

This is no longer merely an inference; the file's appearance was directly correlated with first use of the `sql` tool.

### `vscode.metadata.json`

**Status:** weakly identified

In the inspected session this file contained:

```json
{}
```

That confirms the file exists, but not what it stores when populated.

**Inference:** it is reserved for VS Code-related metadata or integration state. The filename strongly suggests editor linkage, but current evidence does not show the populated schema.

### `heartbeat.log`

**Status:** directly identified

This file contained repeated lines like:

`KEEPALIVE`

with timestamps at regular intervals.

**Conclusion:** `heartbeat.log` records periodic keepalive activity for long-running sessions.

### `keepalive.bat` and `keepalive.ps1`

**Status:** strongly inferred

These files were observed in a long-running Windows session alongside `heartbeat.log`.

Given:

- their names
- the operating system
- the presence of regular `KEEPALIVE` entries

the strongest supported conclusion is:

**Inference:** these are helper scripts used to implement the session keepalive mechanism.

### `inuse.<pid>.lock`

**Status:** inferred

These lock files were observed in active session directories, often with a PID-like suffix.

**Inference:** they mark a session as currently in use and/or help prevent conflicting concurrent ownership of the same session.

The naming pattern is strongly suggestive, but the internal semantics were not directly inspected.

## Differences Between Short and Long Sessions

Shorter or lighter sessions tend to have only the core set:

- `workspace.yaml`
- `events.jsonl`
- `checkpoints/`
- `files/`
- `research/`
- `rewind-snapshots/`
- `vscode.metadata.json`
- `inuse.<pid>.lock`

Longer-lived sessions accumulate additional infrastructure:

- `plan.md`
- `session.db`
- `heartbeat.log`
- keepalive scripts

This suggests the session-state layout is feature- and lifecycle-dependent rather than fixed in all cases.

## Strongest Conclusions

The most strongly supported findings are:

1. `workspace.yaml` stores session/workspace metadata
2. `events.jsonl` is the detailed event log for the session
3. `checkpoints/` stores checkpoint summaries with `index.md` as an index
4. `files/` stores persistent non-repo session artifacts
5. `research/` stores research outputs
6. `rewind-snapshots/index.json` records rewind metadata
7. `session.db` is the backing store for the per-session `sql` database
8. `heartbeat.log` records keepalive activity

## Remaining Uncertainties

Some details remain only partly known:

- the populated schema of `vscode.metadata.json`
- the exact file format inside `rewind-snapshots/backups/`
- the precise locking semantics of `inuse.<pid>.lock`
- whether any other built-in tools persist additional state into `session.db` beyond the `sql` tool

## Practical Mental Model

The session directory appears to be a layered workspace for:

1. **identity and metadata** (`workspace.yaml`)
2. **execution history** (`events.jsonl`)
3. **human-readable planning and checkpointing** (`plan.md`, `checkpoints/`)
4. **artifact storage** (`files/`, `research/`)
5. **recovery/undo state** (`rewind-snapshots/`)
6. **structured per-session data** (`session.db`)
7. **liveness bookkeeping** (`heartbeat.log`, keepalive scripts, lock files)

That is, a session is not merely a chat log. It is a durable work container with multiple storage layers for transcript, plans, artifacts, rewind state, and structured data.
