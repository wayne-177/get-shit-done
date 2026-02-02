# Requirements: GSD Fork v1.1 — Upstream Port & Autonomy Fixes

**Defined:** 2026-02-01
**Core Value:** User decisions from discuss-phase must reach all downstream agents and be respected

## v1 Requirements

Requirements for milestone v1.1. Each maps to roadmap phases.

### Context Flow

- [x] **CTXF-01**: CONTEXT.md from discuss-phase loads in plan-phase Step 4 before agent spawning
- [x] **CTXF-02**: Phase researcher receives CONTEXT.md with decision/discretion/deferred instructions
- [x] **CTXF-03**: Planner receives CONTEXT.md with locked/discretion/out-of-scope classification
- [x] **CTXF-04**: Plan checker receives CONTEXT.md for compliance validation
- [x] **CTXF-05**: Revision loop receives CONTEXT.md to preserve user decisions during revisions
- [x] **CTXF-06**: Plan checker validates plans against user decisions (Dimension 7: Context Compliance)
- [x] **CTXF-07**: Context file detection handles both `CONTEXT.md` and `{phase}-CONTEXT.md` patterns

### Context Display

- [x] **CTXD-01**: Context bar shows 100% when hitting Claude Code's 80% limit (scaled display)
- [x] **CTXD-02**: Color thresholds adjusted for scaled display (green < 63%, yellow < 81%, orange < 95%, red >= 95%)

### User Autonomy

- [x] **AUTO-01**: `git.auto_commit` config toggle controls whether GSD auto-commits or stages-and-prompts
- [x] **AUTO-02**: `/gsd:settings` includes auto-commit preference question
- [x] **AUTO-03**: Executor respects `safety.verify_interfaces` config toggle (STRONGLY RECOMMENDED when enabled, skippable when disabled)
- [x] **AUTO-04**: Executor respects `safety.verify_requirements` config toggle (STRONGLY RECOMMENDED when enabled, skippable when disabled)
- [x] **AUTO-05**: `safety` section in config.json with documented toggleable guardrails
- [x] **AUTO-06**: Reference doc explains override capabilities and when to use them

### Git Branching

- [ ] **GITB-01**: `/gsd:settings` shows branching strategy question (none/phase/milestone)
- [ ] **GITB-02**: Config stores `git.branching_strategy` value
- [ ] **GITB-03**: Phase execution creates branch `gsd/phase-{N}-{slug}` when strategy is "phase"
- [ ] **GITB-04**: Milestone execution creates branch `gsd/{version}-{slug}` when strategy is "milestone"
- [ ] **GITB-05**: Squash merge option available at milestone completion (if upstream implementation found)

## v2 Requirements

Deferred to future milestone. Tracked but not in current roadmap.

### Installer

- **INST-01**: `--uninstall` flag for clean GSD removal

## Out of Scope

Explicitly excluded. Documented to prevent scope creep.

| Feature | Reason |
|---------|--------|
| Gemini CLI support | Multi-LLM complexity not needed, fork is Claude Code specific |
| Discord integration | Community feature for upstream, not relevant to fork |
| OpenCode fixes | Not in scope for this fork |
| `--all` install flag | Multi-platform installer, not needed |
| CI/CD changes | Infrastructure, separate release process |
| `/gsd:whats-new` removal | Already N/A post-v1.9.12 |
| Discord badge | Documentation only |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| CTXF-01 | Phase 1 | Complete |
| CTXF-02 | Phase 1 | Complete |
| CTXF-03 | Phase 1 | Complete |
| CTXF-04 | Phase 1 | Complete |
| CTXF-05 | Phase 1 | Complete |
| CTXF-06 | Phase 1 | Complete |
| CTXF-07 | Phase 1 | Complete |
| CTXD-01 | Phase 1 | Complete |
| CTXD-02 | Phase 1 | Complete |
| AUTO-01 | Phase 2 | Complete |
| AUTO-02 | Phase 2 | Complete |
| AUTO-03 | Phase 2 | Complete |
| AUTO-04 | Phase 2 | Complete |
| AUTO-05 | Phase 2 | Complete |
| AUTO-06 | Phase 2 | Complete |
| GITB-01 | Phase 3 | Pending |
| GITB-02 | Phase 3 | Pending |
| GITB-03 | Phase 3 | Pending |
| GITB-04 | Phase 3 | Pending |
| GITB-05 | Phase 3 | Pending |

**Coverage:**
- v1 requirements: 20 total
- Mapped to phases: 20
- Unmapped: 0

---
*Requirements defined: 2026-02-01*
*Last updated: 2026-02-01 after roadmap creation*
