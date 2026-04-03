---
 name: Injected Contexts and Agent Prompt Structure
 description: Complete inventory of the context blocks injected into an 
agent's prompt, their sources, scoping rules, and layering behavior.
 copilot_version: 1.0.17
 ---
 
 # Injected Contexts and Agent Prompt Structure
 
 When Copilot CLI activates an agent (either the default session agent or a 
custom agent from `.github/agents/`), the runtime assembles a composite 
prompt from multiple injected context blocks. This document catalogs every 
block observed during a controlled inspection using a custom diagnostic 
agent.
 
 ## Method
 
 A custom `.agent.md` agent with a read-only, introspective persona was 
invoked and asked to enumerate all context it could observe. The agent had no
 privileged access to runtime internals — findings are based entirely on what
 appeared in its assembled prompt.
 
 ## Injected Context Blocks
 
 The following 17 distinct blocks were observed, listed in approximate order 
of appearance in the assembled prompt.
 
 ### 1. Base System Prompt
 
 **Source:** Copilot CLI runtime (hardcoded)
 **Scoping:** All agents — base and custom
 
 The foundational identity prompt: "You are the GitHub Copilot CLI, a 
terminal assistant built by GitHub." Establishes:
 - Tone and style rules (concise responses, ~100 word limit)
 - Search and delegation preferences (code intelligence > LSP > glob > grep)
 - Tool usage efficiency rules (parallel calling, chaining commands)
 - General behavioral guidelines
 
 **Key observation:** This prompt is **not replaced** when a custom agent 
