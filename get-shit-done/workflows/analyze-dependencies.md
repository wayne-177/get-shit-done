# Analyze Dependencies Workflow

Dependency analysis for vulnerability detection and package health assessment.

## Purpose

<purpose>
Analyze project dependencies to identify:
1. Security vulnerabilities in production dependencies
2. Outdated packages with available updates
3. Package health and maintenance status

This workflow provides the detection layer for dependency-related issues, feeding into the analyze-codebase.md orchestrator (Phase 11-02) and ultimately the suggestion engine (Phase 12).

**Key principles:**
- Focus on production dependencies (`--omit=dev`) to avoid false positive fatigue
- Use JSON output (`--json`) for machine-readable results
- Don't hand-roll severity scoring - use npm audit's existing categories
- Filter to HIGH/CRITICAL severity for actionable findings
</purpose>

## Prerequisites

<prerequisites>
  <required>
    - Node.js project with package.json
    - npm installed and available in PATH
    - Internet connection for advisory lookups
  </required>

  <optional>
    - jq installed for JSON parsing (falls back to grep patterns if unavailable)
    - curl for deps.dev API enrichment
  </optional>
</prerequisites>

## Process

<step name="check_prerequisites" number="1">
**Verify project has npm dependencies**

```bash
# Check for package.json
if [ ! -f package.json ]; then
  echo "ERROR: No package.json found"
  echo "This workflow requires a Node.js project"
  exit 1
fi

# Check for node_modules (dependencies installed)
if [ ! -d node_modules ]; then
  echo "WARNING: node_modules not found - dependencies may not be installed"
  echo "Consider running: npm install"
fi

# Check for jq availability
if command -v jq &> /dev/null; then
  HAS_JQ=true
else
  HAS_JQ=false
  echo "Note: jq not available, using basic JSON parsing"
fi

echo "Prerequisites satisfied"
```

**Error handling:**
- No package.json → Exit with clear message
- No node_modules → Warn but continue (npm audit still works)
- No jq → Fall back to grep-based parsing
</step>

<step name="run_npm_audit" number="2">
**Execute npm audit for vulnerability scanning**

```bash
# Create temp directory for results
ANALYSIS_DIR="${ANALYSIS_DIR:-.planning/.analysis}"
mkdir -p "$ANALYSIS_DIR"

# Run npm audit with JSON output, production deps only
# Capture exit code separately (non-zero = vulnerabilities found)
npm audit --json --omit=dev > "$ANALYSIS_DIR/audit.json" 2>/dev/null
AUDIT_EXIT=$?

# Exit code interpretation:
# 0 = no vulnerabilities
# 1+ = vulnerabilities found (count varies by npm version)

if [ ! -s "$ANALYSIS_DIR/audit.json" ]; then
  echo "WARNING: npm audit returned empty result"
  echo '{"vulnerabilities":{}}' > "$ANALYSIS_DIR/audit.json"
fi

echo "npm audit complete (exit code: $AUDIT_EXIT)"
```

**Output:** `$ANALYSIS_DIR/audit.json` containing vulnerability data
</step>

<step name="parse_vulnerabilities" number="3">
**Extract and categorize vulnerabilities**

```bash
AUDIT_FILE="$ANALYSIS_DIR/audit.json"

if [ "$HAS_JQ" = true ]; then
  # Count vulnerabilities by severity
  CRITICAL=$(jq '[.vulnerabilities[] | select(.severity == "critical")] | length' "$AUDIT_FILE" 2>/dev/null || echo "0")
  HIGH=$(jq '[.vulnerabilities[] | select(.severity == "high")] | length' "$AUDIT_FILE" 2>/dev/null || echo "0")
  MODERATE=$(jq '[.vulnerabilities[] | select(.severity == "moderate")] | length' "$AUDIT_FILE" 2>/dev/null || echo "0")
  LOW=$(jq '[.vulnerabilities[] | select(.severity == "low")] | length' "$AUDIT_FILE" 2>/dev/null || echo "0")

  # Get total vulnerability count
  TOTAL_VULNS=$(jq '.vulnerabilities | keys | length' "$AUDIT_FILE" 2>/dev/null || echo "0")

  # Extract details for high/critical only (actionable findings)
  jq -r '
    .vulnerabilities | to_entries[] |
    select(.value.severity == "critical" or .value.severity == "high") |
    {
      package: .key,
      severity: .value.severity,
      title: (.value.via[0].title // "Unknown vulnerability"),
      url: (.value.via[0].url // ""),
      fixAvailable: .value.fixAvailable,
      range: .value.range
    }
  ' "$AUDIT_FILE" > "$ANALYSIS_DIR/high_severity.json" 2>/dev/null

else
  # Fallback: basic grep parsing
  CRITICAL=$(grep -c '"severity":"critical"' "$AUDIT_FILE" 2>/dev/null || echo "0")
  HIGH=$(grep -c '"severity":"high"' "$AUDIT_FILE" 2>/dev/null || echo "0")
  MODERATE=$(grep -c '"severity":"moderate"' "$AUDIT_FILE" 2>/dev/null || echo "0")
  LOW=$(grep -c '"severity":"low"' "$AUDIT_FILE" 2>/dev/null || echo "0")
  TOTAL_VULNS=$((CRITICAL + HIGH + MODERATE + LOW))
fi

# Summary
echo "=== Vulnerability Summary ==="
echo "Critical: $CRITICAL"
echo "High: $HIGH"
echo "Moderate: $MODERATE"
echo "Low: $LOW"
echo "Total: $TOTAL_VULNS"
```

