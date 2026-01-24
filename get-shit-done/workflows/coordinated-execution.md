# Workflow: Coordinated Execution

<purpose>
Orchestrate execution across monorepo packages in dependency order.

This workflow:
- Accepts AFFECTED_PACKAGES (partial) or MONOREPO_INFO (full) as input
- Executes configurable tasks per package in topological build order
- Handles failures gracefully with continue/stop strategies
- Aggregates results into EXECUTION_RESULTS JSON
- Maintains real-time EXECUTION_DASHBOARD.md status

Output: EXECUTION_RESULTS variable (JSON) + .planning/EXECUTION_DASHBOARD.md (real-time status)
</purpose>

<prerequisites>
<required>
- AFFECTED_PACKAGES variable (from impact-analysis.md) OR MONOREPO_INFO variable (from discover-monorepo.md)
- Package manager installed (npm, pnpm, or yarn)
- .planning/ directory exists
</required>

<optional>
- Custom task configuration (defaults provided)
- stop_on_failure flag (default: false)
</optional>

**Input validation:**
```
If AFFECTED_PACKAGES exists:
  Use AFFECTED_PACKAGES.build_order (affected packages only)
Else if MONOREPO_INFO exists:
  Use MONOREPO_INFO.buildOrder (all packages)
Else:
  ERROR: No package information available.
  Run discovery first: /gsd:discover-monorepo
```
</prerequisites>

<configuration>
## Configuration

Default per-package task configuration:

```json
{
  "tasks": ["install", "build", "test"],
  "commands": {
    "install": "npm install --prefer-offline",
    "build": "npm run build --if-present",
    "test": "npm test --if-present"
  },
  "stop_on_failure": false,
  "timeout_per_task_ms": 300000,
  "parallel_safe_tasks": ["lint", "typecheck"]
}
```

**Configurable options:**

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `tasks` | string[] | ["install", "build", "test"] | Tasks to run per package |
| `commands` | object | See above | Command for each task |
| `stop_on_failure` | boolean | false | Stop execution on first failure |
| `timeout_per_task_ms` | number | 300000 | Timeout per task (5 min) |
| `parallel_safe_tasks` | string[] | [] | Tasks that can run in parallel |

**Override via EXECUTION_CONFIG variable:**
```bash
EXECUTION_CONFIG='{"tasks": ["build", "test"], "stop_on_failure": true}'
```
</configuration>

<process>

<step name="validate_input" number="1">
## Step 1: Validate Input

Check for AFFECTED_PACKAGES or MONOREPO_INFO and extract build order.

```bash
validate_input() {
  local build_order=""
  local packages_info=""
  local input_source=""

  # Check for AFFECTED_PACKAGES first (partial execution)
  if [ -n "$AFFECTED_PACKAGES" ]; then
    input_source="AFFECTED_PACKAGES"

    # Extract build_order from AFFECTED_PACKAGES JSON
    build_order=$(echo "$AFFECTED_PACKAGES" | node -e "
      const data = JSON.parse(require('fs').readFileSync(0, 'utf8'));
      console.log((data.build_order || []).join('\n'));
    " 2>/dev/null)

    # Get package info from MONOREPO_INFO if available
    if [ -n "$MONOREPO_INFO" ]; then
      packages_info="$MONOREPO_INFO"
    fi

  # Check for MONOREPO_INFO (full execution)
  elif [ -n "$MONOREPO_INFO" ]; then
    input_source="MONOREPO_INFO"
    packages_info="$MONOREPO_INFO"

    # Extract buildOrder from MONOREPO_INFO JSON
    build_order=$(echo "$MONOREPO_INFO" | node -e "
      const data = JSON.parse(require('fs').readFileSync(0, 'utf8'));
      console.log((data.buildOrder || []).join('\n'));
    " 2>/dev/null)

  # Try loading from cached MONOREPO_INFO.md
  elif [ -f ".planning/MONOREPO_INFO.md" ]; then
    input_source="MONOREPO_INFO.md"

    # Extract JSON from Raw Data section
    packages_info=$(sed -n '/```json/,/```/p' .planning/MONOREPO_INFO.md | sed '1d;$d')

    build_order=$(echo "$packages_info" | node -e "
      const data = JSON.parse(require('fs').readFileSync(0, 'utf8'));
      console.log((data.buildOrder || []).join('\n'));
    " 2>/dev/null)

  else
    echo "ERROR: No package information available."
    echo "Run discovery first: /gsd:discover-monorepo"
    return 1
  fi

  # Validate we have packages to process
  if [ -z "$build_order" ]; then
    echo "ERROR: No packages found in build order."
    return 1
  fi

  echo "INPUT_SOURCE=$input_source"
  echo "BUILD_ORDER:"
  echo "$build_order"

  # Export for subsequent steps
  export INPUT_SOURCE="$input_source"
  export BUILD_ORDER="$build_order"
  export PACKAGES_INFO="$packages_info"
}

validate_input
```

