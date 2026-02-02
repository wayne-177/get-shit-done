# Roadmap: GSD Fork v1.1 — Upstream Port & Autonomy Fixes

## Overview

This milestone ports high-value upstream v1.11.1 changes and fixes user autonomy issues in 3 phases. Phase 1 delivers the critical CONTEXT.md flow and display fixes that are the core value of this milestone. Phase 2 makes the framework respect user preferences through configurable guardrails. Phase 3 adds git branching strategy support.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

- [ ] **Phase 1: Context Flow & Display** - CONTEXT.md reaches all downstream agents; context bar displays correctly
- [ ] **Phase 2: User Autonomy** - Framework guardrails become configurable, user overrides respected
- [ ] **Phase 3: Git Branching** - Branch strategy configurable and executed per phase/milestone

## Phase Details

### Phase 1: Context Flow & Display
**Goal**: User decisions from discuss-phase propagate through every downstream agent and the context bar accurately reflects usage
**Depends on**: Nothing (first phase)
**Requirements**: CTXF-01, CTXF-02, CTXF-03, CTXF-04, CTXF-05, CTXF-06, CTXF-07, CTXD-01, CTXD-02
**Success Criteria** (what must be TRUE):
  1. User runs discuss-phase, sets decisions, then plan-phase — all spawned agents (researcher, planner, checker) receive those decisions in CONTEXT.md
  2. Plan checker rejects plans that violate user decisions marked as "locked" in CONTEXT.md (Dimension 7 failure)
  3. Revision loop preserves user decisions — re-planning after checker rejection does not lose CONTEXT.md content
  4. Context bar shows 100% (red) when Claude Code hits its 80% context limit, with color thresholds adjusted for scaled display
  5. Both CONTEXT.md and {phase}-CONTEXT.md file naming patterns are detected and loaded
**Plans**: 2 plans

Plans:
- [ ] 01-01-PLAN.md — Port CONTEXT.md flow to plan-phase.md (Step 4 loading + all 4 agent prompts)
- [ ] 01-02-PLAN.md — Add Dimension 7 to plan checker + scale context bar display

### Phase 2: User Autonomy
**Goal**: Users control framework behavior through config toggles rather than being overridden by hardcoded directives
**Depends on**: Phase 1 (config structure patterns established)
**Requirements**: AUTO-01, AUTO-02, AUTO-03, AUTO-04, AUTO-05, AUTO-06
**Success Criteria** (what must be TRUE):
  1. User sets `git.auto_commit: false` in config — executor stages changes and prompts instead of auto-committing
  2. Running `/gsd:settings` includes auto-commit preference as a configurable question
  3. User sets `safety.verify_interfaces: false` — executor skips interface verification without error (and vice versa when enabled)
  4. User sets `safety.verify_requirements: false` — executor skips requirements verification without error (and vice versa when enabled)
  5. Reference doc exists explaining what each toggle does and when a user might want to disable it
**Plans**: TBD

Plans:
- [ ] 02-01: TBD
- [ ] 02-02: TBD

### Phase 3: Git Branching
**Goal**: Users can choose a branching strategy and have GSD create/manage branches automatically during execution
**Depends on**: Phase 2 (settings UI patterns established, config structure in place)
**Requirements**: GITB-01, GITB-02, GITB-03, GITB-04, GITB-05
**Success Criteria** (what must be TRUE):
  1. Running `/gsd:settings` shows branching strategy question with none/phase/milestone options
  2. User selects "phase" strategy — executing a phase creates a `gsd/phase-{N}-{slug}` branch automatically
  3. User selects "milestone" strategy — executing within a milestone creates a `gsd/{version}-{slug}` branch automatically
  4. Squash merge option is available at milestone completion (or explicitly documented as not-yet-implemented if upstream has no implementation)
**Plans**: TBD

Plans:
- [ ] 03-01: TBD
- [ ] 03-02: TBD

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Context Flow & Display | 0/TBD | Not started | - |
| 2. User Autonomy | 0/TBD | Not started | - |
| 3. Git Branching | 0/TBD | Not started | - |
