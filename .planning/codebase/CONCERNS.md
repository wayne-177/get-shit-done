# Codebase Concerns

**Analysis Date:** 2026-01-24

## Tech Debt

**Monolithic installer script:**
- Issue: Single 1,292-line Node.js file handles all installation complexity (`bin/install.js`). Complex logic includes YAML frontmatter conversion, path resolution, hook registration, environment variable expansion, and cross-runtime support (Claude Code + OpenCode).
- Files: `bin/install.js`
- Impact: Difficult to test individual installation steps; changes to install logic risk breaking both Claude Code and OpenCode installations across Windows/Mac/Linux; new runtime support requires modifying large entrypoint file.
- Fix approach: Extract logical concerns into smaller modules: path resolution (`lib/paths.js`), frontmatter conversion (`lib/converter.js`), hook registration (`lib/hooks.js`), environment setup (`lib/env.js`). Use dependency injection to test each module independently.

**Large agent files with multiple responsibilities:**
- Issue: Agents span 700-1,400 lines each and combine multiple concerns. Example: `agents/gsd-planner.md` (1,386 lines) handles multi-plan generation, constraint checking, workload analysis, and subagent orchestration; `agents/gsd-debugger.md` (1,203 lines) handles failure analysis, debugging strategies, and checkpoint generation.
- Files: `agents/gsd-planner.md`, `agents/gsd-debugger.md`, `agents/gsd-verifier.md` (778 lines)
- Impact: Hard to modify agent behavior without affecting other logic; testing specific workflows requires reading entire file; future agent enhancements risk scope creep.
- Fix approach: Extract orthogonal concerns into separate agent files or reusable workflow modules (e.g., constraint checking as separate workflow, workload analysis as separate workflow). Use cross-references instead of inline logic.

**Codebase Intelligence System removed but patterns remain:**
- Issue: Version 1.9.2 removed the entire Codebase Intelligence System due to overengineering (21MB sql.js dependency, 3,065 lines added). However, references to deprecated system may linger and removal was not comprehensive cleanup.
- Files: CHANGELOG.md (v1.9.2), agents/, workflows/
- Impact: Risk of incomplete removal leaving orphaned code paths or configuration; future developers might attempt to re-implement removed features without understanding why they were deprecated.
- Fix approach: Run comprehensive audit: search for SQLite imports, graph database references, entity file generation patterns; verify all intel hooks (gsd-intel-index.js, gsd-intel-session.js, gsd-intel-prune.js) are permanently removed; document deprecation rationale in README to prevent re-introduction.

**Hardcoded timeout values without configuration:**
- Issue: Timeout values hardcoded throughout workflows. Example: `timeout: 30000` in `analyze-tech-debt.md` (120s knip timeout), `timeout 120` in shell commands; retry logic caps at 3 attempts in `retry-orchestration.md`.
- Files: `get-shit-done/workflows/analyze-tech-debt.md`, `get-shit-done/workflows/retry-orchestration.md`, `get-shit-done/workflows/adaptive-failure-analysis.md`
- Impact: Large projects may timeout on analysis; network-heavy operations fail unexpectedly; users cannot override timeouts for slow environments.
- Fix approach: Extract timeout values to `config.json` with sensible defaults; add timeout calculation based on project size (line count, file count); document timeout override mechanism in GSD-STYLE.md.

**Manual changelog management:**
- Issue: CHANGELOG.md follows Keep a Changelog format but is manually maintained. Version bumps done via `npm version` command require separate CHANGELOG updates; risk of version/changelog skew.
- Files: `CHANGELOG.md`, `package.json`
- Impact: Changelog can become out-of-sync with actual releases; contributors might forget to update; releasing requires multiple manual steps (npm version, edit CHANGELOG, commit, tag, publish).
- Fix approach: Automate changelog generation from conventional commits on release. Use `conventional-changelog` or similar to extract feat/fix/breaking from git log; maintain unreleased section automatically; make npm publish script update CHANGELOG before tagging.

