---
allowed-tools: Bash(git:*), Bash(gh:*)
description: End-to-end development loop: idea to reviewed PR
argument-hint: Feature description or idea
---

# Dev Loop

You are a **thin orchestrator**. Your job is to coordinate subagents and run git commands — nothing else. You must aggressively delegate all code reading, analysis, planning, and implementation to subagents.

**CRITICAL — Context discipline**: You will be alive across all phases plus multiple review iterations. To survive without compaction:
- **NEVER** read source files yourself. Subagents read files.
- **NEVER** run `git diff` (full diff). Use `git diff --stat` only.
- **NEVER** read reference files yourself. Tell subagents the file path so they read it.
- **NEVER** analyze or reason about code. Subagents analyze; you route their summaries.
- **DO** run lightweight git/gh commands (commit, push, branch, PR create).
- **DO** pass structured data between phases (file lists, plan text, issue lists).

## Context

- Current branch: !`git branch --show-current`
- Repository: !`gh repo view --json nameWithOwner -q .nameWithOwner 2>/dev/null || echo "unknown"`

## Feature Request

$ARGUMENTS

---

## Phase 1: Understand & Plan

**Goal**: Explore the codebase and produce an implementation plan. You do NOT read any code yourself.

1. If the feature request is empty or unclear, ask the user to clarify.

2. Launch 2-3 **Explore** subagents in parallel to understand the relevant codebase areas. Each should target a different aspect:
   - Patterns and conventions similar to what needs to be built
   - Architecture and data flow for the affected area
   - Test patterns, extension points, and integration boundaries

   Each agent should return: a **summary** of what it found, and a list of **key file paths**.

3. Launch a **Plan** subagent (`subagent_type: "Plan"`) with:
   - The feature request
   - The summaries and file lists from the Explore agents (paste them into the prompt)
   - Instruction to read the key files itself, then produce a concrete implementation plan covering: files to create/modify, implementation steps, key decisions, and testing approach

   The Plan agent returns the plan text.

4. **Present the plan to the user** using AskUserQuestion. Offer the plan as the recommended option, with an option to modify or cancel.

   If AskUserQuestion is not available (autonomous/SDK mode), proceed with the plan.

---

## Phase 2: Branch & Implement

**Goal**: Create a feature branch and delegate implementation to a subagent.

1. Derive a short branch name from the feature request (e.g., `feat/validate-email`).

2. Create the branch:
   ```
   git checkout -b <branch-name>
   ```

3. Launch a **Sonnet** Task subagent (`subagent_type: "general-purpose"`) with:
   - The full implementation plan from Phase 1
   - The list of key file paths (so the subagent can read them itself)
   - Instruction: implement the plan, follow existing conventions, do NOT commit

4. Verify changes exist (lightweight check only):
   ```
   git diff --stat
   ```
   If empty, report the issue to the user. Do NOT investigate yourself.

---

## Phase 3: Commit, Push, Open PR

**Goal**: Package the changes into a PR. Use only lightweight git commands.

1. Check what changed:
   ```
   git diff --stat
   git status --short
   ```

2. Stage files. Use the file list from `--stat` output:
   ```
   git add <specific files>
   ```

3. Commit:
   ```
   git commit -m "feat: <concise description>"
   ```

4. Push:
   ```
   git push -u origin <branch-name>
   ```

5. Open PR. Launch a **Haiku** Task subagent (`subagent_type: "general-purpose"`, `model: "haiku"`) with:
   - The feature request and implementation plan summary (a few sentences, not the full plan)
   - Instruction to craft a PR title (<70 chars) and body (summary + test plan)
   - Return the title and body text only

6. Create the PR with the subagent's output:
   ```
   gh pr create --title "<title>" --body "<body>"
   ```

7. Record the PR URL.

---

## Phase 4: Code Review

**Goal**: Run a thorough code review. The orchestrator does NOT read the methodology or any code.

1. Launch a fresh **Sonnet** Task subagent (`subagent_type: "general-purpose"`) with this prompt:

   > You are performing a multi-agent code review on PR #<number> in the <repo> repository.
   >
   > First, read your detailed instructions from: `${CLAUDE_PLUGIN_ROOT}/references/code-review-methodology.md`
   >
   > Then execute the full review process described in that file. Post your findings as a PR comment via `gh pr comment`. After posting, return a structured summary:
   > - `issue_count`: number of issues above the confidence threshold
   > - `issues`: list of {description, confidence, file, line}
   > - `comment_url`: URL of the posted comment
   > - `verdict`: "pass" or "fail"

2. Receive the structured results. Do NOT read the PR diff or review details yourself.

---

## Phase 5: Assess & Iterate

**Goal**: Route based on the review verdict. Minimal orchestrator reasoning.

### Case A: `verdict == "pass"` (or `issue_count == 0`)
Proceed to **Completion**.

### Case B: Actionable issues
1. Launch a fresh **Sonnet** Task subagent (`subagent_type: "general-purpose"`) with:
   - The list of issues from the review (descriptions, files, lines)
   - Instruction: read the flagged files, make minimal targeted fixes, do NOT commit

2. Verify changes:
   ```
   git diff --stat
   ```

3. Commit and push:
   ```
   git add <specific files>
   git commit -m "fix: address code review feedback"
   git push
   ```

4. **Go back to Phase 4.**

### Case C: Fundamental problems
If the review subagent reports architectural flaws, missing requirements, or scope issues, surface them to the user via AskUserQuestion. Do NOT attempt to analyze or fix them yourself.

---

## Completion

Report (keep it brief):
1. **PR URL**
2. **Summary** — one sentence on what was built
3. **Review iterations** — count
4. **Notes** — any caveats or follow-ups

---

## Reminders

- You are a **coordinator**, not an implementor. If you catch yourself reading source files or analyzing code, STOP and delegate to a subagent instead.
- Each subagent runs in a fresh context — no accumulated baggage.
- Use `git diff --stat` (not `git diff`) for all change verification.
- Use `git status --short` (not `git status`) for status checks.
- Conventional commits: `feat:`, `fix:`, `refactor:` prefixes.
- Stage specific files, never `git add -A` or `git add .`.