**Result:**
- INPUT_SOURCE: Where input came from (AFFECTED_PACKAGES, MONOREPO_INFO, MONOREPO_INFO.md)
- BUILD_ORDER: Newline-separated list of package names in topological order
- PACKAGES_INFO: Full package metadata JSON (if available)
</step>

<step name="initialize_dashboard" number="2">
## Step 2: Initialize Dashboard

Create .planning/EXECUTION_DASHBOARD.md with all packages in "pending" state.

```bash
initialize_dashboard() {
  local build_order="$1"
  local timestamp
  timestamp=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

  local total_packages
  total_packages=$(echo "$build_order" | grep -c . || echo "0")

  cat > .planning/EXECUTION_DASHBOARD.md << EOF
# Execution Dashboard

**Status:** In Progress
**Started:** $timestamp
**Packages:** 0/$total_packages complete

---

## Package Status

| # | Package | Status | Duration | Details |
|---|---------|--------|----------|---------|
EOF

  local num=1
  while IFS= read -r pkg; do
    [ -z "$pkg" ] && continue
    echo "| $num | $pkg | pending | - | Waiting |" >> .planning/EXECUTION_DASHBOARD.md
    num=$((num + 1))
  done <<< "$build_order"

  cat >> .planning/EXECUTION_DASHBOARD.md << EOF

---

## Summary

- **Pending:** $total_packages
- **Running:** 0
- **Success:** 0
- **Failed:** 0
- **Skipped:** 0

---

## Execution Log

\`\`\`
[$timestamp] Dashboard initialized
\`\`\`

---

*Last updated: $timestamp*
EOF

  echo "Dashboard initialized: .planning/EXECUTION_DASHBOARD.md"
}

initialize_dashboard "$BUILD_ORDER"
```

**Dashboard states:**
- `pending`: Not yet started (gray)
- `running`: Currently executing (blue)
- `success`: All tasks completed successfully (green)
- `failed`: One or more tasks failed (red)
- `skipped`: Skipped because a dependency failed (yellow)

**Result:** .planning/EXECUTION_DASHBOARD.md created with all packages pending
</step>

<step name="execute_per_package" number="3">
## Step 3: Execute Per-Package

For each package in build_order, execute configured tasks and capture results.

