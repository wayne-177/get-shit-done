# GSD Development Methodology Assessment

**Author:** Senior Software Engineering Analyst
**Date:** 2026-02-01
**Subject:** glittercowboy/get-shit-done (upstream)
**Repository Stats:** 10,430 stars, 1,017 forks, 203 open issues, created 2025-12-14
**Version Range Analyzed:** v1.0.0 (2025-12-14) through v1.11.1 (2026-01-31)
**Data Sources:** CHANGELOG.md (131+ versions), 200+ commits, 377+ issues/PRs, CONTRIBUTING.md, GSD-STYLE.md, MAINTAINERS.md, README.md

---

## 1. Development Methodology

### 1.1 Methodology Classification: "Accelerated Dogfooding Iteration"

glittercowboy follows a methodology best described as **rapid prototyping with self-use feedback loops**. It is not Agile in any formal sense -- there are no sprints, no story points, no retrospectives. The CONTRIBUTING.md explicitly bans these practices ("Enterprise Patterns (Banned): Story points, Sprint ceremonies, RACI matrices"). Instead, the pattern is:

1. Build a feature
2. Use it on a real project
3. Discover what breaks or feels wrong
4. Ship a fix within hours

This is evidenced by the sheer density of patch releases. Between v1.5.16 and v1.5.30 (covering roughly January 15-17, 2026), there were **15 releases in approximately 48 hours**. Many are single-issue fixes discovered during real usage:
- v1.5.24: "Stop notification hook now correctly parses STATE.md fields (was always showing 'Ready for input')"
- v1.5.25: "Stop notification hook no longer shows stale project state"
- v1.5.26: "Revised plans now get committed after checker feedback"
- v1.5.27: "Orchestrator corrections between executor completions are now committed"

Each of these represents a bug found during actual use and fixed immediately.

### 1.2 Release Cadence

**Total versions:** ~131 versions in ~48 days (2025-12-14 to 2026-01-31)

**Average:** ~2.7 releases per day.

However, the cadence is extremely uneven:

| Period | Versions | Rate | Character |
|--------|----------|------|-----------|
| Dec 14-15, 2025 | v1.0.0 - v1.2.0 | ~12 in 2 days | Initial rapid prototyping |
| Dec 15-17, 2025 | v1.2.1 - v1.3.8 | ~20 in 3 days | Stabilization + brownfield support |
| Dec 17 - Jan 11, 2026 | v1.3.8 - v1.3.34 | ~26 in 25 days | Slower feature development |
| Jan 12-17, 2026 | v1.4.0 - v1.6.4 | ~40 in 6 days | Major architectural push (parallel execution, subagents) |
| Jan 19-23, 2026 | v1.7.0 - v1.9.12 | ~20 in 5 days | Feature sprint (quick mode, codebase intel, then revert) |
| Jan 24-31, 2026 | v1.10.0 - v1.11.1 | ~4 in 8 days | Slowing cadence, community features |

**Key observation:** The cadence is slowing as the project matures. The CONTRIBUTING.md itself acknowledges the problem: "Current problem: 131 tags in one month. Too noisy." It proposes batching patch releases weekly and tagging only stable milestones. This self-awareness is notable.

**Only 6 formal GitHub Releases** exist despite 131+ changelog entries: v1.9.5, v1.9.6, v1.9.11, v1.10.0, v1.10.1, v1.11.1. This means most versions exist only as npm publishes and git tags, not as curated releases with notes.

### 1.3 Commit Patterns

Commits are predominantly **monolithic feature-plus-docs bundles**. A single commit often includes the feature implementation, changelog update, and README changes. Examples from the commit log:

- `f3db981f` "docs: update changelog and README for v1.11.0" -- documentation bundled with release
- `32571390` "fix(plan-phase): pass CONTEXT.md to all downstream agents" -- single commit for a multi-file feature
- `d1fda80c` "revert: remove codebase intelligence system" -- a single commit reverting an entire feature that spanned multiple prior commits

Community PRs are merged as single squash commits (e.g., `f8fd7104` "Merge pull request #298 from davesienkowski/feature/branching-strategy").

