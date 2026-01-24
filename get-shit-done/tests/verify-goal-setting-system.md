# Goal Setting System Verification Tests

This guide provides manual test procedures to validate that the v4.5 Autonomous Goal Setting features work correctly. These tests ensure the codebase analysis, suggestion engine, CLAUDE.md integration, and auto-refresh work properly both independently and as a complete pipeline.

## Purpose

Validate v4.5 Autonomous Goal Setting enhancements:
1. Dependency analyzer (npm audit + deps.dev enrichment + outdated detection)
2. Security analyzer (CVE-focused, HIGH/CRITICAL filtering, lockfile validation)
3. Tech debt analyzer (Knip integration, dead code detection, weighted scoring)
4. Codebase analysis orchestration (parallel execution, ANALYSIS_REPORT.md)
5. Suggestion generation (scoring formula, knowledge base integration)
6. /gsd:suggest command (staleness detection, on-demand generation)
7. CLAUDE.md integration (section replacement, compact format)
8. Milestone auto-refresh (optional suggestion refresh on completion)
9. End-to-end pipeline validation
10. Backward compatibility with v4.0/v3.0/v2.0

## Prerequisites

### Required Configuration

```bash
# Create or update .planning/config.json with full v4.5 features
cat > .planning/config.json << 'EOF'
{
  "mode": "interactive",
  "retry": {
    "enabled": true,
    "max_attempts": 3,
    "log_attempts": true
  },
  "verification": {
    "enabled": true
  },
  "adaptive": {
    "enabled": true,
    "fallback_to_static": true,
    "history_tracking": true,
    "prediction_enabled": true,
    "show_pattern_hints": true
  },
  "suggestions": {
    "auto_suggest": true,
    "auto_update_claude_md": true,
    "refresh_suggestions_on_milestone": false
  }
}
EOF
```

### Required Files

```bash
# Verify all v4.5 workflow dependencies exist
ls get-shit-done/workflows/analyze-dependencies.md
ls get-shit-done/workflows/analyze-security.md
ls get-shit-done/workflows/analyze-tech-debt.md
ls get-shit-done/workflows/analyze-codebase.md
ls get-shit-done/workflows/generate-suggestions.md
ls get-shit-done/workflows/update-claude-md.md

# Verify templates exist
ls get-shit-done/templates/analysis-report.md
ls get-shit-done/templates/goal-proposals.md
ls get-shit-done/templates/claude-md.md

# Verify commands exist
ls commands/gsd/analyze-codebase.md
ls get-shit-done/commands/suggest.md
```

### Test Workspace Setup

```bash
# Create test directory
mkdir -p .planning/test-workspace-goal-setting
cd .planning/test-workspace-goal-setting
```

---

## Test Scenario 1: Dependency Analyzer

**Objective:** Verify analyze-dependencies.md detects vulnerabilities and outdated packages using npm audit, deps.dev API, and npm outdated

### Prerequisites

- Node.js project with package.json
- npm installed
- Optional: jq for JSON parsing (graceful degradation if missing)

### Setup

```bash
# Ensure package.json exists
ls package.json

# Verify npm is available
npm --version

# Optional: Check if jq is available (workflow handles missing jq)
jq --version 2>/dev/null || echo "jq not available - workflow will use fallback parsing"
```

### Test Execution

**Test 1a: npm audit integration**

```bash
# Run npm audit directly to see baseline
npm audit --json 2>/dev/null | head -50

# Run the analyze-dependencies workflow (manually or via orchestrator)
# Expected: Parses npm audit JSON, extracts vulnerabilities
```

**Test 1b: deps.dev API enrichment**

```bash
# Test deps.dev API availability
curl -s "https://api.deps.dev/v3alpha/systems/npm/packages/lodash" | head -20

# Expected: Workflow enriches npm audit findings with deps.dev advisory data
# Graceful degradation: If API unavailable, uses npm audit data only
```

**Test 1c: Outdated package detection**

```bash
# Run npm outdated directly
npm outdated --json 2>/dev/null

# Expected: Workflow categorizes as:
# - outdated-major: Major version behind (e.g., 3.x → 4.x)
# - outdated-minor: Minor version behind (e.g., 4.1 → 4.3)
# - outdated-patch: Patch version behind (e.g., 4.1.0 → 4.1.2)
```

### Verification Steps

1. **Verify JSON output format:**
   ```bash
   # Check .planning/analysis/dependencies-findings.json exists
   ls .planning/analysis/dependencies-findings.json

   # Verify JSON structure
   cat .planning/analysis/dependencies-findings.json | head -30
   # Expected: JSON with status, findings array, summary
   ```

2. **Verify markdown report:**
   ```bash
   # Check .planning/analysis/dependencies-report.md exists
   cat .planning/analysis/dependencies-report.md | head -50
   # Expected: Human-readable report with vulnerability summary
   ```

