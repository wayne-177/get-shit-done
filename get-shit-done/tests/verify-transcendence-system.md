# Transcendence System Verification Tests

This guide provides manual test procedures to validate that the v5.0 Transcendence features work correctly. These tests ensure the workflow template system, AI workflow generation, monorepo discovery, and coordinated execution work properly both independently and as a complete pipeline.

## Purpose

Validate v5.0 Transcendence enhancements:
1. Template registry (18 workflows, variable definitions)
2. Intent matching (confidence thresholds, normalization)
3. Variable extraction (patterns, type conversion)
4. Intent routing (end-to-end flow)
5. AI workflow generation (safety validation)
6. Generated workflow registry (approval, re-use)
7. Monorepo detection (7 types)
8. Dependency graph construction (cycle detection, topological sort)
9. Impact analysis (git-based change detection)
10. /gsd:discover-monorepo command (flags, caching)
11. Coordinated execution workflow (strategies, ordering)
12. Execution dashboard (Mermaid graph, real-time updates)
13. /gsd:execute-monorepo command (modes, prerequisites)
14. End-to-end pipeline: Workflow factory
15. End-to-end pipeline: Monorepo coordination
16. Backward compatibility with v4.5/v4.0/v3.0

## Prerequisites

### Required Configuration

```bash
# Standard config.json (no v5.0-specific config needed)
cat > .planning/config.json << 'EOF'
{
  "mode": "interactive",
  "retry": {
    "enabled": true,
    "max_attempts": 3
  },
  "verification": {
    "enabled": true
  },
  "adaptive": {
    "enabled": true,
    "prediction_enabled": true,
    "show_pattern_hints": true
  },
  "suggestions": {
    "auto_suggest": true,
    "auto_update_claude_md": true
  }
}
EOF
```

### Required Files

```bash
# Verify workflow template system (Phase 16)
ls get-shit-done/references/template-schema.md
ls get-shit-done/workflow-templates/template-index.json
ls get-shit-done/workflows/intent-matcher.md
ls get-shit-done/workflows/variable-extractor.md
ls get-shit-done/workflows/route-intent.md

# Verify AI workflow generation (Phase 17)
ls get-shit-done/references/workflow-schema.json
ls get-shit-done/templates/generation-prompt.md
ls get-shit-done/references/workflow-safety.md
ls get-shit-done/workflows/generate-workflow.md
ls get-shit-done/workflow-templates/generated/generated-index.json

# Verify monorepo discovery (Phase 18)
ls get-shit-done/workflows/discover-monorepo.md
ls get-shit-done/references/monorepo-patterns.md
ls get-shit-done/workflows/impact-analysis.md
ls get-shit-done/commands/discover-monorepo.md
ls get-shit-done/templates/monorepo-info.md

# Verify coordinated execution (Phase 19)
ls get-shit-done/workflows/coordinated-execution.md
ls get-shit-done/templates/package-execution-summary.md
ls get-shit-done/templates/execution-dashboard.md
ls get-shit-done/commands/execute-monorepo.md
```

### Mock Monorepo Setup

```bash
# Create mock monorepo structure for testing
mkdir -p /tmp/gsd-test-monorepo
cd /tmp/gsd-test-monorepo

# Initialize git
git init

# Create pnpm workspace config
cat > pnpm-workspace.yaml << 'EOF'
packages:
  - 'packages/*'
EOF

# Create root package.json
cat > package.json << 'EOF'
{
  "name": "test-monorepo",
  "private": true,
  "workspaces": ["packages/*"]
}
EOF

# Create package A (no dependencies)
mkdir -p packages/pkg-a
cat > packages/pkg-a/package.json << 'EOF'
{
  "name": "@test/pkg-a",
  "version": "1.0.0",
  "main": "index.js"
}
EOF
echo "module.exports = { name: 'pkg-a' };" > packages/pkg-a/index.js

# Create package B (depends on A)
mkdir -p packages/pkg-b
cat > packages/pkg-b/package.json << 'EOF'
{
  "name": "@test/pkg-b",
  "version": "1.0.0",
  "main": "index.js",
  "dependencies": {
    "@test/pkg-a": "workspace:*"
  }
}
EOF
echo "const a = require('@test/pkg-a'); module.exports = { name: 'pkg-b', uses: a };" > packages/pkg-b/index.js

# Create package C (depends on B)
mkdir -p packages/pkg-c
cat > packages/pkg-c/package.json << 'EOF'
{
  "name": "@test/pkg-c",
  "version": "1.0.0",
  "main": "index.js",
  "dependencies": {
    "@test/pkg-b": "workspace:*"
  }
}
EOF
echo "const b = require('@test/pkg-b'); module.exports = { name: 'pkg-c', uses: b };" > packages/pkg-c/index.js

# Commit initial structure
git add -A
git commit -m "Initial monorepo structure"

# Create feature branch for impact testing
git checkout -b feature/test-change
echo "// updated" >> packages/pkg-a/index.js
git add packages/pkg-a/index.js
git commit -m "Change pkg-a"
git checkout main

echo "Mock monorepo created at /tmp/gsd-test-monorepo"
```

---

## Test Scenario 1: Template Registry

**Objective:** Verify template-index.json loads correctly with all 18 workflow templates and proper schema

### Prerequisites

- get-shit-done/workflow-templates/template-index.json exists

### Setup

```bash
# Verify template registry exists
ls get-shit-done/workflow-templates/template-index.json
```

### Test Execution

**Test 1a: Template count verification**

```bash
# Count templates in index
cat get-shit-done/workflow-templates/template-index.json | grep -c '"id":'
# Expected: 18 templates
```

**Test 1b: Category distribution**

```bash
# Check each category has templates
cat get-shit-done/workflow-templates/template-index.json | grep '"category":' | sort | uniq -c
# Expected categories: planning, execution, analysis, project, monitoring
```

**Test 1c: Variable definitions**

```bash
# Check variable definitions with regex patterns
grep -A 5 '"phase_number"' get-shit-done/workflow-templates/template-index.json
# Expected: pattern field with regex like "[0-9]+"

grep -A 5 '"milestone_version"' get-shit-done/workflow-templates/template-index.json
# Expected: pattern field with regex like "v?[0-9]+\\.[0-9]+"
```

**Test 1d: Schema validation**