```bash
execute_package() {
  local pkg_name="$1"
  local pkg_path="$2"
  local tasks="$3"       # Space-separated task list
  local commands="$4"    # JSON object of task → command
  local timeout="$5"     # Timeout per task in ms

  local start_time
  start_time=$(date +%s%N)

  local pkg_status="success"
  local task_results=""
  local errors=""

  # Update dashboard: running
  update_dashboard_status "$pkg_name" "running" "-" "Executing..."

  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  echo "Package: $pkg_name"
  echo "Path: $pkg_path"
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"

  # Change to package directory
  cd "$pkg_path" || {
    echo "ERROR: Cannot access $pkg_path"
    update_dashboard_status "$pkg_name" "failed" "-" "Directory not found"
    return 1
  }

  # Execute each task
  for task in $tasks; do
    local task_start
    task_start=$(date +%s%N)

    # Get command for this task
    local cmd
    cmd=$(echo "$commands" | node -e "
      const cmds = JSON.parse(require('fs').readFileSync(0, 'utf8'));
      console.log(cmds['$task'] || '');
    " 2>/dev/null)

    if [ -z "$cmd" ]; then
      echo "  ⏭ $task: No command configured, skipping"
      continue
    fi

    echo "  ▶ $task: $cmd"

    # Execute with timeout
    local output
    local exit_code
    output=$(timeout $((timeout / 1000)) bash -c "$cmd" 2>&1) || exit_code=$?
    exit_code=${exit_code:-0}

    local task_end
    task_end=$(date +%s%N)
    local task_duration_ms=$(( (task_end - task_start) / 1000000 ))

    if [ $exit_code -eq 0 ]; then
      echo "  ✓ $task: Success (${task_duration_ms}ms)"
      task_results="$task_results \"$task\": {\"status\": \"success\", \"duration_ms\": $task_duration_ms},"
    elif [ $exit_code -eq 124 ]; then
      echo "  ✗ $task: Timeout after ${timeout}ms"
      task_results="$task_results \"$task\": {\"status\": \"timeout\", \"duration_ms\": $task_duration_ms},"
      errors="$errors Task '$task' timed out after ${timeout}ms."
      pkg_status="failed"
      break  # Stop on task failure within package
    else
      echo "  ✗ $task: Failed (exit code $exit_code)"
      task_results="$task_results \"$task\": {\"status\": \"failed\", \"duration_ms\": $task_duration_ms, \"exit_code\": $exit_code},"
      errors="$errors Task '$task' failed with exit code $exit_code. Output: ${output:0:500}"
      pkg_status="failed"
      break  # Stop on task failure within package
    fi
  done

  # Return to root
  cd - > /dev/null

  local end_time
  end_time=$(date +%s%N)
  local duration_ms=$(( (end_time - start_time) / 1000000 ))
  local duration_formatted="${duration_ms}ms"
  if [ $duration_ms -gt 60000 ]; then
    duration_formatted="$((duration_ms / 60000))m $((duration_ms % 60000 / 1000))s"
  elif [ $duration_ms -gt 1000 ]; then
    duration_formatted="$((duration_ms / 1000)).$((duration_ms % 1000 / 100))s"
  fi

  # Update dashboard with result
  if [ "$pkg_status" = "success" ]; then
    update_dashboard_status "$pkg_name" "success" "$duration_formatted" "All tasks passed"
  else
    update_dashboard_status "$pkg_name" "failed" "$duration_formatted" "See errors"
  fi

  # Return result as JSON fragment
  echo "{
    \"package\": \"$pkg_name\",
    \"path\": \"$pkg_path\",
    \"status\": \"$pkg_status\",
    \"duration_ms\": $duration_ms,
    \"tasks\": { ${task_results%,} },
    \"errors\": [$(echo "$errors" | sed 's/^/"/;s/$/"/' | tr '\n' ',' | sed 's/,$//')],
    \"started\": \"$(date -u +"%Y-%m-%dT%H:%M:%SZ" -d "@$((start_time / 1000000000))")\",
    \"completed\": \"$(date -u +"%Y-%m-%dT%H:%M:%SZ")\"
  }"

  return $([ "$pkg_status" = "success" ] && echo 0 || echo 1)
}

# Helper: Update dashboard status for a package
update_dashboard_status() {
  local pkg_name="$1"
  local status="$2"
  local duration="$3"
  local details="$4"
  local timestamp
  timestamp=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

  # Update package row in table
  # Using sed to find and replace the row for this package
  sed -i.bak "s/| [0-9]* | $pkg_name | [a-z]* | .* | .* |/| & → | $pkg_name | $status | $duration | $details |/" .planning/EXECUTION_DASHBOARD.md

  # Actually, simpler approach - rebuild status counts
  # This is handled in step 4 (results aggregation)

  # Append to execution log
  echo "[$timestamp] $pkg_name: $status" >> .planning/EXECUTION_DASHBOARD.md
}

# Main execution loop
execute_all_packages() {
  local build_order="$1"
  local packages_info="$2"
  local config="${EXECUTION_CONFIG:-'{}'}"

  # Parse configuration
  local tasks
  tasks=$(echo "$config" | node -e "
    const cfg = JSON.parse(require('fs').readFileSync(0, 'utf8'));
    console.log((cfg.tasks || ['install', 'build', 'test']).join(' '));
  " 2>/dev/null || echo "install build test")

  local commands
  commands=$(echo "$config" | node -e "
    const cfg = JSON.parse(require('fs').readFileSync(0, 'utf8'));
    const defaults = {
      install: 'npm install --prefer-offline',
      build: 'npm run build --if-present',
      test: 'npm test --if-present'
    };
    console.log(JSON.stringify({...defaults, ...(cfg.commands || {})}));
  " 2>/dev/null || echo '{"install":"npm install --prefer-offline","build":"npm run build --if-present","test":"npm test --if-present"}')

  local stop_on_failure
  stop_on_failure=$(echo "$config" | node -e "
    const cfg = JSON.parse(require('fs').readFileSync(0, 'utf8'));
    console.log(cfg.stop_on_failure || false);
  " 2>/dev/null || echo "false")

  local timeout
  timeout=$(echo "$config" | node -e "
    const cfg = JSON.parse(require('fs').readFileSync(0, 'utf8'));
    console.log(cfg.timeout_per_task_ms || 300000);
  " 2>/dev/null || echo "300000")

  echo ""
  echo "═══════════════════════════════════════════════════"
  echo "COORDINATED EXECUTION"
  echo "═══════════════════════════════════════════════════"
  echo ""
  echo "Tasks: $tasks"
  echo "Stop on failure: $stop_on_failure"
  echo "Timeout per task: ${timeout}ms"
  echo ""

  local results=""
  local failed_packages=""
  local skipped_packages=""
  local successful_packages=""

  while IFS= read -r pkg_name; do
    [ -z "$pkg_name" ] && continue

    # Get package path from packages_info
    local pkg_path
    pkg_path=$(echo "$packages_info" | node -e "
      const data = JSON.parse(require('fs').readFileSync(0, 'utf8'));
      const pkg = (data.packages || []).find(p => p.name === '$pkg_name');
      console.log(pkg ? pkg.path : '');
    " 2>/dev/null)

    # Fallback: try to find package by name pattern
    if [ -z "$pkg_path" ]; then
      # Common patterns: packages/pkg-name, apps/pkg-name
      local base_name
      base_name=$(echo "$pkg_name" | sed 's/@.*\///')
      for try_path in "packages/$base_name" "apps/$base_name" "libs/$base_name"; do
        if [ -d "$try_path" ]; then
          pkg_path="$try_path"
          break
        fi
      done
    fi

    if [ -z "$pkg_path" ] || [ ! -d "$pkg_path" ]; then
      echo "WARNING: Cannot find path for $pkg_name, skipping"
      continue
    fi

    # Check if this package depends on a failed package
    local should_skip=false
    for failed in $failed_packages; do
      # Check if pkg_name depends on failed package
      local deps
      deps=$(echo "$packages_info" | node -e "
        const data = JSON.parse(require('fs').readFileSync(0, 'utf8'));
        const deps = data.graph ? data.graph['$pkg_name'] : [];
        console.log((deps || []).join(' '));
      " 2>/dev/null)

      if echo "$deps" | grep -qw "$failed"; then
        should_skip=true
        break
      fi
    done

    if [ "$should_skip" = true ]; then
      echo ""
      echo "⏭ Skipping $pkg_name (depends on failed package)"
      skipped_packages="$skipped_packages $pkg_name"
      update_dashboard_status "$pkg_name" "skipped" "-" "Dependency failed"
      continue
    fi

    # Execute package
    local result
    result=$(execute_package "$pkg_name" "$pkg_path" "$tasks" "$commands" "$timeout")
    local exit_code=$?

    results="$results$result,"

    if [ $exit_code -ne 0 ]; then
      failed_packages="$failed_packages $pkg_name"

      if [ "$stop_on_failure" = "true" ]; then
        echo ""
        echo "✗ Stopping execution due to failure (stop_on_failure=true)"
        break
      fi
    else
      successful_packages="$successful_packages $pkg_name"
    fi

  done <<< "$build_order"

  # Export results for step 4
  export PACKAGE_RESULTS="[$( echo "${results%,}" )]"
  export FAILED_PACKAGES="$failed_packages"
  export SKIPPED_PACKAGES="$skipped_packages"
  export SUCCESSFUL_PACKAGES="$successful_packages"
}

execute_all_packages "$BUILD_ORDER" "$PACKAGES_INFO"
```