3. **Verify graceful degradation:**
   ```bash
   # Test with curl unavailable (mock)
   # Expected: Workflow completes without deps.dev enrichment, logs warning
   ```

### Success Criteria

- [ ] npm audit JSON parsed correctly
- [ ] deps.dev API enrichment works (or gracefully skips)
- [ ] Outdated packages categorized by severity (major/minor/patch)
- [ ] JSON output follows schema (status, findings, summary)
- [ ] Markdown report is human-readable
- [ ] Missing optional tools don't cause failure

---

## Test Scenario 2: Security Analyzer

**Objective:** Verify analyze-security.md extracts CVEs with HIGH/CRITICAL focus and validates lockfile integrity

### Prerequisites

- Node.js project with package.json
- package-lock.json or yarn.lock present
- Optional: lockfile-lint for supply chain validation

### Setup

```bash
# Verify lockfile exists
ls package-lock.json || ls yarn.lock

# Optional: Check if lockfile-lint is available
npx lockfile-lint --version 2>/dev/null || echo "lockfile-lint not available - will skip supply chain checks"
```

### Test Execution

**Test 2a: CVE-focused analysis**

```bash
# Run security analysis workflow
# Expected: Extracts CVE identifiers from npm audit output
# Expected: Filters to HIGH and CRITICAL severity by default
```

**Test 2b: lockfile validation**

```bash
# If lockfile-lint available:
npx lockfile-lint --path package-lock.json --allowed-hosts npm --allowed-schemes https

# Expected: Workflow validates lockfile integrity
# Expected: Detects registry hijack attempts, http URLs, suspicious hosts
```

**Test 2c: Fix availability classification**

```bash
# Expected: Classifies vulnerabilities by fix availability:
# - fix-available: npm audit fix can resolve
# - fix-unavailable: Requires manual intervention
# - breaking-fix: Fix requires major version bump
```

### Verification Steps

1. **Verify HIGH/CRITICAL filtering:**
   ```bash
   cat .planning/analysis/security-findings.json
   # Expected: Only HIGH and CRITICAL severity vulnerabilities
   # Expected: LOW and MODERATE filtered out
   ```

2. **Verify CVE extraction:**
   ```bash
   grep -o "CVE-[0-9]\{4\}-[0-9]*" .planning/analysis/security-findings.json | sort -u
   # Expected: List of unique CVE identifiers
   ```

3. **Verify fix classification:**
   ```bash
   grep "fix_status" .planning/analysis/security-findings.json
   # Expected: Each finding has fix_status field
   ```

4. **Verify lockfile validation (if available):**
   ```bash
   grep "lockfile" .planning/analysis/security-report.md
   # Expected: Lockfile validation section with pass/fail status
   ```

### Success Criteria

- [ ] CVE identifiers extracted from vulnerabilities
- [ ] HIGH/CRITICAL filtering removes low-priority noise
- [ ] Fix availability classified (available/unavailable/breaking)
- [ ] Lockfile validation works (or gracefully skips)
- [ ] JSON output follows structured findings format
- [ ] Human-readable markdown report generated

---

## Test Scenario 3: Tech Debt Analyzer

**Objective:** Verify analyze-tech-debt.md detects dead code using Knip and calculates weighted tech debt score

### Prerequisites

- Node.js or TypeScript project
- Optional: Knip installed (`npm install -D knip`)

### Setup

```bash
# Check if Knip is available
npx knip --version 2>/dev/null || echo "Knip not available - will use fallback analysis"

# If not installed, optionally install:
# npm install -D knip
```

### Test Execution

**Test 3a: Knip integration**

```bash
# Run Knip directly to see baseline
npx knip --reporter json 2>/dev/null | head -50

# Expected: Workflow parses Knip output for:
# - Unused files
# - Unused dependencies
# - Unused exports
# - Unlisted dependencies
```

**Test 3b: Dead code analysis**

```bash
# Expected: Identifies:
# - Files not imported anywhere
# - Exports not used by other modules
# - Dependencies in package.json not imported
```

**Test 3c: Weighted scoring system**

```bash
# Expected scoring weights:
# - Unused files: ×5 (file-level dead code is significant)
# - Unused dependencies: ×10 (bloat + potential vulnerabilities)
# - Unused exports: ×2 (API surface cleanup)
# - Outdated packages: ×3 (maintenance burden)
# - Significantly outdated (major): ×8 (high update risk)

# Formula: sum(item_count × weight)
# Score interpretation:
# - 0-50: Low tech debt (HEALTHY)
# - 51-100: Moderate tech debt (REVIEW_RECOMMENDED)
# - 101-200: High tech debt (MAINTENANCE_NEEDED)
# - 201+: Critical tech debt (ACTION_REQUIRED)
```

### Verification Steps

1. **Verify Knip output parsing:**
   ```bash
   cat .planning/analysis/tech-debt-findings.json
   # Expected: JSON with unused_files, unused_deps, unused_exports arrays
   ```