```bash
# Verify each template has required fields
for field in '"id":' '"workflow":' '"category":' '"patterns":' '"examples":'; do
  echo "Checking $field"
  COUNT=$(grep -c "$field" get-shit-done/workflow-templates/template-index.json)
  echo "Found: $COUNT occurrences"
done
# Expected: Each field appears 18 times (once per template)
```

### Verification Steps

1. **Template count:**
   ```bash
   COUNT=$(cat get-shit-done/workflow-templates/template-index.json | grep -c '"id":')
   [ "$COUNT" -eq 18 ] && echo "PASS: 18 templates" || echo "FAIL: Expected 18, got $COUNT"
   ```

2. **All categories present:**
   ```bash
   for cat in planning execution analysis project monitoring; do
     grep -q "\"category\": \"$cat\"" get-shit-done/workflow-templates/template-index.json && \
       echo "PASS: $cat category found" || echo "FAIL: $cat category missing"
   done
   ```

### Success Criteria

- [ ] Template registry contains 18 workflow templates
- [ ] All 5 categories present (planning, execution, analysis, project, monitoring)
- [ ] Variable definitions include regex patterns (phase_number, milestone_version)
- [ ] Each template has required fields (id, workflow, category, patterns, examples)

---

## Test Scenario 2: Intent Matching

**Objective:** Verify intent-matcher.md correctly matches natural language to workflows with confidence scoring

### Prerequisites

- intent-matcher.md workflow exists
- template-index.json loaded

### Setup

```bash
# Verify intent matcher workflow exists
ls get-shit-done/workflows/intent-matcher.md
```

### Test Execution

**Test 2a: HIGH confidence matches (<=0.3)**

```bash
# Test with exact pattern match
# Input: "plan phase 3"
# Expected: Match "plan-phase" template with confidence <=0.3 (HIGH)

# Test with common variation
# Input: "execute the plan"
# Expected: Match "execute-plan" template with confidence <=0.3 (HIGH)
```

**Test 2b: MEDIUM confidence matches (0.3-0.5)**

```bash
# Test with partial match
# Input: "help me plan something"
# Expected: Match with confidence 0.3-0.5 (MEDIUM), requires confirmation
```

**Test 2c: LOW confidence matches (>0.5)**

```bash
# Test with novel request
# Input: "generate a unicorn report"
# Expected: No match or confidence >0.5 (LOW), falls back to Claude
```

**Test 2d: Normalization rules**

```bash
# Test lowercase normalization
# Input: "PLAN PHASE 3" should match same as "plan phase 3"

# Test punctuation removal
# Input: "plan phase 3!" should match same as "plan phase 3"

# Test extra whitespace handling
# Input: "plan   phase   3" should match same as "plan phase 3"
```

**Test 2e: Category hint bonus**

```bash
# Test with category context
# Input: "plan phase 3" with category_hint: "planning"
# Expected: +0.1 confidence bonus applied (lower distance = higher confidence)
```

**Test 2f: Sample inputs validation**

```bash
# Test specific inputs from plan
echo "Testing 'plan phase 3'"
# Expected: Match plan-phase, HIGH confidence

echo "Testing 'execute the plan'"
# Expected: Match execute-plan, HIGH confidence

echo "Testing 'check project health'"
# Expected: Match health-check, HIGH confidence
```

### Verification Steps

1. **Confidence threshold separation:**
   ```bash
   # Verify thresholds documented in workflow
   grep -E "(HIGH|MEDIUM|LOW).*0\.[0-9]" get-shit-done/workflows/intent-matcher.md
   # Expected: <=0.3 HIGH, 0.3-0.5 MEDIUM, >0.5 LOW
   ```

2. **Normalization documented:**
   ```bash
   grep -i "normali" get-shit-done/workflows/intent-matcher.md
   # Expected: lowercase, punctuation, whitespace handling mentioned
   ```

### Success Criteria

- [ ] HIGH confidence (<=0.3) triggers direct execution
- [ ] MEDIUM confidence (0.3-0.5) requires user confirmation
- [ ] LOW confidence (>0.5) falls back to Claude/generation
- [ ] Normalization handles case, punctuation, whitespace
- [ ] Category hint bonus (+0.1) applied correctly
- [ ] Sample inputs match expected templates

---

## Test Scenario 3: Variable Extraction

**Objective:** Verify variable-extractor.md extracts variables from natural language using regex patterns

### Prerequisites

- variable-extractor.md workflow exists

### Setup

```bash
# Verify variable extractor workflow exists
ls get-shit-done/workflows/variable-extractor.md
```

### Test Execution

**Test 3a: phase_number extraction**

```bash
# Test various phase number formats
echo "Testing 'phase 5'"
# Expected: phase_number = 5

echo "Testing 'Phase 12'"
# Expected: phase_number = 12

echo "Testing '5th phase'"
# Expected: phase_number = 5

echo "Testing 'plan phase number 3'"
# Expected: phase_number = 3
```

**Test 3b: milestone_version extraction**

```bash
# Test various version formats
echo "Testing 'v2.0'"
# Expected: milestone_version = v2.0 or 2.0

echo "Testing 'version 3.5'"
# Expected: milestone_version = 3.5

echo "Testing 'milestone 4'"
# Expected: milestone_version = 4
```

**Test 3c: File path extraction**

```bash
# Test file path patterns
echo "Testing 'run plan at .planning/phases/01-foundation/01-01-PLAN.md'"
# Expected: plan_path = .planning/phases/01-foundation/01-01-PLAN.md

echo "Testing 'execute .planning/phases/02-auth/02-03-PLAN.md'"
# Expected: plan_path = .planning/phases/02-auth/02-03-PLAN.md
```

**Test 3d: Missing required variable prompting**

```bash
# Test when required variable missing
echo "Testing 'plan phase' (no number)"
# Expected: Prompt user for phase_number
```

**Test 3e: Type conversion**

```bash
# Test type conversion
# phase_number should be number type
echo "Testing type of phase_number from 'phase 5'"
# Expected: typeof phase_number === 'number'

# plan_path should be string type
echo "Testing type of plan_path from 'run .planning/foo.md'"
# Expected: typeof plan_path === 'string'
```

### Verification Steps

1. **Regex patterns documented:**
   ```bash
   grep -E "pattern|regex" get-shit-done/workflows/variable-extractor.md
   # Expected: Patterns for phase_number, milestone_version, file paths
   ```

2. **Type definitions:**
   ```bash
   grep -E "type.*number|type.*string|type.*path" get-shit-done/workflows/variable-extractor.md
   # Expected: Type conversion rules documented
   ```