## Known Bugs

**Installation on non-TTY terminals:**
- Symptoms: Interactive installation fails silently on CI/CD or headless environments (WSL2, SSH, etc.).
- Files: `bin/install.js` (lines 1096-1130 where readline is created)
- Trigger: Running `npx get-shit-done-cc` without stdin available or in non-interactive context
- Workaround: Use explicit flags (`--global --claude` or `--both --opencode`) to skip interactive prompts; v1.6.4 fixes WSL2 issue but edge cases remain for truly headless environments.
- Status: Fixed in v1.6.4 but may regress if new interactive features added.

**OpenCode command path structure incompatibility:**
- Symptoms: OpenCode users see incorrect command paths in documentation and help text (e.g., `/gsd:help` shown instead of `/gsd-help`).
- Files: `bin/install.js` (line 281: command conversion), documentation files
- Trigger: Installation creates flat command structure but docs reference colon-based paths
- Workaround: Users must know to use dash-based paths (`/gsd-help` not `/gsd:help`); no help system guides them to correct syntax.
- Current mitigation: Frontmatter converter in `bin/install.js` replaces `/gsd:` with `/gsd-` in content (lines 280-281).

**Phase directory matching edge case:**
- Symptoms: Phase detection fails when folder names use different zero-padding (e.g., `05-phase` vs `5-phase`).
- Files: Orchestrator and workflow path resolution (affected across multiple workflows)
- Trigger: Renaming phase directories or using non-standard numbering
- Workaround: Stick to consistent zero-padded naming (01-, 02-, etc. or 1-, 2-)
- Status: Fixed in v1.5.28 but pattern matching remains fragile.

**Windows UNC path handling:**
- Symptoms: Installation fails on Windows with UNC paths (network shares like `\\server\share`)
- Files: `bin/install.js` (path resolution sections)
- Trigger: Installing from network drive on Windows
- Workaround: Copy GSD to local drive before installing; v1.9.5 improved handling but edge cases may remain.

## Security Considerations

**Secrets in environment variables:**
- Risk: Installer reads and propagates environment variables including API keys and credentials. If Claude Code/OpenCode crashes or leaks context, secrets in settings could be exposed.
- Files: `bin/install.js` (env var handling), `get-shit-done/workflows/` (env var usage)
- Current mitigation: `.env` files are read but not stored in config; secrets passed via environment only; `.gitignore` includes `.env` files; installer validates env var names before using them.
- Recommendations: Document that only users with access to machine should run install; add warning about storing secrets in shared config directories; validate that sensitive env vars (API_KEY, TOKEN, SECRET) are never written to JSON config files.

**Hook command injection:**
- Risk: statusline hook runs shell commands from settings.json. If settings file is corrupted or user has malicious config, arbitrary commands could execute.
- Files: `bin/install.js` (hook registration), `hooks/gsd-statusline.js`
- Current mitigation: Hooks are registered by installer only; settings.json validated before use (line 207-211 try/catch); hook commands built with proper escaping.
- Recommendations: Add hook command validation whitelist; escape all hook command arguments; document that settings.json should not be manually edited; add integrity check for settings.json on startup.

**Package dependency supply chain:**
- Risk: Only two dependencies: esbuild (dev-only) and zero production dependencies. Minimal risk, but installers copies entire GSD codebase into user's config directory without verification.
- Files: `package.json`, `bin/install.js`
- Current mitigation: No production dependencies reduces attack surface; npm integrity checks via package-lock.json; GSD source is open-source and auditable on GitHub.
- Recommendations: Document that users should review GSD codebase before installing; add checksum verification option for installed files; implement npm audit pre-install check.

## Performance Bottlenecks