2. **Verify weighted score calculation:**
   ```bash
   grep "tech_debt_score" .planning/analysis/tech-debt-findings.json
   # Expected: Numeric score calculated from weights
   ```

3. **Verify graceful degradation without Knip:**
   ```bash
   # Test without Knip installed
   # Expected: Workflow falls back to outdated package analysis
   # Expected: Logs warning about limited analysis
   ```

4. **Verify score interpretation:**
   ```bash
   grep "status" .planning/analysis/tech-debt-findings.json
   # Expected: HEALTHY, REVIEW_RECOMMENDED, MAINTENANCE_NEEDED, or ACTION_REQUIRED
   ```

### Success Criteria

- [ ] Knip output parsed correctly (or graceful fallback)
- [ ] Dead code detected (unused files, deps, exports)
- [ ] Weighted scoring formula applied correctly
- [ ] Score interpretation matches thresholds
- [ ] JSON output follows structured format
- [ ] Human-readable report generated

---

## Test Scenario 4: Codebase Analysis Orchestration

**Objective:** Verify analyze-codebase.md runs all three analyzers in parallel and produces unified ANALYSIS_REPORT.md

### Prerequisites

- All three analyzer workflows available
- Test workspace with package.json

### Setup

```bash
# Verify all analyzer workflows exist
ls get-shit-done/workflows/analyze-dependencies.md
ls get-shit-done/workflows/analyze-security.md
ls get-shit-done/workflows/analyze-tech-debt.md
```

### Test Execution

**Test 4a: Parallel analyzer execution**

```bash
# Run /gsd:analyze-codebase command
# Expected: All three analyzers run concurrently using background jobs (&)

# Timing check: Should complete faster than sequential execution
time /gsd:analyze-codebase
```

**Test 4b: ANALYSIS_REPORT.md generation**

```bash
# Check output file created
cat .planning/ANALYSIS_REPORT.md

# Expected sections:
# - Executive Summary
# - Dependencies Analysis
# - Security Analysis
# - Tech Debt Analysis
# - Recommendations (for Phase 12)
```

**Test 4c: Priority hierarchy**

```bash
# Expected priority order (highest to lowest):
# 1. CRITICAL security vulnerabilities
# 2. HIGH security vulnerabilities
# 3. Major outdated dependencies
# 4. Tech debt (weighted by score)
# 5. Minor/patch updates

# Verify in Recommendations section
grep -A 20 "## Recommendations" .planning/ANALYSIS_REPORT.md
```

### Verification Steps

1. **Verify parallel execution:**
   ```bash
   # Check for background job spawning in workflow
   # Expected: All three analyzers start near-simultaneously
   ```

2. **Verify unified status:**
   ```bash
   grep "Overall Status" .planning/ANALYSIS_REPORT.md
   # Expected: Single status derived from priority hierarchy
   # HEALTHY → REVIEW_RECOMMENDED → MAINTENANCE_NEEDED → ACTION_REQUIRED → CRITICAL
   ```

3. **Verify Recommendations section format:**
   ```bash
   grep -A 30 "## Recommendations" .planning/ANALYSIS_REPORT.md
   # Expected: Structured data for Phase 12 suggestion engine
   # - goal_type: security-fix | dependency-update | tech-debt-cleanup
   # - effort: quick-fix | moderate | significant
   # - priority: CRITICAL | HIGH | MEDIUM | LOW
   ```

4. **Verify individual analysis sections:**
   ```bash
   grep "## Dependencies" .planning/ANALYSIS_REPORT.md
   grep "## Security" .planning/ANALYSIS_REPORT.md
   grep "## Tech Debt" .planning/ANALYSIS_REPORT.md
   # Expected: All three sections present
   ```

### Success Criteria

- [ ] All three analyzers run in parallel
- [ ] ANALYSIS_REPORT.md created in .planning/
- [ ] Priority hierarchy correctly orders findings
- [ ] Unified status reflects worst finding
- [ ] Recommendations section formatted for Phase 12
- [ ] /gsd:analyze-codebase command works

---

## Test Scenario 5: Suggestion Generation

**Objective:** Verify generate-suggestions.md transforms ANALYSIS_REPORT.md into GOAL_PROPOSALS.md with scoring

### Prerequisites

- ANALYSIS_REPORT.md exists (from Scenario 4)
- Optional: Knowledge base at ~/.claude/gsd-knowledge/ for pattern boosts

### Setup

```bash
# Verify ANALYSIS_REPORT.md exists
ls .planning/ANALYSIS_REPORT.md

# Check if knowledge base exists (optional)
ls ~/.claude/gsd-knowledge/index.json 2>/dev/null || echo "Knowledge base not available - no pattern boosts"
```

### Test Execution

**Test 5a: Analysis parsing**