The conventional commit format is followed consistently: `type(scope): description`. This discipline is maintained throughout the entire 200+ commit history.

### 1.4 Planning Evidence

**No formal planning artifacts are visible in the repository.** There is no ROADMAP.md, no milestones created on GitHub, no project boards. The project itself is a planning tool, but its own development appears to be driven by:

1. **Personal use feedback** -- Most fixes address issues the author discovered while using GSD on real projects
2. **Community requests** -- Issues like #348 "Support custom orchestrator rules", #344 "plan all phases in parallel", #329 "Integrate with New Claude Task Manager"
3. **Reactive problem-solving** -- The codebase intelligence system (v1.9.0) was built over ~2 days then reverted in v1.9.2 after discovering it was overengineered (21MB sql.js dependency)

The closest thing to a roadmap is issue #340 "All v2.1 features" (created 2026-01-28), suggesting forward-looking planning is beginning.

### 1.5 Testing Strategy

**No automated tests exist.** There is no test directory, no test runner configured in package.json, no CI/CD pipeline running tests. The CONTRIBUTING.md mentions "Setting Up Development" with `npm test` but the actual testing appears to be manual dogfooding.

The project does have a CI/CD history, but it is unstable:
- v1.9.5: "Added CI/CD automation for releases"
- v1.9.11: "Switched to manual npm publish workflow (removed GitHub Actions CI/CD)"
- v1.9.12: "Restored auto-release GitHub Actions workflow"
- v1.9.13 commit `339e9112`: "chore: remove GitHub Actions release workflow"

This oscillation between automated and manual publishing within 8 days suggests the CI/CD infrastructure is not mature.

**Quality gates are conceptual, not enforced.** The CONTRIBUTING.md describes branch protection rules as "Optional" and lists checks for `test` and `lint` that do not actually exist.

### 1.6 Documentation Approach

Documentation is a **clear strength**. The project maintains:

1. **CHANGELOG.md** -- Follows Keep a Changelog format, comprehensively maintained from v1.0.0 through v1.11.1. Every version has a description. The changelog was retroactively backfilled in v1.4.26 ("Full changelog history backfilled from git -- 66 historical versions").

2. **GSD-STYLE.md** -- A comprehensive 500+ line style guide covering XML conventions, naming patterns, language/tone rules, anti-patterns, and commit conventions. This is unusually thorough for a project of this size.

3. **CONTRIBUTING.md** -- Covers branch strategy, commit conventions, release workflow, PR guidelines, and explicit anti-patterns. The document is self-aware about project problems (e.g., tag noise).

4. **MAINTAINERS.md** -- Quick reference for release workflows, pre-release handling, hotfixes, and dependency policy.

5. **README.md** -- Well-structured with clear workflow explanation, testimonials, command tables, and configuration documentation.

6. **Inline documentation** -- Every agent, command, and workflow file contains its own documentation as structured markdown with XML tags.

The documentation quality is disproportionately high for the project's age and solo-developer nature.

---

## 2. Paradigm Shifts

The project has undergone **four major paradigm shifts** in 48 days, each representing a significant evolution in architectural thinking.

### 2.1 Shift 1: Monolithic Commands to Thin Orchestrator + Subagents (v1.4.0 - v1.5.18)

**When:** January 12-16, 2026
**What changed:** Commands went from containing all execution logic inline to being thin wrappers that spawn specialized subagent processes.

**Before (pre-v1.4.0):** Each command file (plan-phase.md, execute-phase.md) contained the full implementation logic. This meant the main context window accumulated all the work, leading to context rot.

**After (v1.5.7 - v1.5.18):**
- `gsd-executor` subagent created (v1.5.7): "Dedicated agent for plan execution with full workflow logic built-in"
- `gsd-verifier` subagent created (v1.5.7): "Goal-backward verification that checks if phase goals are actually achieved"
- `gsd-researcher` agent created (v1.5.16): 915 lines with 4 research modes
- `gsd-debugger` agent created (v1.5.16): 990 lines with scientific debugging methodology
- `gsd-planner` agent created (v1.5.18): 1,319 lines consolidating all planning expertise
- `gsd-plan-checker` agent created (v1.5.18): 744 lines with 6 verification dimensions