### Success Criteria

- [ ] phase_number extracted from various formats (phase 5, Phase 12, 5th phase)
- [ ] milestone_version extracted from various formats (v2.0, version 3.5, milestone 4)
- [ ] File paths extracted correctly
- [ ] Missing required variables trigger user prompt
- [ ] Type conversion works (number, string, path)

---

## Test Scenario 4: Intent Routing

**Objective:** Verify route-intent.md orchestrates the complete flow from natural language to workflow execution

### Prerequisites

- route-intent.md workflow exists
- All component workflows available

### Setup

```bash
# Verify route-intent workflow exists
ls get-shit-done/workflows/route-intent.md
```

### Test Execution

**Test 4a: HIGH confidence direct execution**

```bash
# Test end-to-end with high confidence input
# Input: "plan phase 3"
# Expected flow:
# 1. Intent matching finds plan-phase (HIGH confidence <=0.3)
# 2. Variable extraction gets phase_number=3
# 3. Workflow executed directly without confirmation
```

**Test 4b: MEDIUM confidence with confirmation**

```bash
# Test with medium confidence input
# Input: "help me plan my project"
# Expected flow:
# 1. Intent matching finds possible match (MEDIUM confidence 0.3-0.5)
# 2. User presented with confirmation: "Did you mean: [workflow]?"
# 3. On confirmation, variable extraction proceeds
# 4. Workflow executed
```

**Test 4c: LOW confidence fallback to generation**

```bash
# Test with low confidence input
# Input: "create a weekly status report template"
# Expected flow:
# 1. Intent matching fails or LOW confidence (>0.5)
# 2. Falls back to generate-workflow.md
# 3. AI generates workflow based on request
```

**Test 4d: End-to-end routing**

```bash
# Test complete routing for each confidence level
echo "Testing HIGH: 'execute plan 01-01'"
echo "Testing MEDIUM: 'do something with the codebase'"
echo "Testing LOW: 'generate rainbow reports'"
```

### Verification Steps

1. **Routing logic documented:**
   ```bash
   grep -E "HIGH|MEDIUM|LOW" get-shit-done/workflows/route-intent.md
   # Expected: Different handling for each confidence level
   ```

2. **Fallback to generation:**
   ```bash
   grep -i "generate-workflow" get-shit-done/workflows/route-intent.md
   # Expected: Reference to generate-workflow.md for LOW confidence
   ```

### Success Criteria

- [ ] HIGH confidence routes directly to workflow execution
- [ ] MEDIUM confidence requests user confirmation
- [ ] LOW confidence falls back to generate-workflow.md
- [ ] Variables extracted and passed to workflow
- [ ] End-to-end flow works from natural language to execution

---

## Test Scenario 5: AI Workflow Generation

**Objective:** Verify generate-workflow.md creates valid workflows with safety validation

### Prerequisites

- generate-workflow.md workflow exists
- workflow-schema.json exists
- workflow-safety.md exists

### Setup

```bash
# Verify AI generation components exist
ls get-shit-done/workflows/generate-workflow.md
ls get-shit-done/references/workflow-schema.json
ls get-shit-done/references/workflow-safety.md
```

### Test Execution

**Test 5a: Novel request generation**

```bash
# Test workflow generation for novel request
# Input: "generate a dependency audit report"
# Expected: AI generates workflow with:
# - Unique ID
# - Description
# - Steps array
# - Verification criteria
```

**Test 5b: Schema validation**

```bash
# Verify generated workflow matches schema
cat get-shit-done/references/workflow-schema.json
# Expected fields: id, description, steps, verification

# Generated workflow should validate against this schema
```

**Test 5c: Safety validation (BLOCKED)**

```bash
# Test blocklist patterns
# Input requesting: rm -rf, eval, sudo
# Expected: Safety validation returns BLOCKED
# Expected: Generation fails with security warning
```

**Test 5d: Safety validation (WARNING)**

```bash
# Test warning patterns
# Input with filesystem writes to sensitive paths
# Expected: Safety validation returns WARNING
# Expected: User warned before execution
```

**Test 5e: Safety validation (SAFE)**

```bash
# Test allowlist commands
# Input using: npm test, git status, cat, grep
# Expected: Safety validation returns SAFE
# Expected: No warnings, proceeds to confirmation
```

**Test 5f: Human confirmation before execution**

```bash
# Verify human confirmation step
# Expected: Generated workflow NEVER executes without user approval
# Expected: User sees workflow steps before confirming
```

### Verification Steps

1. **Blocklist patterns documented:**
   ```bash
   grep -E "rm -rf|eval|sudo|BLOCK" get-shit-done/references/workflow-safety.md
   # Expected: Dangerous patterns listed
   ```

2. **Allowlist patterns documented:**
   ```bash
   grep -E "npm test|git status|SAFE|ALLOW" get-shit-done/references/workflow-safety.md
   # Expected: Safe patterns listed
   ```

3. **Human confirmation required:**
   ```bash
   grep -i "confirm" get-shit-done/workflows/generate-workflow.md
   # Expected: Confirmation step before execution
   ```

### Success Criteria

- [ ] AI generates valid workflows for novel requests
- [ ] Generated workflows validate against workflow-schema.json
- [ ] BLOCKED: Dangerous patterns rejected (rm -rf, eval, sudo)
- [ ] WARNING: Risky patterns warn user
- [ ] SAFE: Allowlist commands pass validation
- [ ] Human confirmation required before executing generated workflow

---

## Test Scenario 6: Generated Workflow Registry

**Objective:** Verify approved generated workflows are stored and can be re-used

### Prerequisites

- generated-index.json exists
- generate-workflow.md stores approved workflows

### Setup

```bash
# Verify generated workflow registry exists
ls get-shit-done/workflow-templates/generated/generated-index.json
```

### Test Execution

**Test 6a: Workflow storage after approval**

```bash
# After user approves a generated workflow:
# Expected: Workflow saved to generated/ directory
# Expected: generated-index.json updated with new entry
```

**Test 6b: Re-use of previously approved workflows**

```bash
# When user requests same workflow again:
# Input: "generate a dependency audit report" (previously approved)
# Expected: Intent matching finds generated workflow
# Expected: Re-uses stored workflow instead of regenerating
```

**Test 6c: Separate storage from built-in templates**