```bash
# Run generate-suggestions workflow
# Expected: Parses Recommendations section from ANALYSIS_REPORT.md

# Verify parsing
grep "## Recommendations" .planning/ANALYSIS_REPORT.md
```

**Test 5b: Scoring formula**

```bash
# Expected scoring:
# Base score by priority:
#   CRITICAL: 100
#   HIGH: 75
#   MEDIUM: 50
#   LOW: 25

# Effort modifier:
#   quick-fix: ×1.2 (+20%)
#   moderate: ×1.0 (no change)
#   significant: ×0.8 (-20%)

# Pattern boost (if knowledge base available):
#   +10 if similar task succeeded in knowledge base

# Final score = base × effort_modifier + pattern_boost
```

**Test 5c: Knowledge base integration**

```bash
# If knowledge base exists:
ls ~/.claude/gsd-knowledge/patterns/strategy-effectiveness/

# Expected: Workflow queries patterns matching goal types
# Expected: Adds pattern boost (+10) for patterns with >70% success rate
# Expected: Adds warning flag for patterns with >40% failure rate
```

### Verification Steps

1. **Verify GOAL_PROPOSALS.md created:**
   ```bash
   ls .planning/GOAL_PROPOSALS.md
   cat .planning/GOAL_PROPOSALS.md | head -60
   ```

2. **Verify scoring applied:**
   ```bash
   grep -E "Score: [0-9]+" .planning/GOAL_PROPOSALS.md
   # Expected: Each suggestion has numeric score
   ```

3. **Verify effort modifiers:**
   ```bash
   grep -E "(quick-fix|moderate|significant)" .planning/GOAL_PROPOSALS.md
   # Expected: Effort level shown for each suggestion
   ```

4. **Verify pattern warnings (if applicable):**
   ```bash
   grep "Warning:" .planning/GOAL_PROPOSALS.md
   # Expected: Warning flag for goals with high failure rate in knowledge base
   ```

5. **Verify graceful degradation without knowledge base:**
   ```bash
   # Remove knowledge base temporarily
   mv ~/.claude/gsd-knowledge ~/.claude/gsd-knowledge-backup

   # Run workflow
   # Expected: Generates suggestions without pattern boosts
   # Expected: No errors about missing knowledge base

   # Restore
   mv ~/.claude/gsd-knowledge-backup ~/.claude/gsd-knowledge
   ```

### Success Criteria

- [ ] GOAL_PROPOSALS.md created from ANALYSIS_REPORT.md
- [ ] Scoring formula applied (base + effort modifier + pattern boost)
- [ ] Quick wins scored higher than significant efforts
- [ ] Knowledge base patterns integrated (if available)
- [ ] Warning flags for high-failure patterns
- [ ] Graceful degradation without knowledge base

---

## Test Scenario 6: /gsd:suggest Command

**Objective:** Verify /gsd:suggest generates suggestions on-demand with staleness detection and CLAUDE.md update

### Prerequisites

- Suggestion workflows available
- ANALYSIS_REPORT.md exists (or will trigger analysis)

### Setup

```bash
# Verify suggest command exists
ls get-shit-done/commands/suggest.md

# Check existing state
ls .planning/GOAL_PROPOSALS.md 2>/dev/null || echo "No existing proposals"
ls .planning/ANALYSIS_REPORT.md 2>/dev/null || echo "No existing analysis"
```

### Test Execution

**Test 6a: On-demand suggestion generation**

```bash
# Run /gsd:suggest
# Expected: Generates GOAL_PROPOSALS.md

# If ANALYSIS_REPORT.md missing or stale:
# Expected: Runs analyze-codebase first
```

**Test 6b: Staleness detection**

```bash
# Compare timestamps
ANALYSIS_TIME=$(grep "Generated:" .planning/ANALYSIS_REPORT.md | head -1)
PROPOSAL_TIME=$(grep "Analysis Report:" .planning/GOAL_PROPOSALS.md | head -1)

echo "Analysis: $ANALYSIS_TIME"
echo "Proposals: $PROPOSAL_TIME"

# Expected: If timestamps don't match, proposals are stale
# Expected: /gsd:suggest warns about staleness
```

**Test 6c: auto_update_claude_md behavior**

```bash
# Test with auto_update_claude_md=true (default)
# Expected: CLAUDE.md updated after suggestion generation

# Check CLAUDE.md updated
grep "## Suggested Goals" CLAUDE.md

# Test with auto_update_claude_md=false
cat > .planning/config.json << 'EOF'
{
  "suggestions": {
    "auto_update_claude_md": false
  }
}
EOF

# Run /gsd:suggest
# Expected: GOAL_PROPOSALS.md updated, CLAUDE.md NOT updated
```

**Test 6d: disabled auto_suggest behavior**

```bash
# Test with auto_suggest=false
cat > .planning/config.json << 'EOF'
{
  "suggestions": {
    "auto_suggest": false
  }
}
EOF

# Run /gsd:analyze-codebase
# Expected: ANALYSIS_REPORT.md created, GOAL_PROPOSALS.md NOT auto-generated
# Expected: Must run /gsd:suggest manually to generate proposals
```