**Evidence of maturation:** The v1.5.18 changelog entry explicitly describes the architectural principle: "plan-phase refactored to thin orchestrator pattern (310 lines) -- Spawns gsd-planner for planning, gsd-plan-checker for verification. User sees status between agent spawns (not a black box)."

This is the single most important architectural shift. It directly addresses the core problem GSD was built to solve (context rot) by keeping the main context clean while delegating heavy work to fresh subagent contexts.

### 2.2 Shift 2: The Codebase Intelligence Experiment and Revert (v1.9.0 - v1.9.2)

**When:** January 20-21, 2026
**What changed:** A sophisticated codebase intelligence system was built, deployed, and reverted within ~36 hours.

**v1.9.0 added:**
- `/gsd:analyze-codebase` command
- SQLite graph database using sql.js
- Entity file generation with semantic analysis
- PostToolUse hooks for automatic indexing
- SessionStart hooks for context injection
- Query interface for graph database

**v1.9.2 reverted everything:**
- "Removed due to overengineering concerns"
- "Deleted `/gsd:analyze-codebase` command"
- "Removed SQLite graph database and sql.js dependency (21MB)"
- "Removed intel hooks"

The commit history shows this feature consumed significant effort: commits `3a829305` through `b3db2ff9` span approximately 20 commits of implementation, planning docs, and execution -- all reverted by a single commit `d1fda80c` "revert: remove codebase intelligence system."

**What this reveals:** This is the most instructive moment in the project's history. The developer built a complex feature using GSD's own methodology (research, plan, execute), discovered it added too much complexity (21MB dependency, multiple hooks, a full database layer), and had the discipline to revert it entirely rather than sunk-cost-fallacy it into the codebase. The MAINTAINERS.md explicitly references this: "The codebase intelligence system was removed partly because sql.js added 21MB."

Later, a simpler alternative emerged: the `gsd-codebase-mapper` agent uses parallel subagents to analyze a codebase and write markdown files -- no database, no hooks, no dependencies. This demonstrates genuine learning.

### 2.3 Shift 3: From Separate Commands to Unified Flows (v1.5.0 - v1.6.0)

**When:** January 14-17, 2026
**What changed:** The project workflow was repeatedly consolidated from many separate commands into fewer unified flows.

**v1.5.0:** The project flow was `new-project -> research-project -> define-requirements -> create-roadmap` -- four separate commands requiring four context windows.

**v1.6.0:** "BREAKING: Unified /gsd:new-milestone flow -- now mirrors /gsd:new-project with questioning -> research -> requirements -> roadmap in a single command." Also removed `/gsd:discuss-milestone`, `/gsd:create-roadmap`, `/gsd:define-requirements`, `/gsd:research-project` as standalone commands.

**v1.5.21:** "Unified /gsd:new-project flow -- Single command now handles questions -> research -> requirements -> roadmap (~10 min)"

This consolidation shows the developer learned that forcing users to invoke separate commands in sequence creates friction. The final flow (new-project as a single orchestrator that handles all stages) is significantly better UX.

### 2.4 Shift 4: Context Engineering as Core Differentiator (v1.5.19 - v1.11.1)

**When:** January 16-31, 2026
**What changed:** CONTEXT.md evolved from a simple notes file to a structured contract that constrains all downstream agents.

**Early (v1.0.9):** "Phase CONTEXT.md loaded in plan-phase command" -- basic context loading, no structure.

