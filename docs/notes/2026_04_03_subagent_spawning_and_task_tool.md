---
name: Subagent Spawning and the Task Tool
description: Full reference for the task tool parameters, agent types, launch modes, and model selection when spawning subagents.
copilot_version: 1.0.17
---

# Subagent Spawning and the Task Tool

How the parent agent creates subagents using the `task` tool — the parameters, agent types, launch modes, and model configuration available.

## The Task Tool

The `task` tool is the only way an agent spawns subagents. Each call creates a new agent in a **separate context window** that runs independently from the parent's conversation.

### Parameters

| Parameter | Required | Type | Description |
|-----------|----------|------|-------------|
| `name` | yes | string | Short name used to generate a human-readable agent ID (e.g., `"math-helper"`) |
| `prompt` | yes | string | The instructions for the agent. This is the primary context channel — the subagent knows nothing beyond this, repo instructions, and its agent profile. |
| `agent_type` | yes | enum | One of five agent types (see below) |
| `description` | yes | string | 3-5 word summary shown in the UI as the agent's intent |
| `mode` | optional | enum | `"sync"` or `"background"`. Controls whether the parent blocks or continues working. |
| `model` | optional | string | Override the agent type's default model. Accepts a model ID string. |

### Agent Types

| Type | Default Model Tier | Purpose | Output Style |
|------|-------------------|---------|-------------|
| `explore` | fast/cheap (Haiku) | Codebase exploration, answering questions about code | Prose summaries, file paths, code snippets |
| `task` | fast/cheap (Haiku) | Running commands — builds, tests, lints, installs | Terse on success, full output on failure |
| `general-purpose` | standard (Sonnet) | Complex multi-step work requiring full toolset and high-quality reasoning | Full responses, can be substantial |
| `code-review` | standard (Sonnet) | Reviewing code changes for bugs, security issues, logic errors | High signal-to-noise review comments; will NOT modify code |
| `configure-copilot` | not documented | Managing MCP server configuration | Edits config files and reloads servers |

### Launch Modes

- **`sync`** — Parent blocks and waits. Result arrives in the same turn. Multiple sync agents can run in parallel within the same turn. Use for quick tasks where the parent needs the result immediately to proceed.

- **`background`** — Agent runs independently. Parent continues working and receives a notification when the agent completes. Results are retrieved via `read_agent`. Use for most delegated work.

When `mode` is not specified, the default behavior is not explicitly documented. In practice, the convention in this repo is to always specify the mode explicitly.

### Model Selection

The `model` parameter accepts a model ID string. As of Copilot CLI 1.0.17, the available models are:

**Premium tier:**
- `claude-opus-4.6`
- `claude-opus-4.6-1m` (1M context, internal only)
- `claude-opus-4.5`

**Standard tier:**
- `claude-sonnet-4.6`
- `claude-sonnet-4.5`
- `claude-sonnet-4`
- `goldeneye` (internal only)
- `gpt-5.4`
- `gpt-5.3-codex`
- `gpt-5.2-codex`
- `gpt-5.2`
- `gpt-5.1`

**Fast/cheap tier:**
- `claude-haiku-4.5`
- `gpt-5.4-mini`
- `gpt-5-mini`
- `gpt-4.1`

#### Reasoning Level

There is **no parameter to configure reasoning effort** (e.g., low/medium/high thinking budget) when spawning a subagent. The `model` field accepts only the model name — reasoning behavior is determined entirely by the model itself.

The parent session's `ctrl+t` shortcut toggles "model reasoning display" in the UI, but this controls the parent's rendering of reasoning traces, not the reasoning behavior of subagents.

It is currently unknown whether models with built-in extended thinking (e.g., Opus) automatically use extended thinking when running as subagents, or whether the reasoning mode is fixed by the infrastructure.

## Multi-Turn Agents

Background agents can persist beyond their initial response. When an agent finishes a turn but doesn't exit, it enters an **idle** state:

1. **`list_agents`** shows status as `idle` — "waiting for messages"
2. **`write_agent(agent_id, message)`** sends a follow-up message, waking the agent for another turn
3. **`read_agent(agent_id, since_turn)`** retrieves only new responses, avoiding re-reading earlier turns

The lifecycle: **launch → response → idle → write → response → idle → ... → completed**

Unlike spawning a fresh agent each time, multi-turn agents **retain their full conversation context** across turns — they remember prior messages, tool calls, and file changes. This makes them suitable for iterative refinement without the parent having to re-front-load context.

### Agent Management Tools

| Tool | Purpose |
|------|---------|
| `read_agent(agent_id, since_turn?, wait?, timeout?)` | Retrieve status and results. `since_turn` enables incremental reads. `wait: true` blocks until completion (with optional timeout, max 60s). |
| `write_agent(agent_id, message)` | Send a follow-up message to a running or idle agent. Queued if agent is mid-turn. |
| `list_agents(include_completed?)` | Show all active and completed agents with their status. |

## Open Questions

- What is the default `mode` when not specified — sync or background?
- Can reasoning effort be configured per-subagent, or is it fixed by model selection?
- Is there a maximum number of concurrent background agents?
- Do multi-turn agents eventually time out if left idle, or do they persist indefinitely within the session?