### Verification Steps

1. **Verify on-demand generation:**
   ```bash
   # Delete existing proposals
   rm .planning/GOAL_PROPOSALS.md

   # Run /gsd:suggest
   # Expected: GOAL_PROPOSALS.md regenerated
   ls .planning/GOAL_PROPOSALS.md
   ```

2. **Verify staleness detection output:**
   ```bash
   # Modify ANALYSIS_REPORT.md timestamp
   # Run /gsd:suggest
   # Expected: Message about stale analysis or auto-refresh
   ```

3. **Verify CLAUDE.md update:**
   ```bash
   cat CLAUDE.md | grep -A 20 "## Suggested Goals"
   # Expected: Suggestions section with Quick Wins and Requires Planning
   ```

### Success Criteria

- [ ] On-demand suggestion generation works
- [ ] Staleness detection compares timestamps correctly
- [ ] auto_update_claude_md=true updates CLAUDE.md
- [ ] auto_update_claude_md=false skips CLAUDE.md update
- [ ] auto_suggest=false disables automatic generation after analysis

---

## Test Scenario 7: CLAUDE.md Integration

**Objective:** Verify update-claude-md.md updates only the Suggested Goals section while preserving other content

### Prerequisites

- GOAL_PROPOSALS.md exists
- CLAUDE.md exists (or will be created)

### Setup

```bash
# Verify GOAL_PROPOSALS.md exists
ls .planning/GOAL_PROPOSALS.md

# Create or verify CLAUDE.md with custom sections
cat > CLAUDE.md << 'EOF'
# Project Name

## Quick Context
This section contains project-specific context.
DO NOT MODIFY THIS SECTION AUTOMATICALLY.

## Development Notes
- Custom note 1
- Custom note 2

## Suggested Goals
(This section will be auto-updated)

## Custom Section
User-defined content here.
EOF
```

### Test Execution

**Test 7a: Section replacement (only Suggested Goals)**

```bash
# Run update-claude-md workflow
# Expected: Only ## Suggested Goals section updated
# Expected: All other sections preserved exactly

# Verify other sections unchanged
diff <(cat CLAUDE.md | grep -A 5 "## Quick Context") \
     <(echo "## Quick Context
This section contains project-specific context.
DO NOT MODIFY THIS SECTION AUTOMATICALLY.")
```

**Test 7b: Staleness detection**

```bash
# Check staleness indicators
grep -E "(Fresh|Stale|Very Stale)" CLAUDE.md

# Expected staleness thresholds:
# - Fresh: Suggestions generated within 7 days
# - Stale: 7-14 days old
# - Very Stale: >14 days old

# Expected: Staleness indicator shown in Suggested Goals section
```

**Test 7c: Compact format**

```bash
# Verify Quick Wins format (numbered)
grep -A 10 "### Quick Wins" CLAUDE.md
# Expected:
# 1. **Goal name** (Score: XX) - `/gsd:understand "description"`
# 2. **Goal name** (Score: XX) - `/gsd:understand "description"`

# Verify Requires Planning format (bullets)
grep -A 10 "### Requires Planning" CLAUDE.md
# Expected:
# - **Goal name** (Score: XX, Effort: moderate)
# - **Goal name** (Score: XX, Effort: significant)
```

**Test 7d: Silent skip when GOAL_PROPOSALS.md missing**

```bash
# Remove GOAL_PROPOSALS.md
mv .planning/GOAL_PROPOSALS.md .planning/GOAL_PROPOSALS.md.bak

# Run update-claude-md workflow
# Expected: Exit 0 (success), no error message
# Expected: CLAUDE.md unchanged or Suggested Goals section removed

# Restore
mv .planning/GOAL_PROPOSALS.md.bak .planning/GOAL_PROPOSALS.md
```

### Verification Steps

1. **Verify section preservation:**
   ```bash
   # Check all original sections still present
   grep "## Quick Context" CLAUDE.md
   grep "## Development Notes" CLAUDE.md
   grep "## Custom Section" CLAUDE.md
   # Expected: All sections intact
   ```

2. **Verify Suggested Goals updated:**
   ```bash
   grep -A 30 "## Suggested Goals" CLAUDE.md
   # Expected: Contains Quick Wins (numbered) and Requires Planning (bullets)
   ```

3. **Verify staleness indicator:**
   ```bash
   grep -E "Last updated|Staleness" CLAUDE.md
   # Expected: Timestamp and/or staleness indicator present
   ```

### Success Criteria

- [ ] Only ## Suggested Goals section updated
- [ ] All other CLAUDE.md sections preserved exactly
- [ ] Staleness detection works (Fresh/Stale/Very Stale)
- [ ] Compact format with Quick Wins (numbered) and Requires Planning (bullets)
- [ ] Silent skip when GOAL_PROPOSALS.md missing (no error)