**v1.5.19:** "discuss-phase redesigned with intelligent gray area analysis" -- CONTEXT.md gains structured sections (Decisions, Claude's Discretion, Deferred Ideas). "Downstream awareness: discuss-phase now explicitly documents that CONTEXT.md feeds researcher and planner agents."

**v1.5.23-v1.5.25:** Iterative fixes to CONTEXT.md loading ("Phase researcher now loads CONTEXT.md", "Planner agent now reliably loads CONTEXT.md and RESEARCH.md").

**v1.11.1:** The culmination -- "CONTEXT.md from /gsd:discuss-phase now properly flows to ALL downstream agents (researcher, planner, checker, revision loop)" plus a new "Context Compliance" verification dimension that flags plans contradicting user decisions.

This four-version arc shows progressive understanding: first, the author recognized that capturing user intent matters; then learned that capturing it is insufficient -- you must also propagate it to every agent and verify compliance. The v1.11.1 "Dimension 7: Context Compliance" in the plan checker is the mature expression of this insight.

### 2.5 Signs of Learning and Maturation

The trajectory is clearly upward:

1. **Release discipline improving:** From 15 releases in 48 hours to the CONTRIBUTING.md's "New rule: Tag stable milestones, not every commit"
2. **Architectural coherence increasing:** The thin orchestrator pattern is now consistently applied across all commands
3. **Willingness to revert:** The codebase intelligence revert demonstrates engineering judgment over ego
4. **Community integration beginning:** PRs #298 (branching strategy by davesienkowski), #301 (Gemini by community), #232 (OpenCode by dbachelder) show the project is transitioning from solo to collaborative
5. **Self-documentation of problems:** The CONTRIBUTING.md openly acknowledges issues (tag noise, missing CI) rather than pretending they do not exist

---

## 3. What He's Doing Right

### 3.1 The Thin Orchestrator Pattern Is Genuinely Good Architecture

The decision to make commands thin wrappers that spawn specialized subagents is the project's most important architectural choice. It directly solves the core problem (context rot) in an elegant way:

- The main context window stays at 30-40% utilization (per README)
- Each subagent gets a fresh 200k token context for its specialized work
- The orchestrator provides user-facing status updates between agent spawns

**Evidence:** The execute-phase workflow spawns multiple gsd-executor agents in parallel waves, each with their own context. The plan-phase spawns gsd-phase-researcher, then gsd-planner, then gsd-plan-checker in sequence. This is essentially a microservices pattern applied to AI agent orchestration.

### 3.2 Plans as Prompts: XML-Structured Task Definitions

The insight that PLAN.md files are not documentation but executable prompts for Claude is powerful. The XML task structure:

```xml
<task type="auto">
  <name>Create login endpoint</name>
  <files>src/app/api/auth/login/route.ts</files>
  <action>Use jose for JWT. Validate credentials. Return httpOnly cookie.</action>
  <verify>curl -X POST localhost:3000/api/auth/login returns 200 + Set-Cookie</verify>
  <done>Valid credentials return cookie, invalid return 401</done>
</task>
```

This gives each task clear scope, explicit file targets, verification criteria, and done conditions. The `<verify>` tag is particularly valuable -- it forces every task to have a concrete testability criterion before execution begins.

### 3.3 Atomic Git Commits Per Task

The per-task commit strategy (documented in `git-integration.md` as "Commit outcomes, not process") creates excellent observability:

- Each task gets its own commit immediately after completion
- Git bisect can find the exact failing task
- Each task is independently revertable
- Clean history serves as context for future Claude sessions

**Evidence:** The commit format `{type}({phase}-{plan}): {description}` is consistently applied, e.g., `feat(04-05): wire codebase intel injection into planner`.

### 3.4 Aggressive Context Size Management

The quality degradation curve model (0-30% peak, 30-50% good, 50-70% degrading, 70%+ poor) is explicitly documented and drives architectural decisions:

- Plans are limited to 2-3 tasks maximum
- Split triggers are documented: ">3 tasks, multiple subsystems, >5 files per task"
- The statusline hook shows real-time context usage with color-coded warnings

This is one of the few projects that treats AI context management as a first-class engineering concern rather than an afterthought.

### 3.5 The Revert Discipline

The codebase intelligence revert (v1.9.2) demonstrates a critical engineering virtue: the willingness to throw away work that does not meet the bar. The MAINTAINERS.md codifies this into policy with pre-release tags: "If it doesn't work out: Delete pre-release tags, no messy public revert on main."

### 3.6 Documentation as Code

The project treats its documentation files as executable specifications. GSD-STYLE.md is not just a style guide -- it is a prompt for future Claude instances to maintain consistency. The explicit bans (temporal language, enterprise patterns, sycophancy, filler words) function as guardrails for AI-generated contributions.

### 3.7 Strong Naming and UX Conventions

- Command naming follows a consistent `gsd:{verb}-{noun}` pattern
- The "Next Up" format gives users copy-paste-ready commands after each stage
- AskUserQuestion is mandated for all exploration (never plain text prompts)
- The `--skip-research` and `--skip-verify` flags respect power users

### 3.8 Community Awareness and Platform Expansion

The project has expanded from Claude Code-only to supporting OpenCode and Gemini CLI, with community contributors driving the ports. The installer now handles all three runtimes with a single `npx` command. The CONTRIBUTING.md includes a "Community Ports" acknowledgment section, and the branching strategy feature (#298) was a community PR that was merged.

### 3.9 The Discussion Phase Concept

The `/gsd:discuss-phase` command -- where users capture implementation preferences before research or planning begins -- is a genuinely novel contribution to AI-assisted development workflows. It solves the problem of AI making assumptions about visual layout, interaction patterns, or technical choices that the human has strong opinions about. The structured CONTEXT.md output (Decisions / Claude's Discretion / Deferred Ideas) provides clear contracts for downstream agents.

---

## 4. What He's Doing Wrong (or Suboptimally)

### 4.1 Zero Automated Testing

This is the single largest deficiency. A project with 131 versions, 10,000+ stars, and 1,000+ forks has no automated tests of any kind. No unit tests for the installer (bin/install.js). No integration tests for the hook scripts. No validation that agent markdown files parse correctly. No smoke tests for the `npx` installation flow.

**Evidence:** No `test/` or `__tests__/` directory exists. The CONTRIBUTING.md mentions `npm test` in setup instructions, but no test script is defined. The CI/CD configuration has oscillated between automated and manual publishing 4 times in 8 days (v1.9.5 through v1.9.13), suggesting even the release pipeline is manually verified.

**Impact:** Issues like #330 "Update breaks status line - settings.json reference not updated", #355 "gsd-executor not found?", and #339 "fix: remove unresolved @-references from plan templates and workflows" are all regressions that automated tests would catch. The installer has had repeated platform-specific bugs (Windows paths in v1.9.5, WSL2/non-TTY in v1.6.4, OpenCode config paths in v1.9.7) that scream for cross-platform test coverage.

### 4.2 Unsustainable Version Numbering

131 versions in 48 days is not a versioning strategy -- it is a compulsive tag habit. The patch version reached `.30` within v1.5.x (v1.5.30), which violates the spirit of semantic versioning. The project's own CONTRIBUTING.md acknowledges this: "Current problem: 131 tags in one month. Too noisy."

**The proposed fix (batch patches weekly) has not been implemented.** Versions v1.9.7 through v1.9.12 were released within a single day (2026-01-23), with many existing only as changelog entries without corresponding GitHub Releases.

**Impact:** Downstream consumers (like our fork) cannot track meaningful changes because the signal-to-noise ratio in versions is extremely low. The changelog becomes a wall of text rather than a useful upgrade guide.

### 4.3 The "Ship Then Fix" Anti-Pattern

The project repeatedly ships features with known gaps, then patches them over multiple subsequent releases:

**CONTEXT.md flow example:**
- v1.0.9: Basic CONTEXT.md loading added
- v1.5.23: "Phase researcher now loads CONTEXT.md" (was not loading it before?)
- v1.5.24: "Planner agent now reliably loads CONTEXT.md and RESEARCH.md files" (reliability fix)
- v1.5.25: "Researcher agent now reliably loads CONTEXT.md from discuss-phase" (another reliability fix)
- v1.11.1: "CONTEXT.md now properly flows to ALL downstream agents" (still not working correctly 6 versions of fixes later)

**Notification hook example:**
- v1.5.23: Cross-platform completion notification hook added
- v1.5.24: "Stop notification hook now correctly parses STATE.md fields"
- v1.5.25: "Stop notification hook no longer shows stale project state"
- v1.5.29: "Removed blocking notification popups (gsd-notify) on all platforms"

The notification feature was added, patched twice, and then removed entirely within 6 versions spanning ~24 hours.

### 4.4 No Branch Protection or Review Process

Despite the CONTRIBUTING.md prescribing "PRs required" and "No direct commits" to main, the commit history shows the vast majority of commits are direct pushes to main by glittercowboy. Only a handful of community PRs (#298, #301, #232, #233, #347) have gone through the PR process.

**Evidence:** Of the ~200 commits visible in the history, fewer than 10 are merge commits from PRs. The rest are direct pushes.

### 4.5 Inconsistent Version Metadata

Several changelog entries have incorrect dates (likely typos):
- v1.10.1 is dated "2025-01-30" but was published "2026-01-30T13:08:17Z"
- v1.9.12 is dated "2025-01-23" but context suggests it is from 2026
- v1.9.8 is dated "2025-01-22" but context suggests it is from 2026

These date inconsistencies make it harder to reconstruct the actual development timeline.

### 4.6 Growing Issue Backlog Without Triage

The repository has 203 open issues as of 2026-02-01. Many are feature requests from the community (#344 "plan all phases in parallel", #348 "Support custom orchestrator rules", #329 "Integrate with New Claude Task Manager") with no labels, milestones, or prioritization. The issues list also contains duplicates and PRs mixed with bug reports.

**Evidence from the issues sample:**
- #369 "Doesn't follow the instructions / project conventions" -- fundamental usability issue, open
- #343 "gsd-phase-researcher doesn't persist RESEARCH.md to disk" -- data loss bug, open
- #351 "Overly optimistic commits" -- core reliability issue, open
- #366 "fix(parallel): prevent race conditions in parallel subagent execution" -- critical architecture bug, open

Several of these are critical bugs that affect core functionality, yet they sit alongside feature requests for Cursor CLI support (#342) and note-taking (#349) with no visible prioritization.

### 4.7 The Recommend-Dangerous-Flags Pattern

The README prominently recommends `claude --dangerously-skip-permissions` with the justification: "stopping to approve `date` and `git commit` 50 times defeats the purpose." While this is pragmatic for the author's workflow, recommending this to 10,000+ users without adequate security context is risky. The alternative (granular permissions) is buried in a `<details>` collapse.

### 4.8 Tilde Path Fragility

The entire system relies on `~/.claude/` tilde expansion for file paths. The README acknowledges this causes problems: "If file reads fail with tilde paths (~/.claude/...), set CLAUDE_CONFIG_DIR." This is a known fragility in containerized environments, CI/CD pipelines, and Windows. A more robust path resolution system would prevent recurring platform-specific bugs.

---

## 5. What's Indifferent (Neutral Choices)

### 5.1 Markdown-as-Configuration

GSD uses markdown files as both configuration and executable specifications. Agent definitions, workflows, and commands are all `.md` files with YAML frontmatter and XML-tagged sections. This is neither clearly good nor bad:

**Advantages:** Human-readable, version-controllable, directly editable by Claude, no compilation step.
**Disadvantages:** No schema validation, no syntax checking, easy to introduce subtle errors (as seen with unresolved @-references in issue #339).

Alternative approaches (JSON schemas, YAML configs, TypeScript definitions) would provide validation but lose the "plans as prompts" property. This is a genuine tradeoff with no clear winner.

### 5.2 npm as Distribution Channel

Distributing via `npx get-shit-done-cc` is a pragmatic choice. npm is ubiquitous and the zero-install `npx` pattern works well. Alternatives (Homebrew, pip, binary releases) would each have tradeoffs. The npm approach does create a JavaScript dependency for what is essentially a collection of markdown files, but the installer logic (bin/install.js) genuinely needs a runtime.

### 5.3 The Anti-Enterprise Branding

The "No enterprise roleplay bullshit" positioning in the README, the banning of story points and sprint ceremonies in CONTRIBUTING.md, and the overall irreverent tone are stylistic choices. They resonate with the solo-developer target audience (10,000+ stars suggest market fit) but may alienate teams or organizations evaluating the tool.

This is pure positioning, not engineering. Neither correct nor incorrect.

### 5.4 Conventional Commits

Using conventional commits (`feat(scope): description`) is a standard industry practice. It is neither innovative nor problematic. The discipline is maintained consistently, which is more important than the specific format chosen.

### 5.5 Keep a Changelog Format

The changelog follows the Keep a Changelog standard. This is a reasonable choice among several valid formats (GitHub Releases only, auto-generated from commits, etc.). The thoroughness of maintaining it through 131 versions is more notable than the format itself.

### 5.6 Model Profile System

The quality/balanced/budget model profile system for controlling which Claude model each agent uses is a reasonable approach. The default (Opus for planning, Sonnet for execution) reflects current model capabilities. Whether this belongs in the tool vs. being left to the user is debatable -- both approaches have merit.

### 5.7 Single Branch Strategy

Using only `main` with no develop or release branches is fine for a fast-moving solo project. The CONTRIBUTING.md's two-branch model (main + feature branches) is standard and appropriate. It would need evolution if the project grows to multiple active contributors, but that is a future concern.

---

## 6. What Could Be Improved

### 6.1 Add Installation and Hook Tests (Critical)

**Current state:** Zero automated tests. Platform-specific bugs recur repeatedly.

**Concrete recommendation:** Create a minimal test suite covering:

1. **Installer tests:** Verify file copying works for global/local, Claude/OpenCode/Gemini targets. Mock the filesystem. Test path resolution on Windows (backslashes), Linux (tilde expansion), and containers (no HOME variable).
2. **Hook tests:** Verify gsd-statusline.js produces valid output for edge cases (null remaining, negative values, boundary percentages). Verify gsd-check-update.js handles network failures gracefully.
3. **Markdown validation:** A script that scans all command/agent/workflow files for:
   - Valid YAML frontmatter
   - Resolved @-references (catches issue #339)
   - Required sections present (objective, success_criteria)
   - No orphaned file references

**Estimated effort:** 1-2 days for a meaningful test suite. The JavaScript tooling is already present (Node.js).

### 6.2 Implement Version Batching (High)

**Current state:** 131 tags in 48 days. The CONTRIBUTING.md describes the fix but it has not been implemented.

**Concrete recommendation:**
1. Stop publishing every single change as a new npm version
2. Accumulate patch fixes under `[Unreleased]` in the changelog
3. Publish weekly patch releases (e.g., every Monday) unless a critical fix requires immediate release
4. Use pre-release tags (`v1.12.0-alpha.1`) for experimental features before promoting to stable
5. Create GitHub Releases only for minor+ versions with curated release notes

This would reduce version noise by ~80% while maintaining the ability to ship critical fixes immediately.

### 6.3 Add Requirements Verification to Planning (High -- Our Fork Does This Better)

**Current state in upstream:** The plan checker has 6 verification dimensions (7 with the new context compliance). But it does not verify that plans actually implement the specific requirements from REQUIREMENTS.md.

**What our fork adds:** Mandatory requirements verification in both planning and execution workflows. This ensures that every plan traces back to specific requirement IDs and that execution verifies requirements are met, not just tasks completed.

**Evidence:** Our fork's `.planning/issues/002-requirements-verification.md` documents this gap and our fix. Upstream's plan checker verifies "requirement coverage" as Dimension 1, but this checks coverage of phase goals, not formal requirement traceability from REQUIREMENTS.md.

### 6.4 Implement Issue Triage and Prioritization (High)

**Current state:** 203 open issues with no labels, milestones, or prioritization.

**Concrete recommendation:**
1. Create issue labels: `bug`, `feature`, `question`, `duplicate`, `wontfix`, `good-first-issue`
2. Create priority labels: `P0-critical`, `P1-high`, `P2-medium`, `P3-low`
3. Close duplicates and "not-a-bug" issues (several in the current list are usage questions, not bugs)
4. Create a GitHub Project board with columns: Triage / Accepted / In Progress / Done
5. Tag issues #343 (RESEARCH.md data loss), #366 (race conditions), #351 (optimistic commits), and #369 (doesn't follow conventions) as P0-critical

### 6.5 Stabilize CI/CD Pipeline (Medium)

**Current state:** The CI/CD has been added, removed, re-added, and removed again within 8 days.

**Concrete recommendation:**
1. Choose one approach: either GitHub Actions auto-publish or manual `npm publish`
2. If automated: add basic checks (markdown linting, installer dry-run) before publish
3. If manual: document the exact workflow in MAINTAINERS.md (partially done) and enforce it
4. Add a `npm run prepublishOnly` script that runs the markdown validation from recommendation 6.1

### 6.6 Add Structured Error Handling to Agents (Medium)

**Current state:** Agent markdown files describe the happy path thoroughly but often lack error handling guidance. When an agent fails (e.g., researcher cannot find relevant information, planner produces plans that fail checker 3 times), the fallback behavior is inconsistently defined.

**Concrete recommendation:**
1. Every agent should have an explicit `<error_handling>` section
2. Define maximum retry counts (the plan checker has "max 3 revision iterations" -- good; generalize this)
3. Define graceful degradation paths (if research fails, proceed with reduced context rather than blocking)
4. Add timeout guidance for long-running agents

### 6.7 Address the @-Reference Resolution Problem (Medium)

**Current state:** Issue #339 "fix: remove unresolved @-references from plan templates and workflows" is open. @-references are used throughout for lazy loading but there is no validation that the referenced files actually exist.

**Concrete recommendation:**
1. Add a validation step to the installer that checks all @-references resolve
2. Alternatively, add a `/gsd:health-check` command (proposed in issue #338) that validates the integrity of the installation
3. Our fork could contribute this upstream

### 6.8 Decouple the Context Bar from Claude Code Internals (Low)

**Current state:** The context bar (gsd-statusline.js) hardcodes the assumption that Claude Code enforces an 80% context limit. The v1.10.0 fix scales the display to show 100% at 80% actual usage.

**Concrete recommendation:** Rather than hardcoding `80`, read the limit from Claude Code's configuration or accept it as a parameter. If Claude Code changes this threshold (which it may, as the product evolves), the statusline will silently become inaccurate again.

### 6.9 Create an Upgrade/Migration Path (Low)

**Current state:** Upgrading GSD means re-running `npx get-shit-done-cc@latest`, which does a clean install that "removes orphaned files" (v1.6.1). There is no migration tool for breaking changes.

**Evidence:** Issue #310 "feat: add migration command for v1.x to v2.0 upgrade" is open but unaddressed. Issue #330 "Update breaks status line - settings.json reference not updated" shows that upgrades can break existing installations.

**Concrete recommendation:** The installer should detect the current installed version and apply any necessary migrations (renaming files, updating settings.json references, etc.) rather than doing a destructive reinstall.

---

## Summary Assessment

### Overall Grade: B+

GSD is an impressive solo-developer project that has achieved significant community adoption (10,000+ stars) through genuine innovation in AI-assisted development workflows. The thin orchestrator pattern, plans-as-prompts architecture, and context engineering focus represent real contributions to the field.

The primary weaknesses -- zero testing, unsustainable versioning, and the ship-then-fix pattern -- are typical of rapid prototyping projects transitioning to production use. The developer shows clear signs of learning and maturation (the codebase intelligence revert, the CONTRIBUTING.md self-awareness, the slowing release cadence).

### Risk Profile for Our Fork

**Low risk:** The core architecture (commands, agents, workflows, templates) is stable and well-designed. Changes between versions are predominantly additive.

**Medium risk:** The rapid release cadence means our fork will fall behind quickly. The lack of automated tests means upstream changes may introduce regressions we would inherit.

**High risk:** The growing community contribution surface (OpenCode, Gemini, branching strategies) may introduce complexity that does not serve our specific use case. Selective merging (as documented in our `02-file-diff-comparison.md`) is the correct strategy.

### Key Takeaways

1. **Adopt upstream's CONTEXT.md flow fix (v1.11.1)** -- this is a genuine improvement our fork needs
2. **Do not adopt upstream's versioning practices** -- maintain our own sane versioning
3. **Consider contributing our requirements verification back upstream** -- it fills a real gap in their plan checker
4. **The architectural direction is sound** -- the thin orchestrator + specialized agents pattern will scale
5. **Watch the issue backlog** -- if critical bugs (#366 race conditions, #343 data loss) go unaddressed, it signals maintenance capacity problems

---

**Assessment Complete:** 2026-02-01
**Analyst:** Senior Software Engineering Analyst (Claude Opus 4.5)