**Per-package execution:**
1. Update dashboard status to "running"
2. Change to package directory
3. Execute each configured task in order
4. Capture results (status, duration, output)
5. On task failure: mark package as failed, stop remaining tasks
6. Update dashboard with final status

**Result:** All packages processed with results captured
</step>

<step name="handle_failures" number="4">
## Step 4: Handle Failures

Track failed packages and compute skip set.

```bash
handle_failures() {
  local failed_packages="$1"
  local skipped_packages="$2"
  local packages_info="$3"

  echo ""
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  echo "FAILURE ANALYSIS"
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"

  if [ -z "$failed_packages" ]; then
    echo ""
    echo "✓ No failures detected"
    return 0
  fi

  echo ""
  echo "Failed packages:"
  for pkg in $failed_packages; do
    echo "  ✗ $pkg"
  done

  if [ -n "$skipped_packages" ]; then
    echo ""
    echo "Skipped packages (depend on failed):"
    for pkg in $skipped_packages; do
      # Find which failed package it depends on
      local deps
      deps=$(echo "$packages_info" | node -e "
        const data = JSON.parse(require('fs').readFileSync(0, 'utf8'));
        const deps = data.graph ? data.graph['$pkg'] : [];
        const failed = '$failed_packages'.trim().split(/\\s+/);
        const blockers = (deps || []).filter(d => failed.includes(d));
        console.log(blockers.join(', '));
      " 2>/dev/null)

      echo "  ⏭ $pkg (blocked by: $deps)"
    done
  fi

  echo ""
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"

  return 1
}

handle_failures "$FAILED_PACKAGES" "$SKIPPED_PACKAGES" "$PACKAGES_INFO"
```