```bash
# Verify generated workflows stored separately
ls get-shit-done/workflow-templates/generated/
# Expected: Generated workflows here, not in main template-index.json

cat get-shit-done/workflow-templates/template-index.json | grep -c '"id":'
# Expected: Still 18 (built-in templates unchanged)
```

**Test 6d: Workflow ID uniqueness**

```bash
# Verify generated workflow IDs don't conflict
cat get-shit-done/workflow-templates/generated/generated-index.json
# Expected: All IDs unique (e.g., gen-{hash} format)

# Cross-check with built-in templates
# Expected: No ID collisions
```

### Verification Steps

1. **Generated index structure:**
   ```bash
   cat get-shit-done/workflow-templates/generated/generated-index.json
   # Expected: JSON array with generated workflow entries
   ```

2. **ID uniqueness:**
   ```bash
   # Check for duplicate IDs across both registries
   cat get-shit-done/workflow-templates/template-index.json \
       get-shit-done/workflow-templates/generated/generated-index.json \
       | grep '"id":' | sort | uniq -d
   # Expected: No output (no duplicates)
   ```

### Success Criteria

- [ ] Approved generated workflows saved to generated/ directory
- [ ] generated-index.json updated with new entries
- [ ] Previously approved workflows can be re-used
- [ ] Generated workflows stored separately from built-in templates
- [ ] Workflow IDs are unique (no collisions)

---

## Test Scenario 7: Monorepo Detection

**Objective:** Verify discover-monorepo.md correctly detects all 7 monorepo types

### Prerequisites

- discover-monorepo.md workflow exists
- Mock monorepos for each type

### Setup

```bash
# Use mock monorepo from Prerequisites section
cd /tmp/gsd-test-monorepo
```

### Test Execution

**Test 7a: pnpm detection (pnpm-workspace.yaml)**

```bash
# Already exists in mock monorepo
ls pnpm-workspace.yaml
# Expected: discover-monorepo detects type: "pnpm"
```

**Test 7b: turborepo detection (turbo.json)**

```bash
# Create turbo.json
cat > turbo.json << 'EOF'
{
  "pipeline": {
    "build": { "dependsOn": ["^build"] },
    "test": { "dependsOn": ["build"] }
  }
}
EOF
# Expected: discover-monorepo detects type: "turborepo"
# Note: pnpm takes priority if both exist
rm turbo.json  # Clean up for other tests
```

**Test 7c: nx detection (nx.json)**

```bash
# Create nx.json
cat > nx.json << 'EOF'
{
  "targetDefaults": {
    "build": { "dependsOn": ["^build"] }
  }
}
EOF
# Expected: discover-monorepo detects type: "nx"
rm nx.json  # Clean up
```

**Test 7d: lerna detection (lerna.json)**

```bash
# Create lerna.json
cat > lerna.json << 'EOF'
{
  "version": "independent",
  "packages": ["packages/*"]
}
EOF
# Expected: discover-monorepo detects type: "lerna"
rm lerna.json  # Clean up
```

**Test 7e: rush detection (rush.json)**

```bash
# Create rush.json
cat > rush.json << 'EOF'
{
  "rushVersion": "5.0.0",
  "projects": []
}
EOF
# Expected: discover-monorepo detects type: "rush"
rm rush.json  # Clean up
```

**Test 7f: npm-workspaces detection (package.json workspaces)**

```bash
# Remove pnpm-workspace.yaml to test npm workspaces fallback
mv pnpm-workspace.yaml pnpm-workspace.yaml.bak
# package.json already has workspaces array
# Expected: discover-monorepo detects type: "npm-workspaces"
mv pnpm-workspace.yaml.bak pnpm-workspace.yaml  # Restore
```

**Test 7g: single-package fallback**

```bash
# Create single-package project
mkdir -p /tmp/gsd-test-single
cd /tmp/gsd-test-single
cat > package.json << 'EOF'
{ "name": "single-pkg", "version": "1.0.0" }
EOF
# Expected: discover-monorepo detects type: "single-package"
```

**Test 7h: Detection priority order**

```bash
# Priority: pnpm > turborepo > nx > lerna > rush > npm-workspaces > single-package
# If multiple configs exist, highest priority wins
cd /tmp/gsd-test-monorepo
# With pnpm-workspace.yaml present, type should be "pnpm" regardless of others
```

**Test 7i: MONOREPO_INFO JSON output**

```bash
# Expected output structure:
# {
#   "type": "pnpm",
#   "root": "/tmp/gsd-test-monorepo",
#   "packages": [...],
#   "dependency_graph": {...},
#   "build_order": [...]
# }
```

### Verification Steps

1. **All 7 types documented:**
   ```bash
   grep -E "pnpm|turborepo|nx|lerna|rush|npm-workspaces|single-package" \
     get-shit-done/references/monorepo-patterns.md
   # Expected: All 7 types documented
   ```

2. **Priority order documented:**
   ```bash
   grep -i "priority" get-shit-done/workflows/discover-monorepo.md
   # Expected: Detection priority order specified
   ```

### Success Criteria

- [ ] pnpm detected via pnpm-workspace.yaml
- [ ] turborepo detected via turbo.json
- [ ] nx detected via nx.json
- [ ] lerna detected via lerna.json
- [ ] rush detected via rush.json
- [ ] npm-workspaces detected via package.json workspaces array
- [ ] single-package fallback for non-monorepo
- [ ] Detection priority order followed (pnpm > turborepo > nx > lerna > rush > npm-workspaces)
- [ ] MONOREPO_INFO JSON output format correct

---

## Test Scenario 8: Dependency Graph Construction

**Objective:** Verify dependency graph building with cycle detection and topological sort

### Prerequisites

- Mock monorepo with dependencies (from Prerequisites)

### Setup

```bash
cd /tmp/gsd-test-monorepo
```

### Test Execution

**Test 8a: Dependency graph from package.json**

```bash
# Expected graph:
# @test/pkg-a → (no deps)
# @test/pkg-b → @test/pkg-a
# @test/pkg-c → @test/pkg-b

# Graph should include both dependencies and devDependencies
```

**Test 8b: Dependencies and devDependencies included**

```bash
# Add devDependency to pkg-b
cat > packages/pkg-b/package.json << 'EOF'
{
  "name": "@test/pkg-b",
  "version": "1.0.0",
  "dependencies": { "@test/pkg-a": "workspace:*" },
  "devDependencies": { "@test/pkg-a": "workspace:*" }
}
EOF
# Expected: Both deps included in graph
```

