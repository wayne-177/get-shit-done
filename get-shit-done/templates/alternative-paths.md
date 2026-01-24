# Alternative Paths Template

This template defines the XML schema for alternative strategy entries in `ALTERNATIVE_PATHS.md`.

## Purpose

The Alternative Paths library stores proven backup strategies for when tasks fail. During retry execution, the system:
1. Identifies the failure category (from failure-analysis.md)
2. Looks up alternative strategies matching the task type and failure category
3. Selects an appropriate strategy based on attempt number and prior attempts
4. Executes the alternative approach

## XML Schema

Each alternative strategy entry follows this structure:

```xml
<alternative-path>
  <task-type></task-type>
  <failure-category></failure-category>
  <strategy-name></strategy-name>
  <difficulty></difficulty>
  <risk></risk>
  <approach>

  </approach>
  <when-to-use>

  </when-to-use>
  <success-indicators>

  </success-indicators>
  <prerequisites>

  </prerequisites>
</alternative-path>
```

## Field Definitions

### `<task-type>`
The type of task this strategy applies to. Valid values:
- `bash-command` - Shell command execution
- `file-edit` - File modification operations
- `file-create` - New file creation
- `test-run` - Running test suites
- `build-task` - Build/compile operations
- `deployment` - Deployment tasks
- `validation` - Verification/validation steps
- `git-operation` - Git commands
- `any` - Applies to multiple task types

### `<failure-category>`
The failure category from failure-taxonomy.md. Valid values:
- `tool-failure` - Tool/command not found or crashed
- `validation-failure` - Output doesn't meet requirements
- `dependency-failure` - Missing dependencies or prerequisites
- `environment-failure` - Environment issues (permissions, paths, etc.)
- `configuration-failure` - Config problems
- `any` - Applies to multiple categories

### `<strategy-name>`
Short, descriptive name for this alternative approach (2-5 words)

### `<difficulty>`
Implementation complexity. Valid values:
- `simple` - Straightforward, low-risk change
- `moderate` - Requires some investigation
- `complex` - Significant changes or investigation needed

### `<risk>`
Potential impact if strategy fails. Valid values:
- `safe` - No risk of making things worse
- `moderate` - Could have side effects
- `risky` - Might cause additional problems

### `<approach>`
Detailed description of the alternative strategy (2-4 sentences). Should be specific enough for Claude to execute without additional research.

### `<when-to-use>`
Conditions that make this strategy appropriate (1-2 sentences). Include:
- Specific error patterns
- Environmental conditions
- Attempt number recommendations

### `<success-indicators>`
How to know if this strategy worked (1-2 sentences). Observable outcomes that confirm success.

### `<prerequisites>`
What must be true for this strategy to work (1 sentence). Use "None" if no prerequisites.

## Organization

Strategies should be grouped by:
1. **Primary grouping**: `<task-type>`
2. **Secondary grouping**: `<failure-category>`
3. **Ordering within group**: Simple strategies first, complex ones last

## Example Entry

```xml
<alternative-path>
  <task-type>bash-command</task-type>
  <failure-category>tool-failure</failure-category>
  <strategy-name>Add verbose flags for debugging</strategy-name>
  <difficulty>simple</difficulty>
  <risk>safe</risk>
  <approach>
  Re-run the same command with maximum verbosity flags enabled. For npm: add --verbose --loglevel=verbose. For git: add --verbose. For build tools: add -v or --debug. This exposes detailed error information that helps identify the root cause without changing the actual operation.
  </approach>
  <when-to-use>
  Use on first retry when error message is cryptic or lacks detail. Best for commands that failed without clear error output. Not useful if error is already detailed.
  </when-to-use>
  <success-indicators>
  Command produces detailed output showing exactly where failure occurred. Error messages include stack traces, file paths, or specific failure reasons.
  </success-indicators>
  <prerequisites>
  The command must support verbosity flags. Tool must be installed.
  </prerequisites>
</alternative-path>
```

## Usage Notes

- **Growing library**: Start with 10-15 common strategies, add more as patterns emerge
- **Specificity**: Strategies should be specific to GSD workflows, not generic advice
- **Testability**: Each strategy should produce observable success indicators
- **Learnability**: The system learns which strategies work by tracking outcomes
- **No code**: Strategies are markdown descriptions, not executable code modules

## Integration Points

- **failure-analysis.md**: Provides failure-category input
- **path-selection.md**: Uses this schema to select strategies
- **retry-execution**: Reads selected strategy and executes approach
