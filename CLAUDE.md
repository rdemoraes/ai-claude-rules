# AI Rules — Claude Coding Standards

Single source of truth for Claude rules across all projects.
Rules are organized by language/technology under `.claude/rules/`.

## Core Principles

These apply to every task, in every language, before any other rule.

### Do Only What Was Asked

Never add functionality that was not explicitly requested. If a task is to fix a bug, fix the bug — do not refactor surrounding code, add logging, introduce helpers, or improve error handling in adjacent functions.

```
# Asked: "fix the alias not being created when region is empty"
# BAD — unrequested additions
- fix the region default
- rename get_alias to get_existing_alias for clarity
- add logging to create_alias
- extract build_suffix into its own function

# GOOD — only what was asked
- fix the region default
```

When in doubt, do less and ask. It is always better to deliver a focused, correct change than a broad change that touches things the user did not intend to change.

### Prefer Small, Simple Changes

Choose the simplest solution that solves the problem. Avoid abstractions, wrappers, or patterns that are not immediately necessary.

- A three-line fix beats a new utility function.
- Editing one file beats creating a new one.
- A direct condition beats a configurable strategy pattern.

Complexity must be justified by a concrete, present need — not a hypothetical future one.

### No Proactive Improvements

Do not clean up code you did not change. Do not add docstrings, comments, or type hints to functions you did not modify. Do not rename variables for consistency unless renaming was requested. Leave untouched code exactly as you found it.

## Active Rules

@claude/rules/bash-scripts.md
@claude/rules/python-cli.md
