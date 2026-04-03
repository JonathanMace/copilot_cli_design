---
name: Regular, Plan, and Autopilot Mode Differences
description: Observed differences between regular mode, plan mode, and autopilot mode, especially visible message prefixes, reminder blocks, and runtime-driven behavior changes.
copilot_version: 1.0.17
---

# Regular, Plan, and Autopilot Mode Differences

This note documents the differences observed in a single Copilot CLI session while switching between regular mode, `[[PLAN]]` mode, and autopilot mode.

The emphasis is on two things:

1. **Message structure** — what visibly appeared before or after the user's prose
2. **Behavioral changes** — how the runtime nudged or constrained the agent's behavior in each mode

## Evidence Boundary

This note is based on **directly visible prompt content** and the agent's observed behavior in-session. It does **not** claim to fully describe hidden runtime assembly that may exist outside the visible prompt.

That distinction matters because some modes may affect hidden orchestration even when they do not expose a visible marker in the user message.

## Shared Elements Across Modes

Several message-local injected blocks appeared in more than one mode:

- `<current_datetime>` before the prose text
- sometimes `<tools_changed_notice>` before the prose text
- one or more trailing `<reminder>` blocks after the prose text

The reminder blocks in this session were operational, not mode-specific. They described SQL/session state such as:

```xml
<reminder>
<sql_tables>Available tables: todos, todo_deps</sql_tables>
</reminder>
```

and:

```xml
<reminder>
<todo_status>
Todos: 2 done (2 total)
Use sql tool to query ready todos and update status as you work.
</todo_status>
</reminder>
```

No reminder block observed in this session explicitly said "you are in autopilot mode" or "you are in plan mode."

## Regular Mode

## Visible Message Structure

In regular mode, user turns could include:

- `<current_datetime>` before the prose text
- optionally `<tools_changed_notice>` before the prose text
- trailing `<reminder>` blocks after the prose text

There was **no visible regular-mode marker** analogous to `[[PLAN]]`.

## Behavior

Regular mode behaved like the default interactive session:

- answer the user's question directly
- use tools when needed
- no special plan-file workflow
- no visible completion-enforcement follow-up message

In other words, regular mode looked like the baseline conversational state against which the other two modes differed.

## Plan Mode

## Visible Message Structure

Plan mode introduced a clear visible marker in the user message:

```text
[[PLAN]]
```

In the observed plan-mode turn, the visible content before the prose request was:

1. `<current_datetime>`
2. `<tools_changed_notice>`
3. the literal `[[PLAN]]` marker

The message also had a trailing `<reminder>` block after the prose.

This made plan mode the easiest of the three modes to identify from the visible message alone.

## Behavior

Plan mode changed the required workflow substantially:

- analyze the current state
- ask clarifying questions if needed
- create a structured plan
- save that plan to the session plan file
- reflect todos into SQL
- avoid implementation unless explicitly instructed

In this session, plan mode also surfaced a dedicated tool:

- `exit_plan_mode`

That tool accepted a summary of the plan and a recommended next action. The runtime then either approved exit from plan mode or rejected it with feedback and required the plan to be updated.

### Important Behavioral Difference

Plan mode was not just a visible message prefix. It activated a distinct control loop:

1. create or update `plan.md`
2. track tasks in SQL
3. call `exit_plan_mode`
4. handle approval/rejection feedback

This made plan mode both:

- **visibly marked** in the message (`[[PLAN]]`)
- **behaviorally distinct** in tooling and workflow

## Autopilot Mode

## Visible Message Structure

Autopilot mode was more subtle.

In the first observed autopilot turn, the visible prefixes before the prose text were:

1. `<current_datetime>`
2. `<tools_changed_notice>`

The trailing `<reminder>` blocks still appeared after the prose text.

Unlike plan mode, there was **no visible `[[AUTOPILOT]]`-style marker** in the user message.

In a later autopilot turn, the only visible prefix before the prose text was:

1. `<current_datetime>`

Again, there was no explicit autopilot tag in the visible message content.

## Behavior

Although autopilot mode did **not** expose a strong visible message marker, it did produce observable behavioral changes.

### 1. Completion Enforcement

After answering a question in autopilot mode, a follow-up user message appeared instructing the agent that the task was not complete and that it should keep working until it could call `task_complete`.

The message included language to the effect of:

- you have not yet marked the task as complete
- stop planning and start implementing
- keep working autonomously until the task is truly finished

This is the clearest observed behavioral signature of autopilot mode in this session.

### 2. Tool Availability Change

During autopilot mode, the runtime exposed:

- `task_complete`

This tool was then used to formally mark the task complete. Later, a `tools_changed_notice` reported that `task_complete` was no longer available.

This created a runtime pattern that looked like:

1. autopilot task begins
2. `task_complete` becomes available
3. agent is expected to continue autonomously
4. agent calls `task_complete`
5. tool later disappears again

### 3. No Visible Autopilot Reminder

Despite the behavioral changes, the visible reminder blocks remained focused on SQL/todo state. They did **not** mention autopilot control flow.

So the observed autopilot signature was mainly:

- **behavioral**
- **tooling-related**
- **follow-up-message-driven**

not a stable visible prefix in the original user message.

## Side-by-Side Comparison

| Mode | Visible prefix marker | Trailing reminders | Special tooling | Distinct runtime behavior |
|---|---|---|---|---|
| Regular | No explicit mode marker | Yes | None observed | Normal interactive behavior |
| Plan | `[[PLAN]]` | Yes | `exit_plan_mode` | Plan file workflow, SQL todos, approval/revision loop |
| Autopilot | No explicit visible mode marker observed | Yes | `task_complete` | Completion-enforcement follow-up and autonomous-continue behavior |

## Most Important Practical Difference

The key distinction is:

- **Plan mode** changed both the **visible message structure** and the **workflow**
- **Autopilot mode** changed the **workflow and completion behavior**, but did **not** reveal a consistent explicit visible marker in the original user message
- **Regular mode** lacked both of those specialized control mechanisms

## Concrete Takeaways

1. **`[[PLAN]]` is a real visible mode prefix.** It is part of the user message and can be directly inspected.

2. **Autopilot mode was mostly inferred from behavior, not from a message prefix.** The strongest visible evidence came from the later completion-enforcement message and the temporary presence of `task_complete`.

3. **Reminder blocks were orthogonal to mode.** In this session, they tracked SQL state and todo state rather than mode identity.

4. **Tool changes are part of the observable mode story.** `exit_plan_mode` and `task_complete` were concrete signs that the runtime had switched the session into a different behavioral regime.

## Open Question

It remains unknown whether autopilot mode injects additional hidden prompt content that was simply not exposed in the visible message. The observed evidence supports a strong behavioral distinction, but only a weak visible-prefix distinction.