---

## Test Scenario 8: Milestone Auto-refresh

**Objective:** Verify refresh_suggestions_on_milestone configuration controls suggestion refresh during milestone completion

### Prerequisites

- Complete milestone workflow available
- Configuration with refresh_suggestions_on_milestone setting

### Setup

```bash
# Verify complete-milestone workflow exists
ls get-shit-done/workflows/complete-milestone.md

# Check current config
grep "refresh_suggestions_on_milestone" .planning/config.json
```

### Test Execution

**Test 8a: refresh_suggestions_on_milestone=true behavior**

```bash
# Set config to enable auto-refresh
cat > .planning/config.json << 'EOF'
{
  "suggestions": {
    "refresh_suggestions_on_milestone": true
  }
}
EOF

# Run /gsd:complete-milestone
# Expected: Triggers analyze-codebase and generate-suggestions before archive

# Verify suggestions refreshed
ls -la .planning/GOAL_PROPOSALS.md
# Expected: Recent modification timestamp
```

**Test 8b: refresh_suggestions_on_milestone=false behavior (default)**

```bash
# Set config to disable auto-refresh (or use default)
cat > .planning/config.json << 'EOF'
{
  "suggestions": {
    "refresh_suggestions_on_milestone": false
  }
}
EOF

# Note current GOAL_PROPOSALS.md timestamp
BEFORE=$(stat -f "%m" .planning/GOAL_PROPOSALS.md 2>/dev/null || stat -c "%Y" .planning/GOAL_PROPOSALS.md)

# Run /gsd:complete-milestone
# Expected: No suggestion refresh

AFTER=$(stat -f "%m" .planning/GOAL_PROPOSALS.md 2>/dev/null || stat -c "%Y" .planning/GOAL_PROPOSALS.md)
[ "$BEFORE" = "$AFTER" ] && echo "PASS: Suggestions not refreshed" || echo "FAIL: Suggestions were refreshed"
```

**Test 8c: Graceful degradation (never blocks milestone)**

```bash
# Simulate suggestion generation failure
# (e.g., remove analyze-codebase temporarily or cause an error)

# Run /gsd:complete-milestone with refresh_suggestions_on_milestone=true
# Expected: Milestone completion succeeds even if suggestion refresh fails
# Expected: Warning logged but not blocking

# Verify milestone completed
ls .planning/milestones/
```

### Verification Steps

1. **Verify opt-in behavior:**
   ```bash
   # Default is false (opt-in to refresh)
   grep "refresh_suggestions_on_milestone" get-shit-done/workflows/complete-milestone.md
   # Expected: Default behavior documented as false
   ```

2. **Verify non-blocking guarantee:**
   ```bash
   # Complete-milestone should never fail due to suggestion refresh
   # Check for error isolation in workflow
   grep -A 5 "refresh_suggestions" get-shit-done/workflows/complete-milestone.md
   # Expected: Error handling that catches failures and continues
   ```

### Success Criteria

- [ ] refresh_suggestions_on_milestone=true triggers refresh
- [ ] refresh_suggestions_on_milestone=false (default) skips refresh
- [ ] Refresh failure never blocks milestone completion
- [ ] Best-effort behavior with warning logging

---

## Test Scenario 9: End-to-End Pipeline

**Objective:** Verify complete flow from analyze-codebase through CLAUDE.md update works as integrated pipeline

### Prerequisites

- All v4.5 workflows installed
- Test project with package.json and some code

### Setup

```bash
# Clean slate
rm -f .planning/ANALYSIS_REPORT.md
rm -f .planning/GOAL_PROPOSALS.md

# Backup existing CLAUDE.md
cp CLAUDE.md CLAUDE.md.backup 2>/dev/null || true

# Enable full pipeline
cat > .planning/config.json << 'EOF'
{
  "suggestions": {
    "auto_suggest": true,
    "auto_update_claude_md": true
  }
}
EOF
```

### Test Execution

**Step 1: Run /gsd:analyze-codebase**

```bash
# This should trigger the full pipeline:
# analyze-codebase → generate-suggestions → update-claude-md

/gsd:analyze-codebase
```

**Step 2: Verify all intermediate files created**

```bash
# Check ANALYSIS_REPORT.md
ls .planning/ANALYSIS_REPORT.md
head -50 .planning/ANALYSIS_REPORT.md

# Check GOAL_PROPOSALS.md
ls .planning/GOAL_PROPOSALS.md
head -50 .planning/GOAL_PROPOSALS.md

# Check CLAUDE.md updated
grep "## Suggested Goals" CLAUDE.md
```

**Step 3: Test with real project data**

```bash
# Verify analysis reflects actual project state
npm audit --json | head -20  # Compare with ANALYSIS_REPORT.md
npm outdated --json | head -20  # Compare with ANALYSIS_REPORT.md

# Verify suggestions are actionable
grep -E "gsd:understand|gsd:plan-phase" .planning/GOAL_PROPOSALS.md
```

