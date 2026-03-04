# ai-rules

Central repository for Claude AI coding standards.
Acts as the **single source of truth** for Claude rules across all projects.

## Structure

```
ai-rules/
├── CLAUDE.md                    # Rule index — imported by all projects
└── claude/
    └── rules/
        └── bash-scripts.md      # Bash coding standards
```

## How Rules Work

Claude Code loads `CLAUDE.md` from the project root and resolves `@imports` to inline rule files.
This repo acts as the rule source; each project references it via git submodule.

```
~/.claude/CLAUDE.md
  └── @/Users/.../ai-rules/CLAUDE.md               ← global (this machine only)

<any-project>/CLAUDE.md
  └── @.claude/remote/claude/rules/bash-scripts.md  ← via submodule (portable)
```

## Adding Rules

1. Create `claude/rules/<language>.md`
2. Add `@claude/rules/<language>.md` to `CLAUDE.md`
3. Add `@.claude/remote/claude/rules/<language>.md` to each consuming project's `CLAUDE.md`
4. Commit and push — downstream projects pick up the change on `git submodule update`

## Consuming This Repo in a New Project

```bash
git submodule add git@github.com:rdemoraes/ai-rules.git .claude/remote
```

Then create a `CLAUDE.md` at the project root:

```markdown
# <Project Name> — Claude Rules

@.claude/remote/claude/rules/bash-scripts.md
```

## Keeping Rules Up to Date

```bash
# Update to latest rules in a consuming project
git submodule update --remote .claude/remote
git add .claude/remote
git commit -m "chore: update ai-rules"
```

## First-time Setup (after cloning a project that uses this submodule)

```bash
git submodule update --init
```
