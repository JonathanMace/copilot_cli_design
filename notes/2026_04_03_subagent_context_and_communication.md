# Copilot CLI Subagent Context and Communication

How subagents in GitHub Copilot CLI share context, communicate results, and manage the practical constraints of delegated work.

## Subagent Context: What They Know and Don't Know

When the Copilot CLI spawns a subagent (via the `task` tool), the subagent **does not inherit the parent's conversation context**. Each subagent starts with a blank slate. The only context it receives comes from three sources:

1. **The prompt provided at launch** — this is the primary channel. The parent agent writes a self-contained prompt describing the goal, relevant file paths, constraints, and expected output. If something isn't in the prompt, the subagent doesn't know about it.

2. **Repository custom instructions** (`.github/copilot-instructions.md`) — these load automatically for any agent running in the repo, so conventions and project-level rules propagate without the parent restating them.

3. **Agent templates** (`.agent.md` files) — custom agents defined in `.github/agents/` receive their own system-level instructions baked into their profile. For example, a `research-team` agent has its own role definition, coordination strategy, and subagent-spawning rules.

**What is NOT shared:**

- The parent's conversation history
- The parent's SQL session database
- Any in-memory state or variables
- Tool outputs from the parent's prior calls (grep results, file reads, etc.)

The practical implication is that the parent must **front-load everything** the subagent needs into the prompt. If the parent forgets context, the subagent either figures it out from the codebase or gets it wrong — it cannot ask the parent for clarification.

## How Subagents Report Results Back

When a subagent completes, the parent receives its output through **text responses**. The shape of that response depends on the agent type:

- **`task` agents**: Designed to be terse. They return a brief summary on success (e.g., "All 247 tests passed") and full output on failure (stack traces, compiler errors). This keeps the parent's context clean.

- **`explore` agents**: Return prose — summaries, file paths, code snippets, analysis. Optimized for investigation and understanding questions.

- **`general-purpose` agents**: Return their full response, which can be substantial. These are the most flexible but also the most context-hungry.

The parent does **not** receive:

- The subagent's internal reasoning or chain-of-thought
- Which tools the subagent called or in what order (unless it mentions them)
- Partial or intermediate states — only the final response
- Raw tool outputs (e.g., grep results) — only what the subagent chose to include

## The Filesystem as a Communication Channel

All agents share the same machine and filesystem. This makes the filesystem the main **side-channel** beyond text responses:

- **Permanent repo files**: The most common case. When a subagent is told to "create the auth module" or "write a new skill," it creates or edits real files in the working tree. The filesystem change *is* the deliverable. The text response is more of a status report.

- **Session-state files** (`~/.copilot/session-state/.../files/`): Used for ephemeral artifacts — research summaries, analysis notes, intermediate data that shouldn't be committed. This is less common and typically reserved for planning artifacts or very large outputs.

- **Temp-file-as-IPC**: The pattern of "write to a temp file so the parent can read it" is not standard practice. It's an escape hatch for when text responses would be too large to fit comfortably in the parent's context.

For most tasks, the natural output channel matches the task type:

| Task Type | Primary Output Channel |
|---|---|
| Code implementation | Permanent files in the repo |
| Research / investigation | Text response |
| Build / test execution | Text response (terse summary) |
| Large-scale analysis | File in session-state, referenced by path in text response |

## Sync vs Background: Launch Modes, Not Agent Types

"Subagent" is the general concept — any agent spawned via the `task` tool. "Background" is a **launch mode**, not a different kind of agent. Every subagent is launched in one of two modes:

- **`mode: "sync"`** — The parent blocks and waits for the result. Output arrives immediately in the same turn. Useful for quick questions and simple tasks. Multiple sync agents can run in parallel within the same turn.

- **`mode: "background"`** — The agent runs independently. The parent continues working and is notified when the agent completes. Results are retrieved via `read_agent`. This enables true concurrency — the parent can do other work or launch more agents while waiting.

The agent types (`explore`, `task`, `general-purpose`, custom agents) are the same regardless of launch mode. The only difference is scheduling: does the parent wait, or keep working?

The convention in this repo is to **default to background mode** for most delegated work, keeping the main session responsive.

## Managing Context Pressure from Agent Results

The `read_agent` tool returns the **full turn-by-turn response history** of a background agent. This presents a tension: verbose agents can flood the parent's context window.

Available mitigations:

1. **`since_turn` parameter** — Retrieve only turns after a given index. Useful for incremental reads (e.g., "I already saw turns 0-3, give me 4+"). Doesn't fully solve the problem for final retrieval of verbose agents.

2. **Agent type selection** — The `task` agent type is specifically designed to compress output. Choosing it over `general-purpose` for build/test tasks keeps success-case output small.

3. **Prompt instructions** — Telling the agent to be concise: *"Summarize findings in under 200 words"* or *"Be terse in your final response."* This is the main lever the parent has.

4. **File offloading** — For tasks expected to produce large output, the parent instructs: *"Write the full analysis to [path]. In your response, just provide a 2-3 sentence summary and the file path."* This keeps the text response small while preserving the full output on disk.

**What's NOT available:**

- No automatic summarization of agent output
- No way to request only the final turn (since_turn approximates this)
- No token budget on return values
- No built-in compression mechanism

In practice, individual agents rarely cause problems — most produce a few hundred words. The real context pressure comes from launching many background agents and reading all their results, where cumulative token consumption adds up.

## Key Takeaways

1. **Subagents are stateless.** Every spawn starts fresh. The prompt is the only reliable context channel, supplemented by repo-level instructions that load automatically.

2. **Text responses are the default return path.** File-based output is the exception, reserved for large deliverables or repo-permanent artifacts.

3. **The filesystem is shared but not formally structured as IPC.** Agents can read and write files, and this is the natural output for implementation tasks, but there's no temp-file protocol.

4. **Context management is manual.** The parent must choose the right agent type, write concise prompts, and use file offloading when needed. There's no automatic compression or summarization layer.

5. **Background mode is about scheduling, not capability.** All agent types work the same regardless of sync vs background launch. The choice is about whether the parent waits or continues working.
