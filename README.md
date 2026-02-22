# dev-loop

End-to-end development loop that takes a feature idea from concept to a reviewed pull request — with automated code review and fix iterations.

## The Idea

Building a feature typically involves: understanding the codebase, planning, implementing, committing, pushing, opening a PR, getting a code review, fixing issues, and iterating. This plugin automates the entire workflow as a single command.

The code review at each iteration uses a multi-agent approach with confidence-based scoring, ensuring only high-signal issues are surfaced.

## Installation

### From the eumemic marketplace

```
/plugin marketplace add eumemic
/plugin install dev-loop@eumemic
```

### From GitHub

```
/plugin install eumemic/dev-loop
```

## Usage

```
/dev-loop Add a utility function that validates email addresses
/dev-loop Refactor the auth middleware to support JWT tokens
/dev-loop Add pagination to the /api/users endpoint
```

The command accepts a feature description as its argument and handles everything from there.

## How It Works

### Phase 1: Understand & Plan
- Launches Explore subagents to understand the relevant codebase areas
- Designs an implementation plan
- In interactive mode (Claude Code CLI): presents the plan for user approval
- In autonomous mode (SDK agents): proceeds with best judgment

### Phase 2: Branch & Implement
- Creates a feature branch (`feat/<name>`)
- Launches a Sonnet subagent to implement the changes
- Verifies changes were made

### Phase 3: Commit, Push, Open PR
- Commits with a conventional commit message
- Pushes the branch and opens a pull request via `gh`

### Phase 4: Code Review
- Launches a fresh Sonnet subagent with the embedded code-review methodology
- Runs 5 parallel review agents examining different aspects:
  1. CLAUDE.md compliance
  2. Bug scanning
  3. Historical git context
  4. Previous PR comment patterns
  5. Code comment compliance
- Confidence scoring (0-100) filters out false positives (threshold: 80)
- Posts findings as a PR comment

### Phase 5: Assess & Iterate
- **No issues**: Done. Reports success.
- **Actionable issues**: Fixes them, pushes, and re-reviews.
- **Complications**: Surfaces to user for guidance.

## Architecture

```
/dev-loop "feature idea"
    │
    ├── Phase 1: Explore (2-3 subagents) → Plan → User approval
    │
    ├── Phase 2: git checkout -b → Implement (subagent) → Verify
    │
    ├── Phase 3: git commit → git push → gh pr create
    │
    ├── Phase 4: Code Review (subagent with 5 parallel reviewers)
    │   │
    │   └── Phase 5: Issues? ──yes──→ Fix (subagent) → push → loop to Phase 4
    │                    │
    │                    no
    │                    │
    └── Done: report PR URL, summary, iteration count
```

## Plugin Structure

```
dev-loop/
├── .claude-plugin/
│   └── plugin.json                  # Plugin manifest
├── commands/
│   └── dev-loop.md                  # Main orchestration command
├── skills/
│   └── dev-loop/
│       └── SKILL.md                 # Trigger description
├── references/
│   └── code-review-methodology.md   # Embedded code review logic
├── README.md
└── .gitignore
```

## Design Decisions

**Embedded code review**: The review methodology is bundled as a reference file rather than depending on the `code-review` plugin. This ensures each review runs in a fresh subagent context (no accumulated baggage) and the plugin is fully self-contained.

**Interactive vs autonomous**: The command detects interactivity by attempting `AskUserQuestion`. In Claude Code CLI, the user responds. In SDK agents, the command proceeds autonomously.

**Single branch strategy**: One feature branch per run. The PR is created after the first commit; subsequent fix iterations push to the same branch.

**No loop limit**: The review loop continues until the review passes or complications are surfaced to the user.

## Requirements

- `gh` CLI (GitHub CLI) installed and authenticated
- Git repository with a remote

## License

MIT