**Test 8c: Cycle detection**

```bash
# Create circular dependency
cat > packages/pkg-a/package.json << 'EOF'
{
  "name": "@test/pkg-a",
  "version": "1.0.0",
  "dependencies": { "@test/pkg-c": "workspace:*" }
}
EOF
# Expected: discover-monorepo warns about cycle: pkg-a → pkg-c → pkg-b → pkg-a
# Expected: Still produces output, but with cycle warning
# Restore original
cat > packages/pkg-a/package.json << 'EOF'
{ "name": "@test/pkg-a", "version": "1.0.0" }
EOF
```

**Test 8d: Topological sort (build order)**

```bash
# Expected build order for acyclic graph:
# 1. @test/pkg-a (no deps, build first)
# 2. @test/pkg-b (depends on pkg-a)
# 3. @test/pkg-c (depends on pkg-b)

# Verify build_order in output
```

**Test 8e: Mermaid diagram generation**

```bash
# Expected Mermaid output:
# ```mermaid
# graph TD
#   pkg-a["@test/pkg-a"]
#   pkg-b["@test/pkg-b"]
#   pkg-c["@test/pkg-c"]
#   pkg-b --> pkg-a
#   pkg-c --> pkg-b
# ```

# Verify scoped package names escaped correctly
# @test/pkg-a should become test_pkg-a or similar in node IDs
```

### Verification Steps

1. **Graph structure verified:**
   ```bash
   # Run discovery and check dependency_graph field
   # Expected: JSON object with package names as keys
   ```

2. **Build order correctness:**
   ```bash
   # Verify build_order is topologically sorted
   # For A → B → C: order should be [A, B, C]
   ```

### Success Criteria

- [ ] Dependency graph built from package.json files
- [ ] Both dependencies and devDependencies included
- [ ] Cycle detection identifies circular dependencies
- [ ] Topological sort produces correct build order
- [ ] Mermaid diagram generated with proper escaping for scoped packages

---

## Test Scenario 9: Impact Analysis

**Objective:** Verify impact-analysis.md correctly identifies affected packages from git changes

### Prerequisites

- Mock monorepo with feature branch (from Prerequisites)

### Setup

```bash
cd /tmp/gsd-test-monorepo
```

### Test Execution

**Test 9a: Git merge-base strategy**

```bash
# Switch to feature branch
git checkout feature/test-change

# Expected: impact-analysis uses git merge-base --fork-point
# to find divergence point from main
```

**Test 9b: Direct affected packages**

```bash
# pkg-a/index.js was changed
# Expected: pkg-a identified as directly affected

# Check AFFECTED_PACKAGES output
# direct_affected: ["@test/pkg-a"]
```

**Test 9c: Transitive affected packages**

```bash
# pkg-b depends on pkg-a
# pkg-c depends on pkg-b
# Expected: Both pkg-b and pkg-c identified as transitively affected

# transitive_affected: ["@test/pkg-b", "@test/pkg-c"]
```

**Test 9d: Root package.json filtering**

```bash
# Change root package.json
git checkout main
echo '{"name": "test-monorepo", "private": true, "version": "1.0.1"}' > package.json

# Expected: Only deps/workspaces changes in root count
# Adding version shouldn't trigger all packages as affected
```

**Test 9e: AFFECTED_PACKAGES JSON output**

```bash
# Expected output structure:
# {
#   "base": "main",
#   "head": "feature/test-change",
#   "direct_affected": ["@test/pkg-a"],
#   "transitive_affected": ["@test/pkg-b", "@test/pkg-c"],
#   "all_affected": ["@test/pkg-a", "@test/pkg-b", "@test/pkg-c"]
# }
```

### Verification Steps

1. **Merge-base strategy documented:**
   ```bash
   grep -i "merge-base" get-shit-done/workflows/impact-analysis.md
   # Expected: --fork-point strategy mentioned
   ```

2. **Transitive calculation:**
   ```bash
   grep -i "transitive" get-shit-done/workflows/impact-analysis.md
   # Expected: Transitive dependency walking documented
   ```

### Success Criteria

- [ ] Git merge-base used to find divergence point
- [ ] Direct affected packages identified from changed files
- [ ] Transitive affected packages calculated from dependency graph
- [ ] Root package.json filtered (only deps/workspaces changes count)
- [ ] AFFECTED_PACKAGES JSON output format correct

---

## Test Scenario 10: /gsd:discover-monorepo Command

**Objective:** Verify the discover-monorepo command with all flags

### Prerequisites

- discover-monorepo.md command exists

### Setup

```bash
cd /tmp/gsd-test-monorepo
ls get-shit-done/commands/discover-monorepo.md
```

### Test Execution

**Test 10a: --all flag (full discovery)**

```bash
# Run: /gsd:discover-monorepo --all
# Expected: Full discovery without impact analysis
# Expected: MONOREPO_INFO.md created in .planning/
```

**Test 10b: --impact flag (include change impact)**

```bash
# Run: /gsd:discover-monorepo --impact
# Expected: Discovery + impact analysis from current branch
# Expected: AFFECTED_PACKAGES included in output
```

**Test 10c: --refresh flag (force re-discovery)**

```bash
# Run: /gsd:discover-monorepo --refresh
# Expected: Ignores cached results, runs fresh discovery
```

**Test 10d: Caching behavior (7-day default)**

```bash
# Run discovery twice
# First run: Full discovery
# Second run (within 7 days): Uses cache

# Check cache file
ls .planning/.monorepo-cache.json 2>/dev/null

# Expected: Cache contains timestamp and results
```

**Test 10e: MONOREPO_INFO.md template output**

```bash
# Verify output matches template
cat .planning/MONOREPO_INFO.md