**Knip static analysis timeout on large projects:**
- Problem: Dead code detection using knip can timeout on codebases >10k files or with complex dependencies. Hard-coded 120-second timeout (line 162 in `analyze-tech-debt.md`) may be insufficient.
- Files: `get-shit-done/workflows/analyze-tech-debt.md` (line 162)
- Cause: Knip must parse entire project dependency graph; projects with circular dependencies or complex monorepos hit timeout.
- Current workaround: Workflow catches timeout, reports empty results, and notes limitation (line 638).
- Improvement path: Make timeout dynamic based on project size; allow skipping dead code analysis for large projects; run knip with `--strict` to fail fast on issues.

**Context window bloat in large workflows:**
- Problem: Workflows like `execute-plan.md` (1,844 lines) and `understand-goal.md` (1,028 lines) fully inline without references. When Claude loads these, full file content occupies context.
- Files: `get-shit-done/workflows/execute-plan.md`, `get-shit-done/workflows/understand-goal.md`, and others
- Cause: Markdown files designed to be self-contained; no lazy-loading or references system.
- Current workaround: Orchestrators selectively load workflows; only relevant workflow is inlined in agent prompts.
- Improvement path: Extract reusable sub-workflows into separate files; use `@file` references in prompts to lazy-load only needed sections; cache frequently-used workflow sections.

**Multi-agent orchestration serializes dependent tasks:**
- Problem: Retry orchestration workflow (lines 65-76) processes retries serially: analyze failure → select strategy → execute → check → repeat. If a strategy takes 5 minutes, user waits 15+ minutes for 3 attempts before escalation.
- Files: `get-shit-done/workflows/retry-orchestration.md`
- Cause: Orchestration pattern prioritizes safety over speed; no parallelization of independent retry strategies.
- Improvement path: Allow parallel retry strategy execution for stateless tasks; add timeout escalation (give up after N minutes even if attempts remain); provide progress feedback to user during retries.

**Checkpoint resolution blocks workflow:**
- Problem: Workflows pause execution at checkpoints waiting for user input. Long checkpoints (multi-step verification) can delay phase completion by hours.
- Files: `get-shit-done/references/checkpoints.md`
- Current mitigation: Design pattern emphasizes automating verification before checkpoint (Claude starts dev server, runs tests, etc.); checkpoints are for human judgment only.
- Improvement path: Add async checkpoint callback; allow users to check status without blocking; provide estimated wait time for complex verifications.

## Fragile Areas

**Frontmatter YAML parsing:**
- Files: `bin/install.js` (lines 274-365: convertClaudeToOpencodeFrontmatter function)
- Why fragile: Simple regex-based YAML parser doesn't handle edge cases: multi-line values, escaped quotes, non-ASCII characters, inline YAML arrays with complex syntax.
- Safe modification: Add proper YAML parser (js-yaml library); validate all parsed fields before use; add test cases for complex frontmatter examples.
- Test coverage: No tests for frontmatter conversion; edge cases like `allowed-tools: [AskUserQuestion, "Tool With Spaces"]` likely break.

**File copying and directory traversal:**
- Files: `bin/install.js` (lines 403-461: copyFlattenedCommands and copyDirectory functions)
- Why fragile: Recursive directory copy without symlink handling, permission preservation, or atomic operations. If copy fails mid-way, partial installation left in place.
- Safe modification: Use `fs-extra` or similar for robust copy; implement atomic copy (copy to temp dir, then rename); add pre-flight checks to verify source exists and has required permissions.
- Test coverage: Manual testing only; untested on Windows with symlinks or permission-restricted directories.

**Hook registration in settings.json:**
- Files: `bin/install.js` (lines 1001-1025: SessionStart hook registration)
- Why fragile: Assumes specific settings.json structure; mutating settings deeply without validation; if hooks array corrupted, all hooks lose functionality.
- Safe modification: Use JSON schema validation before mutating; implement atomic updates (write to temp file, validate, then replace); add rollback capability if mutation fails.
- Test coverage: Not tested; untested on corrupted settings.json files or concurrent installations.