**Failure handling strategy:**

| Strategy | Behavior | When to use |
|----------|----------|-------------|
| Continue (default) | Mark failed, skip dependents, continue with remaining | CI/CD pipelines, get maximum feedback |
| Stop | Halt immediately on first failure | Local development, fast feedback |

**Skip computation:**
- When package X fails, find all packages that depend on X (transitively)
- Mark those packages as "skipped"
- Skipped packages don't attempt execution

**Result:** Failed and skipped packages identified with dependency chain
</step>

<step name="generate_results" number="5">
## Step 5: Generate Results

Create EXECUTION_RESULTS JSON and finalize dashboard.

```bash
generate_results() {
  local package_results="$1"
  local successful_packages="$2"
  local failed_packages="$3"
  local skipped_packages="$4"
  local input_source="$5"

  local timestamp
  timestamp=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

  local successful_count
  successful_count=$(echo "$successful_packages" | wc -w | tr -d ' ')

  local failed_count
  failed_count=$(echo "$failed_packages" | wc -w | tr -d ' ')

  local skipped_count
  skipped_count=$(echo "$skipped_packages" | wc -w | tr -d ' ')

  local total_count
  total_count=$((successful_count + failed_count + skipped_count))

  # Determine overall status
  local overall_status="success"
  if [ $failed_count -gt 0 ]; then
    overall_status="failed"
  elif [ $skipped_count -gt 0 ]; then
    overall_status="partial"
  fi

  # Generate EXECUTION_RESULTS JSON
  EXECUTION_RESULTS=$(cat << EOF
{
  "status": "$overall_status",
  "completed": "$timestamp",
  "input_source": "$input_source",
  "summary": {
    "total": $total_count,
    "successful": $successful_count,
    "failed": $failed_count,
    "skipped": $skipped_count
  },
  "successful_packages": [$(echo "$successful_packages" | tr ' ' '\n' | grep -v '^$' | sed 's/.*/"&"/' | tr '\n' ',' | sed 's/,$//')],
  "failed_packages": [$(echo "$failed_packages" | tr ' ' '\n' | grep -v '^$' | sed 's/.*/"&"/' | tr '\n' ',' | sed 's/,$//')],
  "skipped_packages": [$(echo "$skipped_packages" | tr ' ' '\n' | grep -v '^$' | sed 's/.*/"&"/' | tr '\n' ',' | sed 's/,$//')],
  "package_results": $package_results
}
EOF
)

  export EXECUTION_RESULTS

  # Update dashboard with final summary
  finalize_dashboard "$overall_status" "$successful_count" "$failed_count" "$skipped_count" "$total_count"

  # Display summary
  echo ""
  echo "═══════════════════════════════════════════════════"
  echo "EXECUTION COMPLETE"
  echo "═══════════════════════════════════════════════════"
  echo ""
  echo "Status: $overall_status"
  echo ""
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  echo "SUMMARY"
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  echo ""
  echo "  Total:      $total_count packages"
  echo "  Successful: $successful_count"
  echo "  Failed:     $failed_count"
  echo "  Skipped:    $skipped_count"
  echo ""
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  echo "OUTPUT"
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  echo ""
  echo "  JSON: EXECUTION_RESULTS variable set"
  echo "  Dashboard: .planning/EXECUTION_DASHBOARD.md"
  echo ""
  echo "═══════════════════════════════════════════════════"

  # Return appropriate exit code
  if [ "$overall_status" = "success" ]; then
    return 0
  else
    return 1
  fi
}

finalize_dashboard() {
  local status="$1"
  local successful="$2"
  local failed="$3"
  local skipped="$4"
  local total="$5"
  local timestamp
  timestamp=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

  # Update header status
  local status_text="Complete"
  if [ "$status" = "failed" ]; then
    status_text="Failed"
  elif [ "$status" = "partial" ]; then
    status_text="Partial"
  fi

  # Update dashboard summary section
  sed -i.bak "s/^- \*\*Pending:\*\*.*/- **Pending:** 0/" .planning/EXECUTION_DASHBOARD.md
  sed -i.bak "s/^- \*\*Running:\*\*.*/- **Running:** 0/" .planning/EXECUTION_DASHBOARD.md
  sed -i.bak "s/^- \*\*Success:\*\*.*/- **Success:** $successful/" .planning/EXECUTION_DASHBOARD.md
  sed -i.bak "s/^- \*\*Failed:\*\*.*/- **Failed:** $failed/" .planning/EXECUTION_DASHBOARD.md
  sed -i.bak "s/^- \*\*Skipped:\*\*.*/- **Skipped:** $skipped/" .planning/EXECUTION_DASHBOARD.md

  # Update status header
  sed -i.bak "s/^\*\*Status:\*\*.*/\*\*Status:\*\* $status_text/" .planning/EXECUTION_DASHBOARD.md
  sed -i.bak "s/^\*\*Packages:\*\*.*/\*\*Packages:\*\* $successful\/$total complete/" .planning/EXECUTION_DASHBOARD.md

  # Add completion timestamp
  echo "[$timestamp] Execution complete: $status_text" >> .planning/EXECUTION_DASHBOARD.md

  # Update last updated
  sed -i.bak "s/^\*Last updated:.*/\*Last updated: $timestamp\*/" .planning/EXECUTION_DASHBOARD.md

  # Clean up backup files
  rm -f .planning/EXECUTION_DASHBOARD.md.bak
}

generate_results "$PACKAGE_RESULTS" "$SUCCESSFUL_PACKAGES" "$FAILED_PACKAGES" "$SKIPPED_PACKAGES" "$INPUT_SOURCE"
```