**Output:** Vulnerability counts by severity, detailed findings in `high_severity.json`
</step>

<step name="check_outdated" number="4">
**Check for outdated packages**

```bash
# Run npm outdated with JSON output
npm outdated --json > "$ANALYSIS_DIR/outdated.json" 2>/dev/null || true

# npm outdated exits non-zero if outdated packages exist, which is expected

if [ "$HAS_JQ" = true ]; then
  # Count outdated packages
  OUTDATED_COUNT=$(jq 'keys | length' "$ANALYSIS_DIR/outdated.json" 2>/dev/null || echo "0")

  # Categorize by update type (major, minor, patch)
  # Compare current vs wanted vs latest
  jq -r '
    to_entries[] |
    {
      package: .key,
      current: .value.current,
      wanted: .value.wanted,
      latest: .value.latest,
      type: .value.type,
      updateType: (
        if (.value.current | split(".")[0]) != (.value.latest | split(".")[0])
        then "major"
        elif (.value.current | split(".")[1]) != (.value.latest | split(".")[1])
        then "minor"
        else "patch"
        end
      )
    }
  ' "$ANALYSIS_DIR/outdated.json" > "$ANALYSIS_DIR/outdated_details.json" 2>/dev/null

  MAJOR_UPDATES=$(jq '[.[] | select(.updateType == "major")] | length' "$ANALYSIS_DIR/outdated_details.json" 2>/dev/null || echo "0")
  MINOR_UPDATES=$(jq '[.[] | select(.updateType == "minor")] | length' "$ANALYSIS_DIR/outdated_details.json" 2>/dev/null || echo "0")
  PATCH_UPDATES=$(jq '[.[] | select(.updateType == "patch")] | length' "$ANALYSIS_DIR/outdated_details.json" 2>/dev/null || echo "0")

else
  # Fallback: count entries in JSON
  OUTDATED_COUNT=$(grep -c '"current":' "$ANALYSIS_DIR/outdated.json" 2>/dev/null || echo "0")
  MAJOR_UPDATES="?"
  MINOR_UPDATES="?"
  PATCH_UPDATES="?"
fi

echo "=== Outdated Packages ==="
echo "Total outdated: $OUTDATED_COUNT"
echo "Major updates: $MAJOR_UPDATES"
echo "Minor updates: $MINOR_UPDATES"
echo "Patch updates: $PATCH_UPDATES"
```

**Output:** Outdated package counts, detailed version info in `outdated_details.json`
</step>

<step name="enrich_with_deps_dev" number="5">
**Optionally enrich high-severity findings with deps.dev metadata**

This step provides additional context for critical vulnerabilities from Google's deps.dev API.

```bash
# Only run if curl available and we have high-severity vulnerabilities
if command -v curl &> /dev/null && [ -f "$ANALYSIS_DIR/high_severity.json" ] && [ -s "$ANALYSIS_DIR/high_severity.json" ]; then

  echo "Enriching findings with deps.dev metadata..."

  # Create enriched output file
  > "$ANALYSIS_DIR/enriched_vulns.json"

  if [ "$HAS_JQ" = true ]; then
    # Extract package names from high severity findings
    PACKAGES=$(jq -r '.package' "$ANALYSIS_DIR/high_severity.json" 2>/dev/null | head -10)

    for pkg in $PACKAGES; do
      # Get latest version info from deps.dev
      # Note: This is best-effort, failures are non-blocking
      VERSION=$(jq -r --arg pkg "$pkg" '.dependencies[$pkg].version // empty' package-lock.json 2>/dev/null || echo "")

      if [ -n "$VERSION" ]; then
        DEPS_INFO=$(curl -s "https://api.deps.dev/v3/systems/npm/packages/$pkg/versions/$VERSION" 2>/dev/null | head -c 5000)

        if [ -n "$DEPS_INFO" ]; then
          # Extract advisory info if available
          echo "{\"package\":\"$pkg\",\"deps_dev\":$DEPS_INFO}" >> "$ANALYSIS_DIR/enriched_vulns.json"
        fi
      fi

      # Rate limiting - be nice to the API
      sleep 0.2
    done
  fi

  echo "deps.dev enrichment complete"
else
  echo "Skipping deps.dev enrichment (curl unavailable or no high-severity findings)"
fi
```