# Expected sections:
# - Overview (type, package count)
# - Packages (table with name, path, dependencies)
# - Dependency Graph (Mermaid diagram)
# - Build Order (numbered list)
# - JSON Appendix (full MONOREPO_INFO JSON)
```

### Verification Steps

1. **All flags documented:**
   ```bash
   grep -E "\-\-all|\-\-impact|\-\-refresh" get-shit-done/commands/discover-monorepo.md
   # Expected: All three flags documented
   ```

2. **Cache duration:**
   ```bash
   grep -i "cache\|7.day" get-shit-done/commands/discover-monorepo.md
   # Expected: 7-day cache duration mentioned
   ```

### Success Criteria

- [ ] --all flag runs full discovery
- [ ] --impact flag includes change impact analysis
- [ ] --refresh flag bypasses cache
- [ ] 7-day default cache behavior works
- [ ] MONOREPO_INFO.md follows template with JSON appendix

---

## Test Scenario 11: Coordinated Execution Workflow

**Objective:** Verify coordinated-execution.md runs tasks across packages in correct order

### Prerequisites

- coordinated-execution.md workflow exists
- Mock monorepo available

### Setup

```bash
cd /tmp/gsd-test-monorepo
ls get-shit-done/workflows/coordinated-execution.md
```

### Test Execution

**Test 11a: AFFECTED_PACKAGES input (partial execution)**

```bash
# Run with specific affected packages
# Input: AFFECTED_PACKAGES = ["@test/pkg-a", "@test/pkg-b"]
# Expected: Only pkg-a and pkg-b executed (not pkg-c)
```

**Test 11b: MONOREPO_INFO input (full execution)**

```bash
# Run with full monorepo info
# Input: All packages from MONOREPO_INFO
# Expected: All packages executed
```

**Test 11c: Dependency-order execution**

```bash
# Expected execution order:
# 1. @test/pkg-a (no deps)
# 2. @test/pkg-b (after pkg-a complete)
# 3. @test/pkg-c (after pkg-b complete)

# Packages should NOT start until dependencies complete
```

**Test 11d: Continue-on-failure strategy (default)**

```bash
# Configure pkg-b to fail
# Expected: pkg-b fails
# Expected: pkg-c still attempts (marked as may be affected)
# Expected: Execution continues for independent packages
```

**Test 11e: Stop-on-failure strategy (alternative)**

```bash
# With stop-on-failure enabled:
# Expected: pkg-b fails
# Expected: pkg-c is SKIPPED (not attempted)
# Expected: Execution stops for dependency chain
```

**Test 11f: Per-package task configuration**

```bash
# Default tasks: install, build, test
# Custom tasks: Can specify per package
# Expected: Tasks run in order for each package
```

**Test 11g: Skipped status for dependency cascade**

```bash
# When pkg-a fails and stop-on-failure:
# Expected: pkg-b status = "skipped" (not attempted)
# Expected: pkg-c status = "skipped" (not attempted)
# Expected: Reason includes "dependency @test/pkg-a failed"
```

### Verification Steps

1. **Execution strategies documented:**
   ```bash
   grep -E "continue-on-failure|stop-on-failure" get-shit-done/workflows/coordinated-execution.md
   # Expected: Both strategies documented
   ```

2. **Dependency ordering:**
   ```bash
   grep -i "topological\|build.order\|dependency.order" get-shit-done/workflows/coordinated-execution.md
   # Expected: Dependency-based ordering documented
   ```

### Success Criteria

- [ ] AFFECTED_PACKAGES input limits execution scope
- [ ] MONOREPO_INFO input runs all packages
- [ ] Dependency-order execution (packages after their deps)
- [ ] Continue-on-failure continues for unrelated packages
- [ ] Stop-on-failure stops dependent packages
- [ ] Per-package task configuration (install, build, test)
- [ ] Skipped status for dependency cascade failures

---

## Test Scenario 12: Execution Dashboard

**Objective:** Verify execution-dashboard.md template renders correctly with status updates

### Prerequisites

- execution-dashboard.md template exists

### Setup

```bash
ls get-shit-done/templates/execution-dashboard.md
```

### Test Execution

**Test 12a: Template rendering**

```bash
# Verify dashboard template can be rendered
cat get-shit-done/templates/execution-dashboard.md

# Expected: Placeholders for:
# - Package statuses
# - Timing information
# - Error details
```

**Test 12b: Mermaid graph with status classes**

```bash
# Expected CSS classes in Mermaid:
# - pending (gray)
# - running (blue)
# - success (green)
# - failed (red)
# - skipped (yellow/orange)

# Example:
# ```mermaid
# graph TD
#   classDef success fill:#10b981
#   classDef failed fill:#ef4444
#   pkg-a:::success
#   pkg-b:::failed
# ```
```

**Test 12c: Real-time dashboard updates**

```bash
# During execution:
# Expected: Dashboard file updated after each package completes
# Expected: Status changes from pending → running → success/failed

# Check file modification time changes during execution
```

**Test 12d: Status table with timing**

```bash
# Expected table format:
# | Package | Status | Duration | Tasks |
# |---------|--------|----------|-------|
# | pkg-a   | success| 12s      | 3/3   |
# | pkg-b   | failed | 8s       | 2/3   |
```

**Test 12e: Error section for failed packages**

```bash
# Expected error section:
# ## Errors
#
# ### @test/pkg-b
# ```
# npm ERR! test failed
# ...
# ```

# Only shown for failed packages
```

### Verification Steps

1. **Mermaid classes documented:**
   ```bash
   grep -E "classDef|pending|running|success|failed|skipped" \
     get-shit-done/templates/execution-dashboard.md
   # Expected: CSS classes for all statuses
   ```

2. **Timing fields:**
   ```bash
   grep -i "duration\|timing\|started\|completed" get-shit-done/templates/execution-dashboard.md
   # Expected: Timing fields in template
   ```

### Success Criteria

- [ ] Dashboard template renders with proper structure
- [ ] Mermaid graph shows status via CSS classes (pending, running, success, failed, skipped)
- [ ] Dashboard updates in real-time during execution
- [ ] Status table includes timing information
- [ ] Error section populated for failed packages

---

## Test Scenario 13: /gsd:execute-monorepo Command

**Objective:** Verify the execute-monorepo command with all flags and modes

### Prerequisites

- execute-monorepo.md command exists

### Setup

```bash
cd /tmp/gsd-test-monorepo
ls get-shit-done/commands/execute-monorepo.md
```

### Test Execution

**Test 13a: --all flag (all packages)**

```bash
# Run: /gsd:execute-monorepo --all
# Expected: Executes tasks for all packages in dependency order
```

**Test 13b: --affected flag (changed packages only)**

```bash
# Run: /gsd:execute-monorepo --affected
# Expected: Runs impact analysis first
# Expected: Only executes affected packages + their dependents
```

**Test 13c: --package flag (specific package)**

```bash
# Run: /gsd:execute-monorepo --package @test/pkg-b
# Expected: Executes only pkg-b (and its dependencies if needed)
```

**Test 13d: --status flag (check current status)**

```bash
# Run: /gsd:execute-monorepo --status
# Expected: Shows current execution status without starting new execution
# Expected: Reads from EXECUTION_DASHBOARD.md if exists
```

**Test 13e: Auto-mode detection**

```bash
# On main branch:
git checkout main
# Expected: Auto-mode selects --all