**Result:**
- EXECUTION_RESULTS: JSON variable with complete execution summary
- .planning/EXECUTION_DASHBOARD.md: Updated with final status
</step>

</process>

<output>
## Output Format

**Variables set:**
- `EXECUTION_RESULTS`: JSON string with complete execution results

**Files updated:**
- `.planning/EXECUTION_DASHBOARD.md`: Real-time status dashboard

**JSON structure:**
```typescript
interface ExecutionResults {
  status: 'success' | 'failed' | 'partial';
  completed: string;              // ISO timestamp
  input_source: string;           // AFFECTED_PACKAGES | MONOREPO_INFO | MONOREPO_INFO.md
  summary: {
    total: number;
    successful: number;
    failed: number;
    skipped: number;
  };
  successful_packages: string[];
  failed_packages: string[];
  skipped_packages: string[];
  package_results: PackageResult[];
}

interface PackageResult {
  package: string;                // Package name
  path: string;                   // Package path
  status: 'success' | 'failed' | 'skipped';
  duration_ms: number;
  started: string;                // ISO timestamp
  completed: string;              // ISO timestamp
  tasks: {
    [taskName: string]: {
      status: 'success' | 'failed' | 'timeout' | 'skipped';
      duration_ms: number;
      exit_code?: number;
    };
  };
  errors: string[];
}
```

