---
name: Open Blind Spots for Agent Application Design
description: High-value Copilot CLI runtime behaviors that remain unclear and could materially affect the design of complex agent-based applications.
copilot_version: 1.0.17
---

# Open Blind Spots for Agent Application Design

This note collects the most important Copilot CLI runtime behaviors that still remain unclear after the current round of inspection.

The intended audience is a developer designing a complex agent-based system on top of Copilot CLI who needs to know which internal behaviors are reliable enough to design around and which remain uncertain.

## Scope

This note **omits** areas that the skillful skills already document clearly, including:

- custom agent frontmatter keys (`name`, `description`, `tools`, `model`, `disable-model-invocation`, `user-invocable`, `mcp-servers`, `metadata`, `target`)
- custom agent placement and naming conventions
- basic skill frontmatter and `SKILL.md` structure
- broad authoring guidance for custom agents and skills

Those are already described concretely in:

- `writing-custom-agents`
- `writing-skills`

Instead, this note focuses on the unresolved runtime-level questions that matter once a developer moves from authoring files to building real orchestration and automation patterns.

## Why These Blind Spots Matter

For simple use, Copilot CLI can be treated like a conversational coding assistant.

For serious agent-based applications, that is not enough. Design choices depend on details like:

- whether background agents are durable or ephemeral
- whether queued messages are ordered and reliable
- whether context compaction silently changes what an agent "remembers"
- whether permissions and tool restrictions propagate predictably
- whether session restore semantics are safe enough to rely on operationally

The behaviors below are therefore not edge cases. They are architectural constraints.

## 1. Background Agent Lifecycle Semantics

### What is known

At the surface level, the runtime exposes:

- `task` for spawning subagents
- `read_agent`
- `write_agent`
- `list_agents`

The visible statuses include at least:

- `running`
- `idle`
- `completed`
- `failed`
- `cancelled`

It is also documented that:

- background agents may become `idle`
- idle agents can receive new messages
- messages sent while the agent is busy are queued

### What remains unclear

- How long can an agent remain idle before the runtime expires it?
- Is idleness preserved across `/restart`?
- Is idleness preserved across `/resume`?
- Is there any automatic garbage collection of abandoned agents?
- Can an agent move from `failed` back to `idle` after follow-up input, or is failure terminal?
- What exactly distinguishes `completed` from `idle` in lifecycle terms?

### Why this matters

This determines whether a background agent can be treated as:

1. a durable worker process
2. a conversational actor with mailbox semantics
3. a one-shot task with a brief afterlife

Those are radically different design assumptions.

## 2. `write_agent` Delivery Guarantees and Mailbox Behavior

### What is known

The runtime states that:

- `write_agent` sends a new user turn to a running or idle background agent
- if the agent is running, the message is queued

### What remains unclear

- Are queued follow-ups guaranteed to be delivered in FIFO order?
- If multiple `write_agent` calls arrive quickly, are they serialized exactly as sent?
- Is there any deduplication or collapse of queued messages?
- Is there a queue length limit?
- Can queued messages be dropped if an agent exits or is cancelled mid-flight?
- Does the sender receive any positive acknowledgment beyond later agent output?

### Why this matters

Anyone designing a supervisor-worker architecture needs to know whether `write_agent` behaves like:

- a reliable mailbox
- a best-effort message queue
- a conversational convenience layer with no strong delivery contract

Without that knowledge, retry logic and orchestration semantics remain guesswork.

## 3. Context Compaction Mechanics

### What is known

The CLI exposes `/compact`, and the runtime clearly supports long sessions whose prompt cannot consist entirely of raw transcript forever.

There is also evidence that:

- current prompt assembly includes prior conversation history
- injected runtime blocks remain visible around that history
- long sessions persist substantial event history on disk

### What remains unclear

- What exactly triggers compaction automatically, if anything?
- Does compaction replace earlier turns with a visible summary message?
- Is compaction lossy with respect to tool outputs?
- Does it preserve precise wording or only summarized intent?
- Is compaction different in regular mode, plan mode, and autopilot mode?
- When a subagent is spawned after compaction, what upstream history is effectively represented in the parent-supplied prompt?

