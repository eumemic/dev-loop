---
name: dev-loop
description: >
  This skill should be used when the user asks to "develop this feature",
  "idea to PR", "dev loop", "implement and review", "build this end to end",
  "take this from idea to PR", "implement, review, and fix",
  "full development cycle", "build and review this",
  "create a PR for this feature", "implement this and open a PR",
  or when the user describes a feature and wants the full development lifecycle
  handled automatically including implementation, code review, and iteration.
---

# Dev Loop

End-to-end development loop that takes an idea from concept to a reviewed pull request.

## What It Does

The `/dev-loop` command orchestrates the full development lifecycle:

1. **Understand & Plan** - Explores the codebase and designs an implementation approach
2. **Branch & Implement** - Creates a feature branch and implements the changes via a focused subagent
3. **Commit, Push, Open PR** - Commits changes, pushes, and opens a pull request
4. **Code Review** - Runs a multi-agent code review on the PR
5. **Fix & Iterate** - If issues are found, fixes them and re-reviews until clean

## Usage

```
/dev-loop Add a utility function that validates email addresses
/dev-loop Refactor the auth middleware to support JWT tokens
/dev-loop Add pagination to the /api/users endpoint
```

The command works in both interactive mode (Claude Code CLI, where it asks for plan approval) and autonomous mode (SDK agents, where it proceeds with best judgment).