**Regex-based command path conversion:**
- Files: `bin/install.js` (lines 277-283: command path replacements)
- Why fragile: Simple global regex replacements can over-match. Example: `/gsd:` replacement matches in documentation links, comments, and command definitions; could replace `/gsd:help-command` in URLs incorrectly.
- Safe modification: Parse frontmatter properly; only replace in specific fields (name, description); use word boundaries in regex; add test cases with edge cases.

## Missing Critical Features

**No automated rollback on failed installs:**
- Problem: If installation fails mid-way, GSD files partially installed but old version still in use. User has corrupted state.
- Blocks: Clean recovery from failed installs; users must manually uninstall and reinstall.
- Current workaround: `--uninstall` flag available to clean state; v1.6.1 improved cleanup but doesn't handle partial state.
- Improvement path: Implement atomic install (stage in temp directory, validate complete, then swap); add rollback trigger if installation incomplete after timeout.

**No build-time integrity verification:**
- Problem: Users run `npx get-shit-done-cc` without verifying package integrity. npm package could be tampered with between download and install.
- Blocks: Users can't verify they're running authentic GSD.
- Current mitigation: Source is open-source on GitHub; npm publish requires auth; package-lock.json provides reproducibility.
- Improvement path: Add code signing (sign release tarballs with maintainer key); provide checksum verification command; integrate with npm audit security tooling.

**No automated upgrade rollback:**
- Problem: If new version has bugs, users manually downgrade by re-running old version or editing package.json.
- Blocks: Fast recovery from bad releases; no automatic rollback path.
- Current workaround: Changelog shows breaking changes; users must read before upgrading.
- Improvement path: Keep previous version installed; add `/gsd:rollback` command to restore prior version; implement canary release testing before major releases.

**No installation health check:**
- Problem: After install completes, no verification that all files copied correctly or hooks registered properly.
- Blocks: Silent failures if installation incomplete; users discover issues only when running commands.
- Current mitigation: Installation shows checkmarks for each step (lines 856-989) but doesn't verify end-to-end functionality.
- Improvement path: Add `/gsd:verify-install` command that checks: all command files present, hooks registered in settings, Claude/OpenCode config accessible, sandbox permissions correct.

## Test Coverage Gaps

**Installation logic untested:**
- What's not tested: Core installation workflows, path resolution across OS, frontmatter conversion, hook registration, settings mutation, multi-runtime installation.
- Files: `bin/install.js` (no test file exists)
- Risk: Installation is critical path; bugs affect all users; no regression tests prevent re-introduction of fixed bugs.
- Priority: High — installation is most common user interaction; bugs here block entire workflow.

**Converter edge cases untested:**
- What's not tested: Frontmatter conversion with complex YAML, tool name mapping edge cases, command path replacement in URLs vs. commands, color name to hex conversion.
- Files: `bin/install.js` (lines 274-365)
- Risk: OpenCode users may have broken commands if frontmatter conversion fails silently.
- Priority: High — affects OpenCode user experience.

**Hook registration untested:**
- What's not tested: Duplicate hook detection, corrupted settings.json handling, concurrent hook registrations, hook removal on uninstall.
- Files: `bin/install.js` (lines 1001-1025, 651-663)
- Risk: Hooks may not fire or may fire multiple times if registration broken; uninstall may leave orphaned hooks.
- Priority: Medium — affects statusline and update checks but not core functionality.

**Adapter pattern for multi-runtime support untested:**
- What's not tested: Installation differences between Claude Code and OpenCode; config directory detection; path escaping on Windows; XDG spec compliance for OpenCode.
- Files: `bin/install.js` (getGlobalDir, getOpencodeGlobalDir functions)
- Risk: OpenCode users may have installation failures if config directory detection broken; Windows users may have path escaping issues.
- Priority: High — affects subset of users but failures are complete install failures.

---

*Concerns audit: 2026-01-24*