### Why this matters

For long-lived agent applications, compaction determines:

- whether prior decisions remain reliably accessible
- whether exact prior constraints survive
- whether agents may begin operating on summarized rather than raw history

That directly affects trust in long-running autonomous workflows.

## 4. Hidden vs Visible Prompt State

### What is known

Many prompt components are directly visible:

- system and developer layers
- repo instructions
- custom agent instructions
- current-message tags like `<current_datetime>` and `<reminder>`

### What remains unclear

- Is there any hidden orchestration prompt material not exposed in the visible transcript?
- Does autopilot add invisible steering beyond the visible mode differences?
- Do compacted summaries, if any, appear as visible transcript entries or as hidden internal state?
- Are subagent prompts built entirely from visible materials plus the supplied prompt, or are there additional hidden runtime templates?

### Why this matters

Users building sophisticated agent systems need to know whether prompt reasoning is explainable from visible artifacts alone, or whether important steering may exist in hidden layers they cannot inspect.

## 5. Concurrency Limits and Scheduling Policy

### What is known

The system supports:

- parallel tool calls
- multiple background agents
- multiple background shell sessions

Large and long-lived sessions clearly exist.

### What remains unclear

- Is there a hard cap on background agents per session?
- Is there a hard cap on active shell sessions?
- Are some agent types prioritized over others?
- Are there per-model concurrency limits?
- Do heavy tasks starve lighter tasks?
- Is scheduling fair, round-robin, priority-based, or purely opportunistic?

### Why this matters

This is crucial for any design that wants to fan out work across multiple agents. Without knowing the scheduler's shape, throughput and latency estimates are unreliable.

## 6. Session Restore Semantics

### What is known

The CLI exposes:

- `/resume`
- `/restart`
- session-state directories
- persistent artifacts like `events.jsonl`, `plan.md`, `session.db`, `files/`, and `research/`

### What remains unclear

- Does `/resume` restore background agents?
- Does it restore idle agents specifically?
- Does it restore shell sessions?
- Does it restore the current SQL database automatically and in full?
- Is the same reminder state regenerated on resume?
- Does `/restart` keep the exact same prompt-visible session identity, or merely reload artifacts into a fresh runtime?

### Why this matters

If resume semantics are weak, then application designs must assume transient workers and explicit reconstruction. If resume semantics are strong, richer persistent-agent patterns become viable.

## 7. Permission and Restriction Propagation

### What is known

We observed:

- tool restrictions changing with custom-agent frontmatter
- runtime permission commands such as `/allow-all`, `/add-dir`, and related controls

The skillful docs also clearly document that custom-agent frontmatter can constrain tools.

### What remains unclear

- How do runtime-granted permissions interact with custom-agent tool restrictions?
- If the parent has broad permissions, do subagents inherit them automatically?
- Can a custom agent further narrow inherited permissions?
- Are URL permissions inherited by subagents?
- Are directory allowlists copied, intersected, or recalculated?
- Does a manually selected custom agent get a different permission envelope than an auto-delegated one?

### Why this matters

Permission propagation determines whether a multi-agent system can safely rely on role separation, least privilege, and constrained workers.

## 8. Tool Failure and Retry Semantics

### What is known

Failures are clearly represented in the runtime and in event logs. Some tools return concise success and richer failure output.

### What remains unclear

- Are any tools retried automatically by the runtime?
- Are network-backed tools treated differently from local tools?
- Are failures surfaced identically in event logs and live prompt output?
- Can partial output be resumed or replayed reliably?
- Does a failure in one parallel branch affect sibling branches?

### Why this matters

Without knowing retry semantics, application-level recovery logic risks either duplicating retries unnecessarily or failing to compensate for missing retries.

## 9. `session_store` Architecture

### What is known

The `sql` tool exposes:

- `session` — writable, per-session
- `session_store` — read-only, cross-session/global history

We also know that `session.db` is the backing store for the per-session database.

### What remains unclear

