# Analyze Tech Debt Workflow

Tech debt detection for unused code, outdated packages, and structural issues.

## Purpose

<purpose>
Detect technical debt in the codebase:
1. Unused code - files, exports, and dependencies not being used
2. Outdated packages - version staleness and available updates
3. Structural issues - patterns indicating maintenance burden

This workflow focuses on structural tech debt (what exists but shouldn't) rather than code quality metrics (how code is written). It uses Knip for dead code detection and npm outdated for version analysis.

**Key principles:**
- Use established tools (Knip, npm) rather than hand-rolled AST traversal
- JSON output for machine-readable results
- Categorize findings for actionable prioritization
- Support projects without Knip config (run with defaults, note limitations)
</purpose>

## Prerequisites

<prerequisites>
  <required>
    - Node.js project with package.json
    - npm installed and available in PATH
  </required>

  <optional>
    - Knip installed (npx knip available)
    - knip.json for project-specific configuration
    - jq for JSON parsing (falls back to grep)
  </optional>
</prerequisites>

## Process

<step name="check_prerequisites" number="1">
**Verify tech debt analysis prerequisites**

```bash
# Check for package.json
if [ ! -f package.json ]; then
  echo "ERROR: No package.json found"
  echo "This workflow requires a Node.js project"
  exit 1
fi

# Check for jq availability
if command -v jq &> /dev/null; then
  HAS_JQ=true
else
  HAS_JQ=false
  echo "Note: jq not available, using basic JSON parsing"
fi

# Check for Knip availability
if npx knip --version &> /dev/null 2>&1; then
  HAS_KNIP=true
  echo "Knip available"
else
  HAS_KNIP=false
  echo "WARNING: Knip not available - dead code detection will be limited"
  echo "Install with: npm install -D knip"
fi

# Check for Knip configuration
if [ -f knip.json ] || [ -f knip.jsonc ] || grep -q '"knip"' package.json 2>/dev/null; then
  HAS_KNIP_CONFIG=true
  echo "Knip configuration found"
else
  HAS_KNIP_CONFIG=false
  echo "Note: No knip.json found, will use defaults (may produce false positives)"
fi

echo "Tech debt analysis prerequisites checked"
```
</step>

<step name="analyze_outdated" number="2">
**Check for outdated packages**

```bash
# Create temp directory for results
ANALYSIS_DIR="${ANALYSIS_DIR:-.planning/.analysis}"
mkdir -p "$ANALYSIS_DIR"

# Run npm outdated with JSON output
npm outdated --json > "$ANALYSIS_DIR/outdated.json" 2>/dev/null || true
# npm outdated exits non-zero if packages are outdated (expected behavior)

if [ "$HAS_JQ" = true ]; then
  # Parse outdated packages
  OUTDATED_COUNT=$(jq 'keys | length' "$ANALYSIS_DIR/outdated.json" 2>/dev/null || echo "0")

  # Categorize by semver impact
  jq -r '
    to_entries | map({
      package: .key,
      current: .value.current,
      wanted: .value.wanted,
      latest: .value.latest,
      type: .value.type,
      homepage: .value.homepage,
      updateType: (
        if (.value.current | split(".")[0]) != (.value.latest | split(".")[0])
        then "major"
        elif (.value.current | split(".")[1]) != (.value.latest | split(".")[1])
        then "minor"
        else "patch"
        end
      )
    })
  ' "$ANALYSIS_DIR/outdated.json" > "$ANALYSIS_DIR/outdated-categorized.json" 2>/dev/null

  # Count by category
  MAJOR_OUTDATED=$(jq '[.[] | select(.updateType == "major")] | length' "$ANALYSIS_DIR/outdated-categorized.json" 2>/dev/null || echo "0")
  MINOR_OUTDATED=$(jq '[.[] | select(.updateType == "minor")] | length' "$ANALYSIS_DIR/outdated-categorized.json" 2>/dev/null || echo "0")
  PATCH_OUTDATED=$(jq '[.[] | select(.updateType == "patch")] | length' "$ANALYSIS_DIR/outdated-categorized.json" 2>/dev/null || echo "0")

  # Identify significantly outdated (2+ major versions behind)
  jq -r '
    .[] | select(.updateType == "major") |
    select(
      ((.current | split(".")[0] | tonumber) + 2) <=
      (.latest | split(".")[0] | tonumber)
    ) | .package
  ' "$ANALYSIS_DIR/outdated-categorized.json" 2>/dev/null > "$ANALYSIS_DIR/significantly-outdated.txt" || true

  SIGNIFICANTLY_OUTDATED=$(wc -l < "$ANALYSIS_DIR/significantly-outdated.txt" 2>/dev/null | tr -d ' ' || echo "0")

else
  # Fallback: count JSON entries
  OUTDATED_COUNT=$(grep -c '"current":' "$ANALYSIS_DIR/outdated.json" 2>/dev/null || echo "0")
  MAJOR_OUTDATED="?"
  MINOR_OUTDATED="?"
  PATCH_OUTDATED="?"
  SIGNIFICANTLY_OUTDATED="?"
fi

echo "=== Outdated Packages ==="
echo "Total: $OUTDATED_COUNT"
echo "Major: $MAJOR_OUTDATED"
echo "Minor: $MINOR_OUTDATED"
echo "Patch: $PATCH_OUTDATED"
echo "Significantly outdated (2+ major): $SIGNIFICANTLY_OUTDATED"
```

**Output:** Categorized outdated package data
</step>

<step name="run_knip_analysis" number="3">
**Run Knip for dead code detection**

```bash
if [ "$HAS_KNIP" = true ]; then
  echo "Running Knip analysis..."

  # Run Knip with JSON reporter
  # Use timeout to prevent hanging on large projects
  timeout 120 npx knip --reporter json > "$ANALYSIS_DIR/knip.json" 2>"$ANALYSIS_DIR/knip-stderr.txt" || true
  KNIP_EXIT=$?

  # Check for timeout or error
  if [ $KNIP_EXIT -eq 124 ]; then
    echo "WARNING: Knip timed out after 120s"
    echo '{"files":[],"dependencies":[],"devDependencies":[],"optionalPeerDependencies":[],"exports":[],"types":[],"duplicates":[]}' > "$ANALYSIS_DIR/knip.json"
    KNIP_TIMEDOUT=true
  elif [ ! -s "$ANALYSIS_DIR/knip.json" ]; then
    echo "WARNING: Knip produced no output"
    echo '{"files":[],"dependencies":[],"devDependencies":[],"optionalPeerDependencies":[],"exports":[],"types":[],"duplicates":[]}' > "$ANALYSIS_DIR/knip.json"
    KNIP_EMPTY=true
  fi

  if [ "$HAS_JQ" = true ]; then
    # Parse Knip results
    UNUSED_FILES=$(jq '.files | length' "$ANALYSIS_DIR/knip.json" 2>/dev/null || echo "0")
    UNUSED_DEPS=$(jq '.dependencies | length' "$ANALYSIS_DIR/knip.json" 2>/dev/null || echo "0")
    UNUSED_DEV_DEPS=$(jq '.devDependencies | length' "$ANALYSIS_DIR/knip.json" 2>/dev/null || echo "0")
    UNUSED_EXPORTS=$(jq '.exports | length' "$ANALYSIS_DIR/knip.json" 2>/dev/null || echo "0")
    UNUSED_TYPES=$(jq '.types | length' "$ANALYSIS_DIR/knip.json" 2>/dev/null || echo "0")
    DUPLICATES=$(jq '.duplicates | length' "$ANALYSIS_DIR/knip.json" 2>/dev/null || echo "0")

    # Extract file lists for report
    jq -r '.files[]' "$ANALYSIS_DIR/knip.json" 2>/dev/null > "$ANALYSIS_DIR/unused-files.txt" || true
    jq -r '.dependencies[]' "$ANALYSIS_DIR/knip.json" 2>/dev/null > "$ANALYSIS_DIR/unused-deps.txt" || true
    jq -r '.exports[] | "\(.name) in \(.pos.filePath)"' "$ANALYSIS_DIR/knip.json" 2>/dev/null > "$ANALYSIS_DIR/unused-exports.txt" || true

  else
    # Fallback: basic counting
    UNUSED_FILES=$(grep -c '"files":' "$ANALYSIS_DIR/knip.json" 2>/dev/null || echo "?")
    UNUSED_DEPS=$(grep -c '"dependencies":' "$ANALYSIS_DIR/knip.json" 2>/dev/null || echo "?")
    UNUSED_EXPORTS=$(grep -c '"exports":' "$ANALYSIS_DIR/knip.json" 2>/dev/null || echo "?")
    UNUSED_DEV_DEPS="?"
    UNUSED_TYPES="?"
    DUPLICATES="?"
  fi

  echo "=== Knip Analysis ==="
  echo "Unused files: $UNUSED_FILES"
  echo "Unused dependencies: $UNUSED_DEPS"
  echo "Unused devDependencies: $UNUSED_DEV_DEPS"
  echo "Unused exports: $UNUSED_EXPORTS"
  echo "Unused types: $UNUSED_TYPES"
  echo "Duplicates: $DUPLICATES"

else
  # No Knip available
  UNUSED_FILES=0
  UNUSED_DEPS=0
  UNUSED_DEV_DEPS=0
  UNUSED_EXPORTS=0
  UNUSED_TYPES=0
  DUPLICATES=0

  echo "=== Knip Analysis ==="
  echo "SKIPPED: Knip not available"
  echo "Install with: npm install -D knip"
fi
```

**Output:**
- `knip.json` - Full Knip output
- `unused-files.txt` - List of unused files
- `unused-deps.txt` - List of unused dependencies
- `unused-exports.txt` - List of unused exports
</step>

<step name="calculate_debt_score" number="4">
**Calculate tech debt indicators**

```bash
# Calculate weighted tech debt score
# Weights reflect impact:
# - Unused dependencies: 10 points each (bloat, security surface)
# - Unused files: 5 points each (maintenance burden)
# - Unused exports: 2 points each (code complexity)
# - Major outdated: 3 points each (maintenance risk)
# - Significantly outdated: 8 points each (serious risk)

if [ "$HAS_JQ" = true ]; then
  UNUSED_DEPS_INT=${UNUSED_DEPS:-0}
  UNUSED_FILES_INT=${UNUSED_FILES:-0}
  UNUSED_EXPORTS_INT=${UNUSED_EXPORTS:-0}
  MAJOR_OUTDATED_INT=${MAJOR_OUTDATED:-0}
  SIG_OUTDATED_INT=${SIGNIFICANTLY_OUTDATED:-0}

  # Handle "?" values
  [ "$UNUSED_DEPS_INT" = "?" ] && UNUSED_DEPS_INT=0
  [ "$UNUSED_FILES_INT" = "?" ] && UNUSED_FILES_INT=0
  [ "$UNUSED_EXPORTS_INT" = "?" ] && UNUSED_EXPORTS_INT=0
  [ "$MAJOR_OUTDATED_INT" = "?" ] && MAJOR_OUTDATED_INT=0
  [ "$SIG_OUTDATED_INT" = "?" ] && SIG_OUTDATED_INT=0

  DEBT_SCORE=$((
    UNUSED_DEPS_INT * 10 +
    UNUSED_FILES_INT * 5 +
    UNUSED_EXPORTS_INT * 2 +
    MAJOR_OUTDATED_INT * 3 +
    SIG_OUTDATED_INT * 8
  ))

  # Determine severity level
  if [ $DEBT_SCORE -ge 100 ]; then
    DEBT_LEVEL="HIGH"
  elif [ $DEBT_SCORE -ge 50 ]; then
    DEBT_LEVEL="MEDIUM"
  elif [ $DEBT_SCORE -gt 0 ]; then
    DEBT_LEVEL="LOW"
  else
    DEBT_LEVEL="CLEAN"
  fi
else
  DEBT_SCORE="?"
  DEBT_LEVEL="UNKNOWN"
fi

echo "=== Tech Debt Score ==="
echo "Score: $DEBT_SCORE"
echo "Level: $DEBT_LEVEL"
```

**Scoring interpretation:**
- 0: Clean - no detected tech debt
- 1-49: Low - minor cleanup opportunities
- 50-99: Medium - should address in next maintenance cycle
- 100+: High - significant debt requiring attention
</step>

<step name="generate_findings" number="5">
**Generate structured tech debt findings**

```bash
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

# Determine action priority
if [ "$DEBT_LEVEL" = "HIGH" ]; then
  ACTION_PRIORITY="RECOMMENDED"
elif [ "$DEBT_LEVEL" = "MEDIUM" ]; then
  ACTION_PRIORITY="SUGGESTED"
else
  ACTION_PRIORITY="OPTIONAL"
fi

# Create findings output
cat > "$ANALYSIS_DIR/tech-debt-findings.json" << EOF
{
  "timestamp": "$TIMESTAMP",
  "type": "tech-debt-analysis",
  "summary": {
    "unused_code": {
      "files": $UNUSED_FILES,
      "dependencies": $UNUSED_DEPS,
      "dev_dependencies": $UNUSED_DEV_DEPS,
      "exports": $UNUSED_EXPORTS,
      "types": $UNUSED_TYPES
    },
    "outdated": {
      "total": $OUTDATED_COUNT,
      "major": $MAJOR_OUTDATED,
      "minor": $MINOR_OUTDATED,
      "patch": $PATCH_OUTDATED,
      "significantly_outdated": $SIGNIFICANTLY_OUTDATED
    },
    "duplicates": $DUPLICATES
  },
  "score": $DEBT_SCORE,
  "level": "$DEBT_LEVEL",
  "priority": "$ACTION_PRIORITY",
  "knip_available": $HAS_KNIP,
  "knip_configured": $HAS_KNIP_CONFIG
}
EOF

echo "=== Tech Debt Analysis Complete ==="
echo "Score: $DEBT_SCORE ($DEBT_LEVEL)"
echo "Priority: $ACTION_PRIORITY"
echo "Findings saved to: $ANALYSIS_DIR/tech-debt-findings.json"
```

**Output:** Structured tech debt findings JSON for orchestrator
</step>

<step name="generate_report" number="6">
**Generate tech debt report**

```bash
cat > "$ANALYSIS_DIR/tech-debt-report.md" << EOF
# Tech Debt Analysis Report

**Generated:** $TIMESTAMP
**Project:** $(basename $(pwd))
**Debt Level:** $DEBT_LEVEL (score: $DEBT_SCORE)

---

## Summary

| Category | Count |
|----------|-------|
| Unused files | $UNUSED_FILES |
| Unused dependencies | $UNUSED_DEPS |
| Unused devDependencies | $UNUSED_DEV_DEPS |
| Unused exports | $UNUSED_EXPORTS |
| Outdated packages | $OUTDATED_COUNT |
| Significantly outdated | $SIGNIFICANTLY_OUTDATED |

---

## Unused Code (Knip Analysis)

EOF

if [ "$HAS_KNIP" = false ]; then
  echo "**Knip not available.** Install to enable dead code detection:" >> "$ANALYSIS_DIR/tech-debt-report.md"
  echo "\`\`\`bash" >> "$ANALYSIS_DIR/tech-debt-report.md"
  echo "npm install -D knip" >> "$ANALYSIS_DIR/tech-debt-report.md"
  echo "\`\`\`" >> "$ANALYSIS_DIR/tech-debt-report.md"
  echo "" >> "$ANALYSIS_DIR/tech-debt-report.md"
elif [ "$HAS_KNIP_CONFIG" = false ]; then
  echo "**Note:** No knip.json configuration found. Results may include false positives." >> "$ANALYSIS_DIR/tech-debt-report.md"
  echo "Consider adding project-specific configuration." >> "$ANALYSIS_DIR/tech-debt-report.md"
  echo "" >> "$ANALYSIS_DIR/tech-debt-report.md"
fi

if [ "$UNUSED_FILES" -gt 0 ] && [ -f "$ANALYSIS_DIR/unused-files.txt" ] && [ -s "$ANALYSIS_DIR/unused-files.txt" ]; then
  echo "### Unused Files ($UNUSED_FILES)" >> "$ANALYSIS_DIR/tech-debt-report.md"
  echo "" >> "$ANALYSIS_DIR/tech-debt-report.md"
  head -20 "$ANALYSIS_DIR/unused-files.txt" | while read -r file; do
    echo "- \`$file\`" >> "$ANALYSIS_DIR/tech-debt-report.md"
  done
  if [ $(wc -l < "$ANALYSIS_DIR/unused-files.txt") -gt 20 ]; then
    echo "- ... and $(($(wc -l < "$ANALYSIS_DIR/unused-files.txt") - 20)) more" >> "$ANALYSIS_DIR/tech-debt-report.md"
  fi
  echo "" >> "$ANALYSIS_DIR/tech-debt-report.md"
fi

if [ "$UNUSED_DEPS" -gt 0 ] && [ -f "$ANALYSIS_DIR/unused-deps.txt" ] && [ -s "$ANALYSIS_DIR/unused-deps.txt" ]; then
  echo "### Unused Dependencies ($UNUSED_DEPS)" >> "$ANALYSIS_DIR/tech-debt-report.md"
  echo "" >> "$ANALYSIS_DIR/tech-debt-report.md"
  echo "These packages are in package.json but not imported anywhere:" >> "$ANALYSIS_DIR/tech-debt-report.md"
  echo "" >> "$ANALYSIS_DIR/tech-debt-report.md"
  head -20 "$ANALYSIS_DIR/unused-deps.txt" | while read -r dep; do
    echo "- \`$dep\`" >> "$ANALYSIS_DIR/tech-debt-report.md"
  done
  echo "" >> "$ANALYSIS_DIR/tech-debt-report.md"
  echo "**Cleanup:** \`npm uninstall $(head -5 "$ANALYSIS_DIR/unused-deps.txt" | tr '\n' ' ')\`" >> "$ANALYSIS_DIR/tech-debt-report.md"
  echo "" >> "$ANALYSIS_DIR/tech-debt-report.md"
fi

if [ "$UNUSED_EXPORTS" -gt 0 ] && [ -f "$ANALYSIS_DIR/unused-exports.txt" ] && [ -s "$ANALYSIS_DIR/unused-exports.txt" ]; then
  echo "### Unused Exports ($UNUSED_EXPORTS)" >> "$ANALYSIS_DIR/tech-debt-report.md"
  echo "" >> "$ANALYSIS_DIR/tech-debt-report.md"
  echo "Exports that are not imported anywhere:" >> "$ANALYSIS_DIR/tech-debt-report.md"
  echo "" >> "$ANALYSIS_DIR/tech-debt-report.md"
  head -15 "$ANALYSIS_DIR/unused-exports.txt" | while read -r export; do
    echo "- $export" >> "$ANALYSIS_DIR/tech-debt-report.md"
  done
  if [ $(wc -l < "$ANALYSIS_DIR/unused-exports.txt") -gt 15 ]; then
    echo "- ... and $(($(wc -l < "$ANALYSIS_DIR/unused-exports.txt") - 15)) more" >> "$ANALYSIS_DIR/tech-debt-report.md"
  fi
  echo "" >> "$ANALYSIS_DIR/tech-debt-report.md"
fi

cat >> "$ANALYSIS_DIR/tech-debt-report.md" << EOF

---

## Outdated Packages

| Update Type | Count |
|-------------|-------|
| Major | $MAJOR_OUTDATED |
| Minor | $MINOR_OUTDATED |
| Patch | $PATCH_OUTDATED |
| **Total** | **$OUTDATED_COUNT** |

EOF

if [ "$SIGNIFICANTLY_OUTDATED" != "?" ] && [ "$SIGNIFICANTLY_OUTDATED" -gt 0 ]; then
  echo "### Significantly Outdated (2+ major versions)" >> "$ANALYSIS_DIR/tech-debt-report.md"
  echo "" >> "$ANALYSIS_DIR/tech-debt-report.md"
  echo "These packages are significantly behind and may have breaking changes:" >> "$ANALYSIS_DIR/tech-debt-report.md"
  echo "" >> "$ANALYSIS_DIR/tech-debt-report.md"
  while read -r pkg; do
    [ -n "$pkg" ] && echo "- \`$pkg\`" >> "$ANALYSIS_DIR/tech-debt-report.md"
  done < "$ANALYSIS_DIR/significantly-outdated.txt"
  echo "" >> "$ANALYSIS_DIR/tech-debt-report.md"
fi

cat >> "$ANALYSIS_DIR/tech-debt-report.md" << EOF

---

## Recommendations

EOF

# Generate recommendations based on debt level
case "$DEBT_LEVEL" in
  "HIGH")
    cat >> "$ANALYSIS_DIR/tech-debt-report.md" << EOF
### Priority: HIGH

The codebase has significant tech debt. Consider:

1. **Remove unused dependencies** - reduces install time, security surface
   \`\`\`bash
   # Review first, then remove
   npm uninstall <packages>
   \`\`\`

2. **Delete unused files** - reduces complexity, improves navigation
   - Review \`unused-files.txt\` before deleting
   - Some may be dynamically loaded

3. **Update significantly outdated packages** - reduces security risk
   \`\`\`bash
   npx npm-check-updates -i  # Interactive update
   \`\`\`

EOF
    ;;

  "MEDIUM")
    cat >> "$ANALYSIS_DIR/tech-debt-report.md" << EOF
### Priority: MEDIUM

The codebase has moderate tech debt. Consider addressing during next maintenance cycle:

1. **Unused dependencies** - quick win for cleaner package.json
2. **Major version updates** - schedule for testing
3. **Unused exports** - cleanup when touching affected files

EOF
    ;;

  "LOW")
    cat >> "$ANALYSIS_DIR/tech-debt-report.md" << EOF
### Priority: LOW

The codebase is in good shape. Minor cleanup opportunities:

1. Consider cleaning up unused exports as part of regular development
2. Keep dependencies reasonably up to date

EOF
    ;;

  "CLEAN")
    cat >> "$ANALYSIS_DIR/tech-debt-report.md" << EOF
### All Clear!

No significant tech debt detected. Keep up the good maintenance!

EOF
    ;;
esac

# Knip configuration recommendation if missing
if [ "$HAS_KNIP" = true ] && [ "$HAS_KNIP_CONFIG" = false ]; then
  cat >> "$ANALYSIS_DIR/tech-debt-report.md" << EOF

### Knip Configuration

Consider adding \`knip.json\` for more accurate results:

\`\`\`json
{
  "entry": ["src/index.ts"],
  "project": ["src/**/*.ts"],
  "ignore": ["**/*.test.ts", "**/*.spec.ts"],
  "ignoreDependencies": ["@types/*"]
}
\`\`\`

See [Knip documentation](https://knip.dev/) for configuration options.

EOF
fi

cat >> "$ANALYSIS_DIR/tech-debt-report.md" << EOF

---

## Debt Score Calculation

| Category | Count | Weight | Points |
|----------|-------|--------|--------|
| Unused dependencies | $UNUSED_DEPS | ×10 | $((UNUSED_DEPS * 10)) |
| Unused files | $UNUSED_FILES | ×5 | $((UNUSED_FILES * 5)) |
| Unused exports | $UNUSED_EXPORTS | ×2 | $((UNUSED_EXPORTS * 2)) |
| Major outdated | $MAJOR_OUTDATED | ×3 | $((MAJOR_OUTDATED * 3)) |
| Significantly outdated | $SIGNIFICANTLY_OUTDATED | ×8 | $((SIGNIFICANTLY_OUTDATED * 8)) |
| **Total** | | | **$DEBT_SCORE** |

**Thresholds:** Clean (0) | Low (1-49) | Medium (50-99) | High (100+)

---

*Generated by analyze-tech-debt workflow*
EOF

echo "Report saved to: $ANALYSIS_DIR/tech-debt-report.md"
```

**Output:** Detailed tech debt report with recommendations
</step>

## Output Format

<output>
The workflow produces:

**Primary outputs:**
1. **tech-debt-findings.json** - Machine-readable findings for orchestrator
2. **tech-debt-report.md** - Human-readable analysis report

**Supporting outputs:**
- `outdated.json` - Raw npm outdated output
- `outdated-categorized.json` - Categorized by update type
- `significantly-outdated.txt` - Packages 2+ major versions behind
- `knip.json` - Full Knip output (if available)
- `unused-files.txt` - List of unused files
- `unused-deps.txt` - List of unused dependencies
- `unused-exports.txt` - List of unused exports

**Findings JSON structure:**
```json
{
  "timestamp": "2026-01-08T10:30:00Z",
  "type": "tech-debt-analysis",
  "summary": {
    "unused_code": {
      "files": 5,
      "dependencies": 3,
      "dev_dependencies": 2,
      "exports": 15,
      "types": 0
    },
    "outdated": {
      "total": 12,
      "major": 4,
      "minor": 5,
      "patch": 3,
      "significantly_outdated": 2
    },
    "duplicates": 0
  },
  "score": 85,
  "level": "MEDIUM",
  "priority": "SUGGESTED",
  "knip_available": true,
  "knip_configured": true
}
```

**Level values:**
- `CLEAN` - No detected tech debt (score 0)
- `LOW` - Minor cleanup opportunities (score 1-49)
- `MEDIUM` - Should address in maintenance cycle (score 50-99)
- `HIGH` - Significant debt requiring attention (score 100+)
- `UNKNOWN` - Could not calculate (missing tools)

**Priority values:**
- `OPTIONAL` - Clean or low debt
- `SUGGESTED` - Medium debt, recommended cleanup
- `RECOMMENDED` - High debt, should prioritize
</output>

## Error Handling

<error_handling>
**Graceful degradation:**
- If Knip unavailable: Skip dead code detection, note in findings
- If jq unavailable: Fall back to basic JSON parsing
- If Knip times out: Report empty results, note timeout
- If npm outdated fails: Continue with other analyses

**Non-blocking failures:**
- Knip errors → Log warning, continue with outdated analysis
- Malformed Knip output → Use empty defaults

**Hard failures (workflow exits):**
- No package.json → Cannot analyze, exit with clear message
</error_handling>

## Integration

<integration>
**Called by:**
- `analyze-codebase.md` orchestrator (Phase 11-02)
- Manual invocation for tech debt audits

**Feeds into:**
- Suggestion engine (Phase 12) for cleanup goal proposals
- CLAUDE.md maintenance suggestions

**Difference from analyze-dependencies.md:**
- analyze-dependencies: Security focus with vulnerability detection
- analyze-tech-debt: Maintenance focus with unused code and staleness

**Example orchestrator integration:**
```bash
# Run tech debt analysis
source analyze-tech-debt.md

# Check debt level
LEVEL=$(jq -r '.level' "$ANALYSIS_DIR/tech-debt-findings.json")
if [ "$LEVEL" = "HIGH" ]; then
  echo "suggestion: Prioritize tech debt cleanup"
fi
```
</integration>

## Examples

### Example 1: Clean Codebase

**Scenario:** Well-maintained project with no tech debt

**Output:**
```
=== Outdated Packages ===
Total: 3
Major: 0
Minor: 2
Patch: 1
Significantly outdated (2+ major): 0

=== Knip Analysis ===
Unused files: 0
Unused dependencies: 0
Unused devDependencies: 0
Unused exports: 0
Unused types: 0
Duplicates: 0

=== Tech Debt Score ===
Score: 0
Level: CLEAN

=== Tech Debt Analysis Complete ===
Score: 0 (CLEAN)
Priority: OPTIONAL
```

---

### Example 2: High Tech Debt

**Scenario:** Project with accumulated tech debt

**Output:**
```
=== Outdated Packages ===
Total: 25
Major: 8
Minor: 10
Patch: 7
Significantly outdated (2+ major): 3

=== Knip Analysis ===
Unused files: 12
Unused dependencies: 5
Unused devDependencies: 3
Unused exports: 35
Unused types: 2
Duplicates: 1

=== Tech Debt Score ===
Score: 164
Level: HIGH

=== Tech Debt Analysis Complete ===
Score: 164 (HIGH)
Priority: RECOMMENDED
```

**Recommendations generated:**
1. URGENT: Remove 5 unused dependencies
2. HIGH: Delete 12 unused files
3. HIGH: Update 3 significantly outdated packages

---

### Example 3: No Knip Available

**Scenario:** Project without Knip installed

**Output:**
```
WARNING: Knip not available - dead code detection will be limited
Install with: npm install -D knip

=== Outdated Packages ===
Total: 10
Major: 3
Minor: 4
Patch: 3
Significantly outdated (2+ major): 1

=== Knip Analysis ===
SKIPPED: Knip not available

=== Tech Debt Score ===
Score: 17
Level: LOW

=== Tech Debt Analysis Complete ===
Score: 17 (LOW)
Priority: OPTIONAL
```

---

*Tech debt analysis workflow for GSD autonomous goal setting*
