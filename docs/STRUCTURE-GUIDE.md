# GSD Repository Structure Guide

## Overview

The repository has a slightly confusing naming convention where the repo itself is called `get-shit-done` and it contains a subdirectory also called `get-shit-done`. This is intentional - the subdirectory contains the "runtime" files that get installed.

## Directory Layout

```
~/code/get-shit-done/                    ← Git repository root
├── .git/                                ← Git metadata
├── .github/                             ← GitHub workflows, PR templates
│
├── agents/                              ← Agent definitions
│   ├── gsd-codebase-mapper.md          (11 agents total)
│   ├── gsd-debugger.md
│   ├── gsd-executor.md
│   └── ...
│
├── commands/                            ← Skill/command definitions
│   └── gsd/                            (35 commands)
│       ├── help.md
│       ├── new-project.md
│       ├── suggest.md                  ← Your agentic addition
│       └── ...
│
├── get-shit-done/                       ← Runtime content (the core)
│   ├── lib/                            ← Failure system libraries (your addition)
│   ├── references/                     ← Reference documentation
│   ├── templates/                      ← Template files for generated docs
│   ├── tests/                          ← Verification workflows (your addition)
│   ├── workflow-templates/             ← Intent matching (your addition)
│   └── workflows/                      ← Workflow definitions (42 total)
│
├── bin/                                 ← Installer
│   └── install.js                      ← Run this to install
│
├── hooks/                               ← Claude Code hooks
│   ├── gsd-check-update.js
│   └── gsd-statusline.js
│
├── scripts/                             ← Build scripts
│   └── build-hooks.js
│
├── assets/                              ← Images, logos
├── docs/                                ← Documentation (your addition)
│
├── README.md                            ← Main documentation
├── CHANGELOG.md                         ← Version history
├── CONTRIBUTING.md                      ← Contribution guidelines
├── GSD-STYLE.md                         ← Code style guide
├── LICENSE                              ← MIT license
├── package.json                         ← npm package definition
└── package-lock.json
```

## Installation Mapping

When you run `node bin/install.js` (or `npx get-shit-done-cc@latest`):

| Source (repo) | Destination (installed) |
|---------------|------------------------|
| `agents/` | `~/.claude/agents/` |
| `commands/gsd/` | `~/.claude/commands/gsd/` |
| `get-shit-done/` | `~/.claude/get-shit-done/` |
| `hooks/` | `~/.claude/hooks/` (bundled) |

## Why the Nested get-shit-done/?

The inner `get-shit-done/` directory contains only the "runtime" files that Claude Code needs:
- workflows (the actual logic)
- templates (document templates)
- references (reference docs)

The outer repo contains:
- Source code (`bin/`, `scripts/`)
- Development files (`package.json`)
- Documentation (`README.md`, etc.)
- Agents and commands (which install to different locations)

This separation keeps the installed `~/.claude/get-shit-done/` clean with only runtime files.

## Your Agentic Additions

Files you added (not in upstream):

```
get-shit-done/
├── lib/                    ← Failure analysis libraries
├── tests/                  ← Verification test workflows
├── workflow-templates/     ← Intent matching infrastructure
├── workflows/
│   ├── adaptive-failure-analysis.md
│   ├── predict-failures.md
│   ├── retry-orchestration.md
│   ├── discover-monorepo.md
│   ├── understand-goal.md
│   └── ... (30 total unique workflows)
├── templates/
│   ├── alternative-paths.md
│   ├── health-status.md
│   └── ... (13 unique templates)
└── references/
    ├── config-schema.md
    ├── knowledge-base-api.md
    └── ... (10 unique references)

commands/gsd/
├── analyze-codebase.md
├── suggest.md
├── understand.md
└── ... (8 unique commands)
```

## Making Changes

1. **Edit files in** `~/code/get-shit-done/`
2. **Re-install** with `node bin/install.js` OR manually copy changed files
3. **Restart Claude Code** to pick up agent/command changes
4. **Commit & push** to preserve changes

## Syncing with Upstream

```bash
cd ~/code/get-shit-done
git fetch upstream
git merge upstream/main
# Resolve conflicts
git push origin main
node bin/install.js  # Re-install
```

---

*Created: 2026-01-24*