**Note:** deps.dev enrichment is optional and best-effort. Failures do not block the workflow.
</step>

<step name="generate_findings" number="6">
**Generate structured findings list**

```bash
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

# Create findings output
cat > "$ANALYSIS_DIR/dependency-findings.json" << EOF
{
  "timestamp": "$TIMESTAMP",
  "type": "dependency-analysis",
  "summary": {
    "vulnerabilities": {
      "total": $TOTAL_VULNS,
      "critical": $CRITICAL,
      "high": $HIGH,
      "moderate": $MODERATE,
      "low": $LOW
    },
    "outdated": {
      "total": $OUTDATED_COUNT,
      "major": $MAJOR_UPDATES,
      "minor": $MINOR_UPDATES,
      "patch": $PATCH_UPDATES
    }
  },
  "status": "$([ $((CRITICAL + HIGH)) -gt 0 ] && echo "ACTION_REQUIRED" || echo "OK")",
  "priority": "$([ "$CRITICAL" -gt 0 ] && echo "CRITICAL" || ([ "$HIGH" -gt 0 ] && echo "HIGH" || echo "NORMAL"))"
}
EOF

echo "=== Analysis Complete ==="
echo "Findings saved to: $ANALYSIS_DIR/dependency-findings.json"
echo "Status: $([ $((CRITICAL + HIGH)) -gt 0 ] && echo "ACTION_REQUIRED" || echo "OK")"
```

**Output:** Structured findings JSON suitable for orchestrator consumption
</step>

<step name="format_report" number="7">
**Generate human-readable report (optional)**

```bash
# Generate markdown report for human consumption
cat > "$ANALYSIS_DIR/dependency-report.md" << EOF
# Dependency Analysis Report

**Generated:** $TIMESTAMP
**Project:** $(basename $(pwd))

---

## Security Vulnerabilities

| Severity | Count |
|----------|-------|
| Critical | $CRITICAL |
| High | $HIGH |
| Moderate | $MODERATE |
| Low | $LOW |
| **Total** | **$TOTAL_VULNS** |

EOF

# Add high-severity details if present
if [ "$HAS_JQ" = true ] && [ -f "$ANALYSIS_DIR/high_severity.json" ] && [ -s "$ANALYSIS_DIR/high_severity.json" ]; then
  echo "### High-Severity Findings" >> "$ANALYSIS_DIR/dependency-report.md"
  echo "" >> "$ANALYSIS_DIR/dependency-report.md"

  jq -r '
    "- **\(.package)** (\(.severity)): \(.title)\n  - Range: \(.range)\n  - Fix available: \(.fixAvailable)"
  ' "$ANALYSIS_DIR/high_severity.json" >> "$ANALYSIS_DIR/dependency-report.md" 2>/dev/null

  echo "" >> "$ANALYSIS_DIR/dependency-report.md"
fi

cat >> "$ANALYSIS_DIR/dependency-report.md" << EOF

---

## Outdated Packages

| Update Type | Count |
|-------------|-------|
| Major | $MAJOR_UPDATES |
| Minor | $MINOR_UPDATES |
| Patch | $PATCH_UPDATES |
| **Total** | **$OUTDATED_COUNT** |

---

## Recommendations

EOF

# Generate recommendations based on findings
if [ "$CRITICAL" -gt 0 ]; then
  echo "1. **URGENT:** Address $CRITICAL critical vulnerabilities immediately" >> "$ANALYSIS_DIR/dependency-report.md"
  echo "   - Run: \`npm audit fix\` to auto-fix where possible" >> "$ANALYSIS_DIR/dependency-report.md"
  echo "   - Review: \`npm audit\` for manual fixes required" >> "$ANALYSIS_DIR/dependency-report.md"
  echo "" >> "$ANALYSIS_DIR/dependency-report.md"
fi

if [ "$HIGH" -gt 0 ]; then
  echo "2. **HIGH:** Review $HIGH high-severity vulnerabilities" >> "$ANALYSIS_DIR/dependency-report.md"
  echo "   - Prioritize packages with fix available" >> "$ANALYSIS_DIR/dependency-report.md"
  echo "" >> "$ANALYSIS_DIR/dependency-report.md"
fi

if [ "$MAJOR_UPDATES" != "?" ] && [ "$MAJOR_UPDATES" -gt 0 ]; then
  echo "3. **MAINTENANCE:** $MAJOR_UPDATES major version updates available" >> "$ANALYSIS_DIR/dependency-report.md"
  echo "   - Review changelogs for breaking changes before updating" >> "$ANALYSIS_DIR/dependency-report.md"
  echo "   - Consider: \`npx npm-check-updates -u\` for interactive updates" >> "$ANALYSIS_DIR/dependency-report.md"
fi

if [ $((CRITICAL + HIGH)) -eq 0 ] && [ "$OUTDATED_COUNT" -eq 0 ]; then
  echo "**All clear!** No security issues or outdated packages detected." >> "$ANALYSIS_DIR/dependency-report.md"
fi

cat >> "$ANALYSIS_DIR/dependency-report.md" << EOF

---

*Generated by analyze-dependencies workflow*
EOF

echo "Report saved to: $ANALYSIS_DIR/dependency-report.md"
```

