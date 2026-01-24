# Update Warning: Do NOT Use /gsd:update

**This fork contains agentic additions merged from wayne-177/get-shit-done-agentic.**

## The Problem

The `/gsd:update` command runs `npx get-shit-done-cc@latest` which pulls from the **npm registry** (glittercowboy's published package). This would **overwrite** your `~/.claude/get-shit-done/` with upstream, **wiping your agentic additions**.

## Safe Update Procedure

Instead of `/gsd:update`, manually sync with upstream:

```bash
# 1. Pull upstream changes
cd ~/code/get-shit-done
git fetch upstream
git merge upstream/main

# 2. Resolve any conflicts (especially in execute-phase.md)

# 3. Push to your fork
git push origin main

# 4. Re-install from your fork
node bin/install.js
# OR manually copy:
cp -r get-shit-done/* ~/.claude/get-shit-done/
cp -r agents/* ~/.claude/agents/
cp -r commands/gsd/* ~/.claude/commands/gsd/
```

## What Would Be Lost

If you run `/gsd:update`, these agentic additions would be overwritten:

### Workflows (30)
- Failure Recovery: adaptive-failure-analysis, adaptive-path-selection, auto-correction, predict-failures, retry-orchestration
- Codebase Analysis: analyze-codebase, analyze-dependencies, analyze-security, analyze-tech-debt, impact-analysis
- Knowledge Base: init-knowledge-base, extract-patterns, import-patterns, decay-patterns, analyze-patterns
- Goal Refinement: understand-goal, generate-suggestions, route-intent
- Monorepo: discover-monorepo, coordinated-execution
- Other: health-check, create-roadmap, create-milestone, generate-workflow, update-claude-md, variable-extractor, intent-matcher, discuss-milestone, plan-phase, research-phase

### Commands (8)
- analyze-codebase, consider-issues, create-roadmap, discuss-milestone
- execute-plan, health-check, suggest, understand

### Enhanced execute-phase.md
- Failure prediction integration
- Knowledge base hints
- Adaptive retry
- Regression checking

### Supporting Infrastructure
- lib/ (failure system libraries)
- tests/ (verification workflows)
- workflow-templates/ (intent matching)

## Remotes Reference

```
origin   = wayne-177/get-shit-done      (your fork - commits go here)
upstream = glittercowboy/get-shit-done  (source - pull updates from here)
```

---

*Created: 2026-01-24*
*After merge of upstream v1.9.13 + agentic additions*