### Verification Steps

1. **Verify pipeline completed:**
   ```bash
   # All three output files should exist
   [ -f .planning/ANALYSIS_REPORT.md ] && echo "Analysis: PASS" || echo "Analysis: FAIL"
   [ -f .planning/GOAL_PROPOSALS.md ] && echo "Proposals: PASS" || echo "Proposals: FAIL"
   grep -q "## Suggested Goals" CLAUDE.md && echo "CLAUDE.md: PASS" || echo "CLAUDE.md: FAIL"
   ```

2. **Verify timestamp chain:**
   ```bash
   # ANALYSIS_REPORT.md Generated timestamp
   grep "Generated:" .planning/ANALYSIS_REPORT.md

   # GOAL_PROPOSALS.md Analysis Report timestamp (should match)
   grep "Analysis Report:" .planning/GOAL_PROPOSALS.md

   # CLAUDE.md last updated (should be recent)
   stat CLAUDE.md
   ```

3. **Verify data flows correctly:**
   ```bash
   # Count vulnerabilities in analysis
   VULN_COUNT=$(grep -c "CVE-" .planning/ANALYSIS_REPORT.md)

   # Check security suggestions generated
   grep "security-fix" .planning/GOAL_PROPOSALS.md

   # Verify suggestions appear in CLAUDE.md
   grep -c "Score:" CLAUDE.md
   ```

### Success Criteria

- [ ] /gsd:analyze-codebase triggers full pipeline
- [ ] All intermediate files created (ANALYSIS_REPORT.md, GOAL_PROPOSALS.md)
- [ ] CLAUDE.md updated with suggestions
- [ ] Data flows correctly through all stages
- [ ] Timestamps chain correctly
- [ ] Works with real project data (not just mocks)

---

## Test Scenario 10: Backward Compatibility

**Objective:** Verify existing v4.0/v3.0/v2.0 projects work without v4.5 features

### Test 10a: No config.json (defaults)

```bash
# Remove config entirely
mv .planning/config.json .planning/config.json.backup

# Run /gsd:analyze-codebase
# Expected: Works with default settings
# - auto_suggest: true (generates suggestions)
# - auto_update_claude_md: true (updates CLAUDE.md)

# Restore
mv .planning/config.json.backup .planning/config.json
```

### Test 10b: v4.0 config (no v4.5 options)

```bash
# v4.0 config without suggestions section
cat > .planning/config.json << 'EOF'
{
  "mode": "interactive",
  "retry": {
    "enabled": true
  },
  "verification": {
    "enabled": true
  },
  "adaptive": {
    "enabled": true,
    "prediction_enabled": true,
    "show_pattern_hints": true
  }
}
EOF

# Run /gsd:analyze-codebase
# Expected: Works with defaults for missing suggestions.* options
# Expected: No errors about missing configuration
```

### Test 10c: auto_suggest=false

```bash
cat > .planning/config.json << 'EOF'
{
  "suggestions": {
    "auto_suggest": false
  }
}
EOF

# Run /gsd:analyze-codebase
# Expected: ANALYSIS_REPORT.md created
# Expected: GOAL_PROPOSALS.md NOT auto-generated
# Expected: No error, just skips suggestion generation

ls .planning/ANALYSIS_REPORT.md  # Should exist
ls .planning/GOAL_PROPOSALS.md   # Should NOT exist (unless previously created)
```

### Test 10d: auto_update_claude_md=false

```bash
cat > .planning/config.json << 'EOF'
{
  "suggestions": {
    "auto_suggest": true,
    "auto_update_claude_md": false
  }
}
EOF

# Note CLAUDE.md current state
BEFORE=$(cat CLAUDE.md | md5sum)

# Run /gsd:suggest
# Expected: GOAL_PROPOSALS.md created/updated
# Expected: CLAUDE.md NOT updated

AFTER=$(cat CLAUDE.md | md5sum)
[ "$BEFORE" = "$AFTER" ] && echo "PASS: CLAUDE.md unchanged" || echo "FAIL: CLAUDE.md was modified"
```

### Test 10e: Verify no breaking changes to existing workflows

```bash
# Test that existing workflows still work
# /gsd:plan-phase should work normally
# /gsd:execute-plan should work normally
# /gsd:complete-milestone should work normally (without suggestion refresh unless opted in)

# Verify no errors in common operations
/gsd:progress  # Should show project status without errors
```

### Verification Steps

1. **Verify graceful defaults:**
   ```bash
   # All v4.5 features should have sensible defaults
   # auto_suggest: true
   # auto_update_claude_md: true
   # refresh_suggestions_on_milestone: false
   ```

2. **Verify no required migration:**
   ```bash
   # Existing projects should work without config changes
   # No "required field missing" errors
   ```