# On feature branch:
git checkout feature/test-change
# Expected: Auto-mode selects --affected
```

**Test 13f: --tasks flag customization**

```bash
# Run: /gsd:execute-monorepo --all --tasks "lint,test"
# Expected: Runs only lint and test (not install, build)
```

**Test 13g: Prerequisite auto-execution**

```bash
# Without MONOREPO_INFO.md:
# Run: /gsd:execute-monorepo --all
# Expected: Automatically runs /gsd:discover-monorepo first
# Expected: Then proceeds with execution
```

### Verification Steps

1. **All flags documented:**
   ```bash
   grep -E "\-\-all|\-\-affected|\-\-package|\-\-status|\-\-tasks" \
     get-shit-done/commands/execute-monorepo.md
   # Expected: All flags documented
   ```

2. **Auto-mode logic:**
   ```bash
   grep -i "auto.mode\|main.branch\|feature" get-shit-done/commands/execute-monorepo.md
   # Expected: Auto-mode detection logic documented
   ```

### Success Criteria

- [ ] --all flag executes all packages
- [ ] --affected flag executes changed packages + dependents
- [ ] --package flag executes specific package
- [ ] --status flag shows current execution status
- [ ] Auto-mode: main branch → --all, feature branch → --affected
- [ ] --tasks flag customizes task list
- [ ] Prerequisite auto-execution (discover-monorepo if needed)

---

## Test Scenario 14: End-to-End Pipeline - Workflow Factory

**Objective:** Test complete flow from natural language input through workflow execution

### Prerequisites

- All workflow factory components installed

### Setup

```bash
cd /tmp/gsd-test-monorepo
mkdir -p .planning
```

### Test Execution

**Test 14a: Template match flow**

```bash
# Input: "plan phase 5"
# Expected flow:
# 1. route-intent receives input
# 2. intent-matcher matches "plan-phase" (HIGH confidence)
# 3. variable-extractor extracts phase_number=5
# 4. plan-phase workflow executed with phase=5
```

**Test 14b: Novel request flow**

```bash
# Input: "analyze code complexity and suggest refactoring targets"
# Expected flow:
# 1. route-intent receives input
# 2. intent-matcher fails to match (LOW confidence)
# 3. Falls back to generate-workflow
# 4. AI generates workflow steps
# 5. Safety validation runs (SAFE expected)
# 6. User confirms generated workflow
# 7. Workflow executes
# 8. Workflow saved to generated-index.json for re-use
```

**Test 14c: Generated workflow re-use**

```bash
# Input: Same request as Test 14b (after approval)
# Expected flow:
# 1. route-intent receives input
# 2. intent-matcher checks generated-index.json
# 3. Finds previously approved workflow
# 4. Uses stored workflow (no regeneration)
```

### Verification Steps

1. **Template match verified:**
   ```bash
   # Run with known template input
   # Verify correct workflow executed
   # Verify variables passed correctly
   ```

2. **Generated workflow saved:**
   ```bash
   # After novel request approval
   cat get-shit-done/workflow-templates/generated/generated-index.json
   # Expected: New entry for approved workflow
   ```

### Success Criteria

- [ ] Template match: natural language → matched template → execution
- [ ] Novel request: natural language → AI generation → confirmation → execution
- [ ] Generated workflow saved after approval
- [ ] Re-use: same request finds saved workflow

---

## Test Scenario 15: End-to-End Pipeline - Monorepo Coordination

**Objective:** Test complete flow from discovery through coordinated execution

### Prerequisites

- Mock monorepo with 3 packages

### Setup

```bash
cd /tmp/gsd-test-monorepo
rm -rf .planning
mkdir -p .planning
```

### Test Execution

**Test 15a: Full pipeline flow**

```bash
# Step 1: Run /gsd:discover-monorepo --all
# Expected: MONOREPO_INFO.md created with 3 packages

# Step 2: Run /gsd:discover-monorepo --impact (from feature branch)
# Expected: AFFECTED_PACKAGES identified

# Step 3: Run /gsd:execute-monorepo --affected
# Expected: Coordinated execution of affected packages
# Expected: Dashboard shows real-time progress
```

**Test 15b: Build order verification**

```bash
# Expected order: pkg-a → pkg-b → pkg-c
# Verify by checking dashboard updates

# pkg-a should start first and complete
# pkg-b should start only after pkg-a completes
# pkg-c should start only after pkg-b completes
```

**Test 15c: Dashboard real-time updates**

```bash
# During execution, monitor dashboard
watch -n 1 cat .planning/EXECUTION_DASHBOARD.md

# Expected: Status changes visible:
# - All start as "pending"
# - Current package shows "running"
# - Completed packages show "success" or "failed"
```

### Verification Steps

1. **All intermediate files created:**
   ```bash
   ls .planning/MONOREPO_INFO.md
   ls .planning/EXECUTION_DASHBOARD.md
   # Expected: Both files exist after pipeline completion
   ```

2. **Build order followed:**
   ```bash
   # Check dashboard for execution order
   grep -E "started|completed" .planning/EXECUTION_DASHBOARD.md
   # Expected: pkg-a before pkg-b before pkg-c
   ```

### Success Criteria

- [ ] discover-monorepo creates MONOREPO_INFO.md
- [ ] impact-analysis identifies affected packages
- [ ] coordinated-execution runs in dependency order
- [ ] Dashboard shows real-time progress
- [ ] Correct build order followed (deps before dependents)

---

## Test Scenario 16: Backward Compatibility

**Objective:** Verify existing workflows work unchanged without v5.0 features

### Prerequisites

- Existing GSD installation with v4.5 features

### Test Execution

**Test 16a: Without v5.0 features (existing workflows unchanged)**

```bash
# Test that all existing commands still work:
# /gsd:plan-phase
# /gsd:execute-plan
# /gsd:complete-milestone
# /gsd:suggest
# /gsd:progress

# Expected: All commands work identically to v4.5
```

**Test 16b: Partial feature usage (only monorepo, no workflow generation)**

```bash
# User can use monorepo features without workflow factory
# /gsd:discover-monorepo works
# /gsd:execute-monorepo works
# Natural language routing can be ignored

