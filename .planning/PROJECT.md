# GSD Fork — Upstream Port & Autonomy Fixes

## What This Is

A local fork of glittercowboy/get-shit-done (based on v1.9.13) used as the primary development workflow tool. This project ports selected upstream v1.11.1 improvements and addresses user autonomy issues where the framework overrides user direction.

## Core Value

User decisions from `/gsd:discuss-phase` must reach all downstream agents and be respected — the framework serves the user, not the other way around.

## Requirements

### Validated

<!-- Shipped and confirmed valuable. -->

- Existing GSD workflow: new-project, plan-phase, execute-phase, verify-work
- Requirements verification in planning and execution
- Interface verification in planning and execution
- Codebase mapping via `/gsd:map-codebase`
- Status line hook with context bar display
- All 35 slash commands functional
- 11 subagent specifications working

### Active

<!-- Current scope. Building toward these. -->

- [ ] Port context bar scaling fix (show 100% at 80% limit)
- [ ] Port CONTEXT.md flow fix (discuss-phase decisions reach all agents)
- [ ] Port context compliance verification (Dimension 7 in plan checker)
- [ ] Port context file detection improvement
- [ ] Make auto-commit configurable via config toggle
- [ ] Soften NON-NEGOTIABLE directives to STRONGLY RECOMMENDED with config toggles
- [ ] Add user-override mechanism for workflow guardrails
- [ ] Port git branching strategy configuration (none/phase/milestone)
- [ ] Port git branching execution logic
- [ ] Port squash merge option (if implementation exists upstream)

### Out of Scope

<!-- Explicit boundaries. Includes reasoning to prevent re-adding. -->

- Gemini CLI support — multi-LLM complexity not needed, fork is Claude Code specific
- Discord integration — community feature for upstream, not relevant
- OpenCode fixes — not in scope for this fork
- `--all` install flag — multi-platform installer, not needed
- CI/CD changes — infrastructure, separate release process
- `/gsd:whats-new` removal — already N/A post-v1.9.12
- Discord badge — documentation only

## Current Milestone: v1.1 Upstream Port

**Goal:** Port high-value upstream v1.11.1 changes and fix user autonomy issues in the GSD fork.

**Target features:**
- CONTEXT.md full flow (discuss-phase → researcher → planner → checker → revision)
- Context compliance verification (Dimension 7)
- Context bar UX fix
- Configurable auto-commit and safety guardrails
- Git branching strategy support

## Context

- Fork is based on v1.9.13 (doesn't exist as official upstream release — custom version)
- Our custom additions (requirements/interface verification) are in separate files from upstream changes — zero conflict expected
- Upstream uses tilde paths (`~/.claude/...`), we use absolute paths (`/Users/macuser/.claude/...`) — cosmetic difference
- 15 upstream changes analyzed: 4 clean ports, 3 merge required, 0 conflicts, 8 skipped
- Research completed 2026-02-01 across 5 agents (changelog, diff, portability, methodology, autonomy audit)
- **Future milestone idea:** Codify agent methodology learnings (from `prompt_studio/src/features/research/deo/METHODOLOGY_AND_SOP.md`) into GSD platform — task-aware model selection, incremental disk writes, sequential agent support, adversarial validation, token tracking. Decide whether this belongs in GSD (markdown prompts) or DEO (programmatic control).

## Constraints

- **Compatibility**: All changes must preserve existing requirements/interface verification workflows
- **Incremental**: Port Priority 1 first, use for several days, then proceed to Priority 2
- **No regressions**: Existing 35 commands and 11 agents must continue working
- **Atomic commits**: Each logical change gets its own commit

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Skip Gemini/OpenCode/Discord | Fork is Claude Code specific, complexity not justified | — Pending |
| 3-phase approach (critical → autonomy → branching) | Risk-ordered, each phase standalone testable | — Pending |
| Soften NON-NEGOTIABLE to configurable | User autonomy audit finding 1.2 — framework shouldn't override user | — Pending |
| Research git branching implementation before porting | Files identical local/upstream, need to find where config is consumed | — Pending |

---
*Last updated: 2026-02-01 after milestone v1.1 initialization*