3. **Verify feature isolation:**
   ```bash
   # Disabling one v4.5 feature shouldn't affect others
   # e.g., auto_update_claude_md=false shouldn't break auto_suggest
   ```

### Success Criteria

- [ ] No config.json: Works with defaults
- [ ] v4.0 config: Works without suggestions section
- [ ] auto_suggest=false: Analysis works, suggestions skipped
- [ ] auto_update_claude_md=false: Suggestions work, CLAUDE.md skipped
- [ ] No warnings or errors for backward-compatible scenarios
- [ ] Primary workflows (plan, execute, milestone) unaffected

---

## Test Results Checklist

After running all scenarios, verify:

- [ ] **Scenario 1:** Dependency analyzer works with npm audit + deps.dev + outdated
- [ ] **Scenario 2:** Security analyzer extracts CVEs with HIGH/CRITICAL focus
- [ ] **Scenario 3:** Tech debt analyzer uses Knip with weighted scoring
- [ ] **Scenario 4:** Orchestrator runs analyzers in parallel, produces unified report
- [ ] **Scenario 5:** Suggestion engine applies scoring formula with pattern boosts
- [ ] **Scenario 6:** /gsd:suggest command handles staleness and CLAUDE.md updates
- [ ] **Scenario 7:** CLAUDE.md integration preserves other sections
- [ ] **Scenario 8:** Milestone auto-refresh respects opt-in configuration
- [ ] **Scenario 9:** End-to-end pipeline works from analysis to CLAUDE.md
- [ ] **Scenario 10:** Backward compatibility maintained with v4.0/v3.0 configs

**Overall Goal Setting System Health:**
- [ ] All analyzer workflows produce valid JSON output
- [ ] Priority hierarchy correctly orders findings
- [ ] Scoring formula calculates correct values
- [ ] CLAUDE.md format is compact and scannable
- [ ] All config flags work as documented
- [ ] Graceful degradation when tools unavailable
- [ ] Non-blocking guarantee honored (errors never block workflow)
- [ ] Backward compatibility 100% maintained

---

## Cleanup After Tests

```bash
# Remove test files
rm -rf .planning/test-workspace-goal-setting
rm -f .planning/ANALYSIS_REPORT.md
rm -f .planning/GOAL_PROPOSALS.md
rm -rf .planning/analysis/

# Restore CLAUDE.md if backed up
mv CLAUDE.md.backup CLAUDE.md 2>/dev/null || true

# Reset config
git checkout .planning/config.json 2>/dev/null || true

# Remove test workspace
rmdir .planning/test-workspace-goal-setting 2>/dev/null || true
```

---

## Troubleshooting

### Analyzers Not Running

**Check:**
```bash
# Verify workflows exist
ls get-shit-done/workflows/analyze-*.md

# Verify npm is available
npm --version

# Verify package.json exists
ls package.json
```

### Suggestions Not Generating

**Check:**
```bash
# Verify ANALYSIS_REPORT.md exists
ls .planning/ANALYSIS_REPORT.md

# Check auto_suggest config
grep "auto_suggest" .planning/config.json

# Verify generate-suggestions workflow exists
ls get-shit-done/workflows/generate-suggestions.md
```

### CLAUDE.md Not Updating

**Check:**
```bash
# Verify GOAL_PROPOSALS.md exists
ls .planning/GOAL_PROPOSALS.md

# Check auto_update_claude_md config
grep "auto_update_claude_md" .planning/config.json

# Verify update-claude-md workflow exists
ls get-shit-done/workflows/update-claude-md.md
```

### Staleness Detection Not Working

**Check:**
```bash
# Compare timestamps manually
grep "Generated:" .planning/ANALYSIS_REPORT.md
grep "Analysis Report:" .planning/GOAL_PROPOSALS.md

# Timestamps should match for "fresh" status
```

### Pattern Boosts Not Applied

**Check:**
```bash
# Verify knowledge base exists
ls ~/.claude/gsd-knowledge/index.json

# Verify adaptive.enabled = true
grep "adaptive" .planning/config.json

# Check for matching patterns
ls ~/.claude/gsd-knowledge/patterns/strategy-effectiveness/
```

---

## Summary

These tests validate that v4.5 Autonomous Goal Setting:

1. **Analyzes codebases** - Dependencies, security, tech debt with parallel execution
2. **Generates suggestions** - Scoring formula with effort modifiers and pattern boosts
3. **Updates CLAUDE.md** - Compact format with staleness detection
4. **Integrates with milestones** - Optional refresh on completion
5. **Maintains backward compatibility** - All config options have sensible defaults
6. **Degrades gracefully** - Missing tools don't block workflows
7. **Never blocks** - All features are best-effort, errors isolated

**Test coverage:** 10 scenarios covering all v4.5 autonomous goal setting features
**Execution:** Manual testing with bash commands
**Outcome:** Complete validation of v4.5 Autonomous Goal Setting milestone