- Where is `session_store` physically stored?
- Is it SQLite-based?
- Is it global per user, per machine, or broader?
- What are its update and retention semantics?
- Are writes to it near-real-time or checkpointed later?
- Is it derived entirely from session-state artifacts, or maintained as a separate pipeline?

### Why this matters

For agent applications that want to exploit long-term memory, `session_store` may be the most important persistence layer in the system. Its trustworthiness depends on understanding its refresh model and scope.

## 10. `session.db` Beyond the Obvious Tables

### What is known

The `session.db` file appears on first successful `sql` use and exposes at least:

- `todos`
- `todo_deps`

### What remains unclear

- Are hidden runtime tables added later?
- Are indexes created automatically?
- Are there bookkeeping tables not shown in the basic query?
- Do other features write to the same file over time?

### Why this matters

If `session.db` is shared by multiple runtime features, then application code must avoid accidental collisions and understand its broader schema. If it is dedicated purely to `sql`, it is a safer workspace.

## 11. Rewind Semantics at the File and Session Level

### What is known

We observed:

- `rewind-snapshots/index.json`
- `rewind-snapshots/backups/`
- snapshot metadata tied to specific user messages

### What remains unclear

- Does `/rewind` restore only repository files, or also session artifacts?
- Does it affect `plan.md`?
- Does it affect `session.db`?
- Does it affect background-agent state?
- Is rewind purely filesystem-level, or also prompt-history aware?

### Why this matters

A complex agent application may rely on rewind as a safety mechanism. That is only safe if the rollback boundary is clearly understood.

## 12. Tool Vocabulary and Aliasing for Agent Frontmatter

### What is known

The skillful skill documents a `tools` field in custom-agent frontmatter and gives an example like:

```yaml
tools: ["read", "search", "edit"]
```

### What remains unclear

- What is the authoritative full vocabulary of allowed tool names?
- Are there aliases mapping high-level names like `read` to concrete tool names like `view`?
- Are namespace-qualified names allowed?
- Are groups/macros supported?
- What happens if an unknown tool name is listed?

### Why this matters

This is the remaining blind spot around frontmatter tooling. The existence of the field is documented; the exact runtime vocabulary still is not.

## 13. Model Behavior Beyond Model Selection

### What is known

The task/subagent interface supports a `model` string. No separate reasoning-level parameter is visible.

### What remains unclear

- Are hidden model-specific reasoning modes enabled automatically?
- Do different models have different system-prompt augmentations?
- Are there different token or latency budgets by model tier?
- Does background behavior differ by model?

### Why this matters

If the platform silently changes reasoning depth, latency, or behavior by model, then application-level scheduling and cost assumptions may be wrong.

## 14. Mode-Specific Hidden Orchestration

### What is known

Regular mode, plan mode, and autopilot mode differ in visible behavior and prompt shaping.

### What remains unclear

- Does autopilot add hidden continuation pressure beyond the visible instructions?
- Does plan mode affect any hidden execution heuristics beyond the visible `[[PLAN]]` workflow?
- Do mode changes affect subagent spawning decisions internally?

### Why this matters

If mode changes alter hidden orchestration policies, then mode becomes an architectural parameter rather than just a user-interface preference.

## Practical Prioritization

If the goal is to reduce uncertainty for builders of complex agent systems, the best next investigations are probably:

1. **Background agent lifecycle and mailbox semantics**
2. **Context compaction and hidden prompt state**
3. **Permission propagation and tool restriction interaction**
4. **Session restore and rewind scope**
5. **`session_store` architecture**

These are the areas most likely to affect correctness, safety, and scalability.

## Strongest Safe Summary

The biggest remaining blind spots are no longer about how to *author* agents and skills. The skillful documentation already covers much of that.

The real uncertainty now sits at the runtime boundary:

- lifecycle
- delivery guarantees
- compaction
- hidden prompt state
- scheduler behavior
- permission inheritance
- restore/rewind scope
- persistence architecture

Those are the behaviors most likely to surprise an advanced user building a real agent application on top of Copilot CLI.
