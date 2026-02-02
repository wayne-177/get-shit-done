---
name: gsd:settings
description: Configure GSD workflow toggles and model profile
allowed-tools:
  - Read
  - Write
  - AskUserQuestion
---

<objective>
Allow users to toggle workflow agents on/off and select model profile via interactive settings.

Updates `.planning/config.json` with workflow preferences and model profile selection.
</objective>

<process>

## 1. Validate Environment

```bash
ls .planning/config.json 2>/dev/null
```

**If not found:** Error - run `/gsd:new-project` first.

## 2. Read Current Config

```bash
cat .planning/config.json
```

Parse current values (default to `true` if not present):
- `workflow.research` — spawn researcher during plan-phase
- `workflow.plan_check` — spawn plan checker during plan-phase
- `workflow.verifier` — spawn verifier during execute-phase
- `model_profile` — which model each agent uses (default: `balanced`)
- `git.auto_commit` — auto-commit after each task (default: `true`)
- `git.branching_strategy` — branching approach (default: `"none"`)
- `safety.verify_interfaces` — verify interfaces before coding (default: `true`)
- `safety.verify_requirements` — verify requirements before completion (default: `true`)

## 3. Present Settings

Use AskUserQuestion with current values shown:

```
AskUserQuestion([
  {
    question: "Which model profile for agents?",
    header: "Model",
    multiSelect: false,
    options: [
      { label: "Quality", description: "Opus everywhere except verification (highest cost)" },
      { label: "Balanced (Recommended)", description: "Opus for planning, Sonnet for execution/verification" },
      { label: "Budget", description: "Sonnet for writing, Haiku for research/verification (lowest cost)" }
    ]
  },
  {
    question: "Spawn Plan Researcher? (researches domain before planning)",
    header: "Research",
    multiSelect: false,
    options: [
      { label: "Yes", description: "Research phase goals before planning" },
      { label: "No", description: "Skip research, plan directly" }
    ]
  },
  {
    question: "Spawn Plan Checker? (verifies plans before execution)",
    header: "Plan Check",
    multiSelect: false,
    options: [
      { label: "Yes", description: "Verify plans meet phase goals" },
      { label: "No", description: "Skip plan verification" }
    ]
  },
  {
    question: "Spawn Execution Verifier? (verifies phase completion)",
    header: "Verifier",
    multiSelect: false,
    options: [
      { label: "Yes", description: "Verify must-haves after execution" },
      { label: "No", description: "Skip post-execution verification" }
    ]
  },
  {
    question: "Auto-commit after each task?",
    header: "Git: Auto-Commit",
    multiSelect: false,
    options: [
      { label: "Yes (Recommended)", description: "Atomic commits for each completed task" },
      { label: "No", description: "Stage changes and prompt — you commit manually" }
    ]
  },
  {
    question: "Git branching strategy?",
    header: "Git: Branching Strategy",
    multiSelect: false,
    options: [
      { label: "None (Recommended)", description: "Commit directly to current branch" },
      { label: "Per Phase", description: "Create gsd/phase-{N}-{name} branch for each phase" },
      { label: "Per Milestone", description: "Create gsd/{version}-{name} branch for entire milestone" }
    ]
  },
  {
    question: "Verify interfaces before writing code?",
    header: "Safety: Interface Verification",
    multiSelect: false,
    options: [
      { label: "Yes (Strongly Recommended)", description: "Read schemas/types before writing dependent code" },
      { label: "No", description: "Skip interface verification — faster but riskier" }
    ]
  },
  {
    question: "Verify requirements before marking tasks complete?",
    header: "Safety: Requirement Verification",
    multiSelect: false,
    options: [
      { label: "Yes (Strongly Recommended)", description: "Check requirement text literally before completion" },
      { label: "No", description: "Skip requirement verification — faster but may miss coverage" }
    ]
  }
])
```

**Response parsing:**

Map AskUserQuestion responses to config values:
- Model profile: "Quality" → "quality", "Balanced" → "balanced", "Budget" → "budget"
- Yes/No questions: Response label starts with "Yes" → `true`, starts with "No" → `false`
- Branching strategy: If response includes "Per Phase" → "phase", else if includes "Per Milestone" → "milestone", else → "none"

**Pre-select based on current config values.**

## 4. Update Config

Merge new settings into existing config.json:

```json
{
  ...existing_config,
  "model_profile": "quality" | "balanced" | "budget",
  "workflow": {
    "research": true/false,
    "plan_check": true/false,
    "verifier": true/false
  },
  "git": {
    "auto_commit": true/false,
    "branching_strategy": "none" | "phase" | "milestone"
  },
  "safety": {
    ...existing_safety,
    "verify_interfaces": true/false,
    "verify_requirements": true/false
  }
}
```

Write updated config to `.planning/config.json`.

## 5. Confirm Changes

Display:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► SETTINGS UPDATED
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

| Setting              | Value |
|----------------------|-------|
| Model Profile        | {quality/balanced/budget} |
| Plan Researcher      | {On/Off} |
| Plan Checker         | {On/Off} |
| Execution Verifier   | {On/Off} |
| Auto-Commit          | {On/Off} |
| Branching Strategy   | {None/Per Phase/Per Milestone} |
| Interface Verify     | {On/Off} |
| Requirement Verify   | {On/Off} |

These settings apply to future /gsd:plan-phase and /gsd:execute-phase runs.

Quick commands:
- /gsd:set-profile <profile> — switch model profile
- /gsd:plan-phase --research — force research
- /gsd:plan-phase --skip-research — skip research
- /gsd:plan-phase --skip-verify — skip plan check
```

</process>

<success_criteria>
- [ ] Current config read
- [ ] User presented with 8 settings (profile + 3 workflow toggles + 2 git toggles + 2 safety toggles)
- [ ] Config updated with model_profile and workflow section
- [ ] Changes confirmed to user
</success_criteria>