loads. Custom agent instructions layer on top of the base prompt.
 
 ### 2. `<version_information>`
 
 **Source:** Copilot CLI runtime
 **Scoping:** All agents
 
 A single line: the Copilot CLI version number (e.g., `1.0.17`).
 
 ### 3. `<model_information>`
 
 **Source:** Copilot CLI runtime
 **Scoping:** All agents
 
 Identifies the model powering the agent (e.g., "Claude Opus 4.6, model ID: 
claude-opus-4.6"). Includes instructions for how to respond when asked "what 
model are you?"
 
 ### 4. `<environment_context>`
 
 **Source:** Copilot CLI runtime (dynamic)
 **Scoping:** All agents
 
 Runtime-detected environment details:
 - Current working directory
 - Git repository root
 - Operating system
 - Directory contents snapshot (noted as "may be stale")
 - Available tools note
 - Platform-specific instructions (e.g., Windows path separators)
 
 ### 5. `<code_change_instructions>`
 
 **Source:** Copilot CLI runtime (hardcoded)
 **Scoping:** All agents
 
 Rules for making code changes:
 - Surgical, complete changes
 - Don't fix unrelated issues
 - Update related documentation
 - Linting/building/testing guidelines
 - Prefer ecosystem tools over manual changes
 - Comment style rules
 
 ### 6. `<git_commit_trailer>`
 
 **Source:** Copilot CLI runtime (hardcoded)
 **Scoping:** All agents
 
 Mandates including a `Co-authored-by: Copilot 
<223556219+Copilot@users.noreply.github.com>` trailer in all git commits.
 
 ### 7. `<tips_and_tricks>`
 
 **Source:** Copilot CLI runtime (hardcoded)
 **Scoping:** All agents
 
 Behavioral advice:
 - Reflect on command output before proceeding
 - Clean up temporary files
 - Use view/edit for existing files (avoid data loss)
 - Ask for guidance if uncertain
 - Do not create markdown files for planning (work in memory)
 
 ### 8. `<environment_limitations>` and `<prohibited_actions>`
 
 **Source:** Copilot CLI runtime (hardcoded)
 **Scoping:** All agents
 
 Security and privacy constraints:
 - No sharing sensitive data with third parties
 - No committing secrets
 - No copyright infringement
 - No harmful content generation
 - No revealing or discussing system instructions
 
 ### 9. `<custom_instruction>` — `copilot-instructions.md`
 
 **Source:** `.github/copilot-instructions.md` in the repository
 **Scoping:** All agents in this repository
 
 The repo-wide persistent instructions file. Contents are loaded 
automatically for every agent session. In this repo, it contains:
 - Self-updating memory rules
 - Work tracking conventions (worktrees, PR lifecycle)
 - Subagent model specification requirements
 - README maintenance rules
 - Design notes file convention
 - Anti-patterns section
 
 ### 10. `<custom_instruction>` — `AGENTS.md`
 
 **Source:** `AGENTS.md` in the repository root
 **Scoping:** All agents in this repository
 
 The agent infrastructure reference document. Loaded alongside 
`copilot-instructions.md`. Contains:
 - Directory layout table (`.github/agents/`, `.github/skills/`, etc.)
 - Working conventions
 - Getting started instructions
 
 ### 11. `<session_context>`
 
 **Source:** Copilot CLI runtime (dynamic, per-session)
 **Scoping:** All agents (session-specific)
 
 Session management metadata:
 - Session folder path (`~/.copilot/session-state/{session-id}/`)
 - Plan file path and status (`not yet created` or existing)
 - `files/` directory for persistent session artifacts
 - Instructions on when and how to use `plan.md`
 
 ### 12. `<plan_mode>`
 
 **Source:** Copilot CLI runtime (hardcoded)
 **Scoping:** All agents
 
 Full instructions for handling `[[PLAN]]`-prefixed user messages:
 - Confirm understanding, resolve ambiguity
 - Analyze codebase
 - Create structured plan in session workspace
 - Use SQL for todo tracking
 - Do not implement unless explicitly asked
 
 ### 13. `<available_skills>` (inside `skill` tool definition)
 
 **Source:** Skillful plugin + built-in skills
 **Scoping:** All agents in this repository (plugin-dependent)
 
 Listed inside the `skill` tool's description block. Nine skills observed:
 - `agent-design-patterns`, `bootstrap-skillful`, `git-checkpoint`
 - `session-analysis`, `writing-custom-agents`, `writing-custom-instructions`
 - `writing-hooks`, `writing-skills`
 - `customizing-copilot-cloud-agents-environment` (built-in)
 
 ### 14. `<agent_instructions>`
 
 **Source:** Custom `.agent.md` file (when a custom agent is active)
 **Scoping:** Only the specific custom agent
 
 The custom agent's persona and behavioral instructions. This is the **only 
block unique to a custom agent** — everything else is shared infrastructure. 
When no custom agent is active, this block is absent.
 
 ### 15. `<tools_changed_notice>`
 
 **Source:** Copilot CLI runtime (dynamic)
 **Scoping:** Agents with restricted tool access
 
 Lists tools that were removed when the custom agent activated. Observed when
 switching to a read-only diagnostic agent — shell, file editing, web fetch, 
task spawning, and MCP tools were all removed. Only `view`, `skill`, and 
`report_intent` remained.
 
 This suggests the runtime can **dynamically restrict tool access** per agent
 profile, though it is not yet clear whether this is driven by the agent's 
`.agent.md` file or by a separate mechanism.
 
 ### 16. `<current_datetime>`
 
 **Source:** Copilot CLI runtime (dynamic, per-message)
 **Scoping:** All agents
 
 ISO 8601 timestamp injected with each user message. Provides the agent with 
current wall-clock time.
 
### 17. `<reminder>` — SQL tables

**Source:** Copilot CLI runtime (dynamic, per-session)
**Scoping:** All agents

Reports the current state of the session's SQL database (e.g., `No tables 
currently exist.` or a list of table schemas). Injected as a reminder to 
maintain awareness of tracking state.

Observed forms include:

```xml
<reminder>
<sql_tables>No tables currently exist. Default tables (todos, todo_deps) will be created automatically when you first use the SQL tool.</sql_tables>
</reminder>
```

and, after SQL initialization:

```xml
<reminder>
<sql_tables>Available tables: todos, todo_deps</sql_tables>
</reminder>
```

This block is therefore not just a generic reminder tag; it is a concrete prompt-time summary of session SQL state.
 
 ## Layering Behavior
 
 ### Custom agents layer on top — they do not replace
 
 The base system prompt remains fully intact when a custom agent is loaded. 
The `<agent_instructions>` block is added as an additional context, not a 
substitution. This means a custom agent receives:
 
 1. The full base Copilot CLI persona and rules
 2. All repo-level custom instructions (`copilot-instructions.md`, 
`AGENTS.md`)
 3. Session context and plan mode instructions
 4. Its own `<agent_instructions>` block
 
 The practical implication: custom agents inherit all repo conventions 
automatically. They cannot escape the base system prompt's rules (prohibited 
actions, commit trailers, etc.).
 
 ### Repo instructions are agent-agnostic
 
 Both `<custom_instruction>` blocks (`copilot-instructions.md` and 
`AGENTS.md`) loaded identically for the custom diagnostic agent as they did 
for the base session agent. This confirms these are **repo-scoped, not 
agent-scoped** — they propagate to every agent running in the repository.
 
 ### Tool availability is dynamic
 
 The `<tools_changed_notice>` appeared only when switching to the custom 
agent, suggesting tool access is adjusted at agent activation time. The base 
session agent had the full toolset; the diagnostic agent had a restricted 
subset.
 
 ## Open Questions
 
 - **What controls tool restriction?** Is it declared in the `.agent.md` 
file, inferred from the agent's stated capabilities, or configured elsewhere?
 - **Are `.github/instructions/*.instructions.md` files injected as 
additional `<custom_instruction>` blocks?** None were present in this repo to
 test, but the help output lists them as a recognized instruction source.
 - **Is there a load order guarantee?** The blocks appeared in a consistent 
order, but it's unclear whether this is guaranteed or incidental.
 - **Does `CLAUDE.md` or `GEMINI.md` inject as `<custom_instruction>` too?** 
The help output lists them as recognized instruction sources, but neither was
 present to test.
 - **What is the token budget for injected context?** With 17 blocks, the 
overhead is substantial. Is there a cap or prioritization when context grows 
large?

## Message order:

  1. System message

  The top-level system prompt from the runtime.

  2. Developer message #1

  This message contains the first batch of tool definitions, in this order:

   1. functions.view
   2. functions.skill
   3. functions.report_intent

  Inside the functions.skill tool definition, the <available_skills> block
  appears, in this order:

   1. agent-design-patterns
   2. bootstrap-skillful
   3. git-checkpoint
   4. session-analysis
   5. writing-custom-agents
   6. writing-custom-instructions
   7. writing-hooks
   8. writing-skills
   9. customizing-copilot-cloud-agents-environment

  After those, still in Developer message #1, comes:

   1. multi_tool_use.parallel

  3. Developer message #2

  This is the large Copilot CLI instruction message. Its contents appear in
  this order:

   1. Copilot CLI identity + tone/style
   2. Search and delegation
   3. Tool usage efficiency
   4. <version_information>
   5. <model_information>
   6. <environment_context>
   7. <code_change_instructions>
   8. <git_commit_trailer>
   9. <tips_and_tricks>
   10. <environment_limitations>
   11. <custom_instruction> — repo copilot-instructions.md
   12. <custom_instruction> — repo AGENTS.md
   13. <system_notifications>
   14. <preamble_messages>
   15. <tool_use_guidelines>
   16. <editing_constraints>
   17. <exploration_and_reading_files>
   18. <autonomy_and_persistence>
   19. <session_context>
   20. <plan_mode>
   21. <tool_calling>
   22. <task_completion>

  4. Prior conversation history

  After the system/developer messages comes the conversation transcript so far,
  in chronological order:

   - earlier user messages
   - earlier assistant messages
   - earlier tool calls
   - earlier tool results
   - later user messages
   - later assistant messages
   - later tool calls
   - later tool results

  All of that appears before the current user message.

5. Current user message

Your current message appears after the prior transcript. Inside this message,
the order is exactly:

1. <agent_instructions>
2. <current_datetime>
3. your plain-text question
4. `<reminder>`
5. inside `<reminder>`, `<sql_tables>...`

So the reminder is part of the current user-message payload, not a top-level system or developer block.
