---
allowed-tools: Bash(git:*), Bash(gh:*)
description: End-to-end development loop: idea to reviewed PR
argument-hint: Feature description or idea
---

# Dev Loop

Orchestrate the full development lifecycle: understand the codebase, plan, implement, commit, push, open a PR, run code review, fix issues, and iterate until clean.

## Context

- Current branch: !`git branch --show-current`
- Git status: !`git status --short`
- Repository: !`gh repo view --json nameWithOwner -q .nameWithOwner 2>/dev/null || echo "unknown"`

## Feature Request

$ARGUMENTS

---

## Phase 1: Understand & Plan

**Goal**: Explore the codebase and design an implementation approach.

1. Parse the feature request above. If it is empty or unclear, ask the user to clarify before proceeding.

2. Launch 2-3 **Explore** subagents in parallel to understand the relevant codebase areas. Each agent should target a different aspect:
   - Find code similar to what needs to be built (patterns, conventions, existing utilities)
   - Map the architecture and data flow for the affected area
   - Identify test patterns, extension points, and integration boundaries
   Each agent should return a list of 5-10 key files to read.

3. Read the key files identified by the agents to build deep understanding.

4. Design an implementation plan covering:
   - Which files to create or modify
   - Key implementation decisions and trade-offs
   - Testing approach
   - A clear, sequential list of steps

5. **Present the plan to the user for approval** using AskUserQuestion. Offer the plan as the recommended option, with an option to modify it, or to cancel. If the user wants modifications, incorporate them and re-present.

   If AskUserQuestion is not available (autonomous/SDK mode), proceed with the plan as designed.

---

## Phase 2: Branch & Implement

**Goal**: Create a feature branch and implement the changes.

1. Derive a short, descriptive branch name from the feature request (e.g., `feat/validate-email`, `feat/add-pagination`).

2. Create the feature branch:
   ```
   git checkout -b <branch-name>
   ```

3. Launch a fresh **Sonnet** Task subagent (`subagent_type: "general-purpose"`) with:
   - The approved implementation plan
   - The list of key files and codebase context gathered in Phase 1
   - Clear instruction to implement the changes but NOT commit
   - Instruction to follow existing code conventions and patterns

4. After the subagent returns, verify that changes exist:
   ```
   git diff --stat
   ```
   If no changes were made, investigate and retry or surface the issue to the user.

---

## Phase 3: Commit, Push, Open PR

**Goal**: Package the changes into a PR.

1. Review all changes:
   ```
   git diff
   git status
   ```

2. Stage the relevant files (prefer specific files over `git add -A`):
   ```
   git add <specific files>
   ```

3. Create a commit with a descriptive message following conventional commits style:
   ```
   git commit -m "feat: <concise description of what was built>"
   ```

4. Push the branch:
   ```
   git push -u origin <branch-name>
   ```

5. Open a pull request:
   ```
   gh pr create --title "<short title>" --body "<summary of changes, approach, and any notes>"
   ```

6. Record the PR number and URL for subsequent steps.

---

## Phase 4: Code Review

**Goal**: Run a thorough, multi-agent code review on the PR.

1. Read the code review methodology from `${CLAUDE_PLUGIN_ROOT}/references/code-review-methodology.md`.

2. Launch a fresh **Sonnet** Task subagent (`subagent_type: "general-purpose"`) with:
   - The full code review methodology as its instructions
   - The PR number and repository name
   - Access to `gh` commands for interacting with the PR
   - Instruction to post findings as a PR comment AND return structured results

3. The subagent will:
   - Run the multi-agent code review (eligibility check, 5 parallel reviewers, confidence scoring)
   - Post findings as a PR comment via `gh pr comment`
   - Return structured results: `{issue_count, issues[], comment_url}`

4. Receive the structured results from the subagent.

---

## Phase 5: Assess & Iterate

**Goal**: Decide whether to fix issues, surface complications, or declare success.

Evaluate the code review results:

### Case A: No issues above threshold
The review passed. Proceed to **Completion**.

### Case B: Actionable issues found
1. Analyze the review findings and plan targeted fixes.
2. Launch a fresh **Sonnet** Task subagent (`subagent_type: "general-purpose"`) to implement the fixes:
   - Provide the specific issues to fix (descriptions, files, lines)
   - Provide codebase context from Phase 1
   - Instruct it to make minimal, targeted changes — fix only what was flagged
   - Instruct it NOT to commit
3. Verify changes exist via `git diff --stat`.
4. Commit the fixes:
   ```
   git add <specific files>
   git commit -m "fix: address code review feedback"
   ```
5. Push to the same branch:
   ```
   git push
   ```
6. **Go back to Phase 4** (code review) for the next iteration.

### Case C: Unanticipated complications
If the review surfaces issues that suggest fundamental problems with the approach (architectural flaws, missing requirements, scope creep), surface these to the user and wait for guidance before continuing. Use AskUserQuestion if available.

---

## Completion

Report the results:

1. **PR URL** — link to the pull request
2. **Summary** — what was built and the approach taken
3. **Review iterations** — how many code review cycles were needed
4. **Notes** — any caveats, remaining TODOs, or follow-up suggestions

---

## Guidelines

- **Subagent isolation**: Each implementation and review subagent runs in a fresh context. This prevents context pollution between iterations.
- **Minimal changes**: Fix subagents should only address flagged issues, not refactor or improve adjacent code.
- **No build verification**: Do not run builds, linters, or type checkers. Those are handled by CI.
- **Conventional commits**: Use `feat:`, `fix:`, `refactor:` etc. prefixes.
- **Specific staging**: Always `git add` specific files rather than `git add -A` or `git add .`.
- **Full SHA in links**: When linking to code in PR comments, always use the full git SHA.
