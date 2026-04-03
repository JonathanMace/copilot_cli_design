# Copilot CLI Design

Design notes and research on GitHub Copilot CLI — agent architecture, subagent communication, context management, and workflow patterns.

## Self-Updating Instructions

These instructions are your persistent memory across sessions. When you discover a useful design pattern, workflow improvement, or lesson learned during work, add it to this file immediately. If it's not written here, you will forget it.

When you discover an anti-pattern to avoid, add it to this file immediately.

This rule applies to itself: if you discover a better way to organize these instructions, update the structure.

## Work Tracking

Any subagents working on a dedicated coding task (i.e., working on an issue or creating a PR) must work in a dedicated git worktree. Follow the `git-checkpoint` skill for the full branch → worktree → commit → PR → merge → cleanup lifecycle.

When work is complete:
- Push and merge your own work via PR
- Delete the remote branch after merge
- Delete the local branch once merged
- Remove the worktree

## Subagent Model Specification

When spawning subagents, you **must** always specify the model explicitly. Never leave the model unspecified.

Default models (unless instructed otherwise):
- **Coding tasks**: `claude-opus-4.6`
- **Writing tasks**: `gpt-5.4`

This rule applies recursively — include it in subagent prompts so they also specify models when spawning their own subagents.

## README Maintenance

Always keep the README up to date. When creating new reports, documents, or significant artifacts, check whether the README needs to be updated to reflect the changes.

## Anti-Patterns

<!-- Add anti-patterns here as you discover them. Include brief context so future agents understand the rationale. -->