# Expected: Monorepo features work independently
```

**Test 16c: Existing commands work identically**

```bash
# Test core workflow:
# 1. /gsd:plan-phase 1
# 2. /gsd:execute-plan 01-01
# 3. /gsd:complete-milestone

# Expected: No behavior changes from v4.5
```

**Test 16d: No configuration required for v5.0 features**

```bash
# v5.0 features require NO config.json changes
# All features are opt-in via explicit use

# Test with v4.5 config.json (no v5.0 options)
cat > .planning/config.json << 'EOF'
{
  "mode": "interactive",
  "retry": { "enabled": true },
  "verification": { "enabled": true },
  "adaptive": { "enabled": true },
  "suggestions": { "auto_suggest": true }
}
EOF

# Expected: All v4.5 features work
# Expected: v5.0 features available when commands used
```

**Test 16e: Graceful degradation in non-monorepo projects**

```bash
# In single-package project:
cd /tmp/gsd-test-single

# /gsd:discover-monorepo should detect "single-package" type
# /gsd:execute-monorepo should work with single package
# No errors for non-monorepo projects

# Expected: Graceful handling, not error
```

### Verification Steps

1. **Existing commands tested:**
   ```bash
   # Quick smoke test of core commands
   # Expected: All return without errors
   ```

2. **Config compatibility:**
   ```bash
   # v4.5 config should work without modification
   # No "unknown field" warnings for v5.0 options (not added yet)
   ```

### Success Criteria

- [ ] Existing workflows (plan, execute, milestone) work unchanged
- [ ] Partial feature usage works (monorepo without workflow factory)
- [ ] Existing commands (plan-phase, execute-plan, etc.) work identically
- [ ] No configuration required for v5.0 features (all opt-in)
- [ ] Graceful degradation in non-monorepo projects

---

## Test Results Checklist

After running all scenarios, verify:

- [ ] **Scenario 1:** Template registry has 18 templates with proper schema
- [ ] **Scenario 2:** Intent matching uses confidence thresholds correctly
- [ ] **Scenario 3:** Variable extraction handles all patterns
- [ ] **Scenario 4:** Intent routing orchestrates complete flow
- [ ] **Scenario 5:** AI workflow generation validates safety
- [ ] **Scenario 6:** Generated workflows stored and re-used
- [ ] **Scenario 7:** Monorepo detection works for all 7 types
- [ ] **Scenario 8:** Dependency graph construction with cycle detection
- [ ] **Scenario 9:** Impact analysis identifies affected packages
- [ ] **Scenario 10:** /gsd:discover-monorepo command with all flags
- [ ] **Scenario 11:** Coordinated execution follows dependency order
- [ ] **Scenario 12:** Execution dashboard updates in real-time
- [ ] **Scenario 13:** /gsd:execute-monorepo command with all flags
- [ ] **Scenario 14:** End-to-end workflow factory pipeline
- [ ] **Scenario 15:** End-to-end monorepo coordination pipeline
- [ ] **Scenario 16:** Backward compatibility maintained

**Overall Transcendence System Health:**
- [ ] Workflow template system matches intents correctly
- [ ] AI generation produces safe, valid workflows
- [ ] Monorepo detection handles all package manager types
- [ ] Dependency graph enables correct build ordering
- [ ] Coordinated execution respects dependencies
- [ ] Dashboard provides real-time visibility
- [ ] All features are opt-in (no required configuration)
- [ ] Backward compatibility 100% maintained

---

## Cleanup After Tests

```bash
# Remove test monorepo
rm -rf /tmp/gsd-test-monorepo
rm -rf /tmp/gsd-test-single

# Remove any generated test files
rm -f .planning/MONOREPO_INFO.md
rm -f .planning/EXECUTION_DASHBOARD.md
rm -f .planning/.monorepo-cache.json

# Reset config if modified
git checkout .planning/config.json 2>/dev/null || true
```

---

## Troubleshooting

### Intent Matching Not Working

**Check:**
```bash
# Verify template registry exists and loads
cat get-shit-done/workflow-templates/template-index.json | head -20

# Verify intent-matcher workflow exists
ls get-shit-done/workflows/intent-matcher.md

# Check for pattern array in templates
grep '"patterns":' get-shit-done/workflow-templates/template-index.json
```

### Monorepo Not Detected

**Check:**
```bash
# Verify config files exist
ls pnpm-workspace.yaml turbo.json nx.json lerna.json rush.json 2>/dev/null

# Check package.json for workspaces
grep '"workspaces"' package.json

# Verify discover-monorepo workflow exists
ls get-shit-done/workflows/discover-monorepo.md
```

### Coordinated Execution Fails

**Check:**
```bash
# Verify MONOREPO_INFO exists (prerequisite)
ls .planning/MONOREPO_INFO.md

# Check build order in info
grep -A 10 '"build_order"' .planning/MONOREPO_INFO.md

# Verify packages have valid package.json
for pkg in packages/*/; do
  echo "Checking $pkg"
  cat "${pkg}package.json" | head -5
done
```

### Dashboard Not Updating

**Check:**
```bash
# Verify dashboard template exists
ls get-shit-done/templates/execution-dashboard.md

# Check if dashboard file is being written
ls -la .planning/EXECUTION_DASHBOARD.md

# Watch for changes during execution
watch -n 1 ls -la .planning/EXECUTION_DASHBOARD.md
```

### Generated Workflows Not Saving

**Check:**
```bash
# Verify generated directory exists
ls get-shit-done/workflow-templates/generated/

# Check generated-index.json is writable
touch get-shit-done/workflow-templates/generated/test.txt && rm get-shit-done/workflow-templates/generated/test.txt

# Verify workflow was approved (not just generated)
# Approval required before saving
```

---

## Summary

These tests validate that v5.0 Transcendence:

1. **Workflow Template System** - Routes natural language to 18 built-in workflows with confidence-based matching
2. **AI Workflow Generation** - Creates workflows for novel requests with safety validation and human confirmation
3. **Monorepo Discovery** - Auto-detects 7 monorepo types and builds dependency graphs
4. **Coordinated Execution** - Runs tasks across packages in dependency order with failure strategies
5. **End-to-End Pipelines** - Complete flows work from input to execution
6. **Backward Compatibility** - All v4.5 features work unchanged

**Test coverage:** 16 scenarios covering all v5.0 Transcendence features
**Execution:** Manual testing with bash commands
**Outcome:** Complete validation of v5.0 Transcendence milestone