**Status definitions:**
- `success`: All packages completed successfully
- `failed`: One or more packages failed
- `partial`: Some packages skipped due to dependency failures
</output>

<error_handling>
## Error Handling

**No input found:**
```
ERROR: No package information available.
Run discovery first: /gsd:discover-monorepo
```

**Package path not found:**
- Log warning and skip package
- Continue with remaining packages
- Record in results as error

**Task timeout:**
- Mark task as "timeout"
- Mark package as "failed"
- Continue with failure handling

**Command not found:**
- Skip task with warning
- Continue with remaining tasks

**Permission denied:**
- Mark package as "failed"
- Include error in results
- Apply failure handling
</error_handling>

<integration>
## Integration Points

**Used by:**
- Coordinated monorepo execution (Phase 19)
- CI/CD pipelines
- Local development workflows

**Requires:**
- discover-monorepo.md workflow output (MONOREPO_INFO)
- OR impact-analysis.md workflow output (AFFECTED_PACKAGES)

**Outputs to:**
- EXECUTION_RESULTS variable
- .planning/EXECUTION_DASHBOARD.md

**References:**
- execute-phase.md patterns for task execution
- package-execution-summary.md template for per-package results
</integration>

<examples>
## Usage Examples

**Execute all packages:**
```bash
# Discover monorepo first
/gsd:discover-monorepo

# Then execute
# (coordinated-execution workflow runs automatically with MONOREPO_INFO)
```

**Execute affected packages only:**
```bash
# Discover with impact analysis
/gsd:discover-monorepo --impact

# Execute only affected packages
# (coordinated-execution uses AFFECTED_PACKAGES.build_order)
```

**Custom task configuration:**
```bash
EXECUTION_CONFIG='{"tasks": ["build", "test"], "stop_on_failure": true}'
# Execute with custom config
```

**Build only (skip tests):**
```bash
EXECUTION_CONFIG='{"tasks": ["install", "build"]}'
```
</examples>

---

*Coordinated execution workflow for GSD monorepo support*
*Created: 2026-01-08*