**Output:** Human-readable markdown report at `dependency-report.md`
</step>

## Output Format

<output>
The workflow produces:

**Primary outputs:**
1. **dependency-findings.json** - Machine-readable findings for orchestrator
2. **dependency-report.md** - Human-readable analysis report

**Supporting outputs:**
- `audit.json` - Raw npm audit output
- `outdated.json` - Raw npm outdated output
- `high_severity.json` - Filtered high/critical vulnerabilities
- `outdated_details.json` - Categorized outdated packages
- `enriched_vulns.json` - deps.dev enriched data (if available)

**Findings JSON structure:**
```json
{
  "timestamp": "2026-01-08T10:30:00Z",
  "type": "dependency-analysis",
  "summary": {
    "vulnerabilities": {
      "total": 5,
      "critical": 1,
      "high": 2,
      "moderate": 2,
      "low": 0
    },
    "outdated": {
      "total": 12,
      "major": 3,
      "minor": 5,
      "patch": 4
    }
  },
  "status": "ACTION_REQUIRED",
  "priority": "CRITICAL"
}
```

**Status values:**
- `OK` - No high/critical vulnerabilities
- `ACTION_REQUIRED` - Has high or critical vulnerabilities

**Priority values:**
- `CRITICAL` - Has critical vulnerabilities
- `HIGH` - Has high (but no critical) vulnerabilities
- `NORMAL` - No urgent security issues
</output>

## Error Handling

<error_handling>
**Graceful degradation:**
- If jq unavailable: Fall back to grep-based JSON parsing
- If curl unavailable: Skip deps.dev enrichment
- If npm audit fails: Report empty results, don't fail workflow
- If deps.dev unreachable: Continue without enrichment

**Non-blocking failures:**
- Network errors during enrichment → Log warning, continue
- Rate limiting from deps.dev → Process available results
- Malformed JSON from external APIs → Skip that entry

**Hard failures (workflow exits):**
- No package.json → Cannot analyze, exit with clear message
- npm not installed → Cannot run analysis, exit with message
</error_handling>

## Integration

<integration>
**Called by:**
- `analyze-codebase.md` orchestrator (Phase 11-02)
- Manual invocation for dependency audits

**Feeds into:**
- Suggestion engine (Phase 12) for goal proposals
- CLAUDE.md dynamic suggestions section

**Environment variables:**
- `ANALYSIS_DIR` - Override output directory (default: `.planning/.analysis`)

**Example orchestrator integration:**
```bash
# Run dependency analysis
source analyze-dependencies.md

# Check if action required
STATUS=$(jq -r '.status' "$ANALYSIS_DIR/dependency-findings.json")
if [ "$STATUS" = "ACTION_REQUIRED" ]; then
  # Flag for suggestion engine
  echo "dependency_issues=true" >> "$ANALYSIS_DIR/flags.env"
fi
```
</integration>

## Examples

### Example 1: Clean Project (No Issues)

**Scenario:** Well-maintained project with current dependencies

**Output:**
```
=== Vulnerability Summary ===
Critical: 0
High: 0
Moderate: 0
Low: 0
Total: 0

=== Outdated Packages ===
Total outdated: 2
Major updates: 0
Minor updates: 1
Patch updates: 1

=== Analysis Complete ===
Status: OK
```

---

### Example 2: Project with Security Issues

**Scenario:** Project with critical vulnerability

**Output:**
```
=== Vulnerability Summary ===
Critical: 1
High: 3
Moderate: 5
Low: 2
Total: 11

=== Outdated Packages ===
Total outdated: 15
Major updates: 4
Minor updates: 6
Patch updates: 5

=== Analysis Complete ===
Status: ACTION_REQUIRED
Priority: CRITICAL
```

**Recommendations generated:**
1. URGENT: Address 1 critical vulnerability immediately
2. HIGH: Review 3 high-severity vulnerabilities
3. MAINTENANCE: 4 major version updates available

---

*Dependency analysis workflow for GSD autonomous goal setting*
