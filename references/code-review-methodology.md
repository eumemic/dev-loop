# Code Review Methodology

This document defines the multi-agent code review process used by the dev-loop plugin. It is injected into subagent prompts to perform structured, confidence-scored code reviews on pull requests.

## Review Process

You are performing a code review on a pull request. Follow these steps precisely.

### Step 1: Eligibility Check

Use a Haiku agent to check if the pull request:
- (a) is closed
- (b) is a draft
- (c) does not need a code review (e.g., automated PR, or very simple and obviously ok)
- (d) already has a code review comment from a previous iteration of this loop

If any of (a)-(c) apply, stop and report that the PR is not eligible for review. For (d), proceed — this is a re-review after fixes.

### Step 2: Gather CLAUDE.md Files

Use a Haiku agent to find all relevant CLAUDE.md files:
- The root CLAUDE.md file (if one exists)
- Any CLAUDE.md files in directories whose files the PR modified

Return only the file paths, not the contents.

### Step 3: Summarize the Change

Use a Haiku agent to view the pull request and return a brief summary of the change.

### Step 4: Multi-Agent Review

Launch 5 parallel Sonnet agents to independently review the change. Each agent should return a list of issues with the reason each was flagged (e.g., CLAUDE.md adherence, bug, historical context, etc.):

1. **CLAUDE.md Compliance** — Audit changes against the CLAUDE.md files. Note that CLAUDE.md is guidance for Claude as it writes code, so not all instructions will be applicable during code review.
2. **Bug Scan** — Read the file changes and do a shallow scan for obvious bugs. Focus on the changes themselves without reading extra context. Focus on large bugs; avoid nitpicks. Ignore likely false positives.
3. **Historical Context** — Read git blame and history of modified code to identify bugs in light of that historical context.
4. **PR Archaeology** — Read previous pull requests that touched these files, and check for any comments on those PRs that may also apply here.
5. **Code Comment Compliance** — Read code comments in the modified files and verify the changes comply with any guidance in those comments.

### Step 5: Confidence Scoring

For each issue found in Step 4, launch a parallel Haiku agent that takes the PR, issue description, and list of CLAUDE.md files (from Step 2), and returns a confidence score. The agent should score each issue on a scale from 0-100:

- **0**: Not confident at all. False positive that doesn't stand up to light scrutiny, or a pre-existing issue.
- **25**: Somewhat confident. Might be real, but may be a false positive. Agent wasn't able to verify. If stylistic, not explicitly called out in CLAUDE.md.
- **50**: Moderately confident. Verified as real, but might be a nitpick or uncommon in practice. Not very important relative to the rest of the PR.
- **75**: Highly confident. Double-checked and very likely a real issue hit in practice. Existing approach is insufficient. Very important to functionality, or directly mentioned in CLAUDE.md.
- **100**: Absolutely certain. Confirmed real issue that will happen frequently. Evidence directly confirms this.

For issues flagged due to CLAUDE.md instructions, the scoring agent should double-check that the CLAUDE.md actually calls out that issue specifically.

### Step 6: Filter

Filter out any issues with a score below 80. If no issues meet this threshold, report that no issues were found.

### False Positive Examples

These should NOT be flagged:
- Pre-existing issues (not introduced by this PR)
- Something that looks like a bug but is not actually a bug
- Pedantic nitpicks that a senior engineer wouldn't call out
- Issues that a linter, typechecker, or compiler would catch (missing imports, type errors, formatting). These are caught by CI.
- General code quality issues (test coverage, security, documentation) unless explicitly required in CLAUDE.md
- Issues called out in CLAUDE.md but explicitly silenced in code (e.g., lint ignore comments)
- Changes in functionality that are likely intentional or directly related to the broader change
- Real issues on lines the user did not modify

### Step 7: Post Results

Use `gh pr comment` to post the review. Format:

**If issues found:**
```
### Code review

Found N issues:

1. <brief description> (CLAUDE.md says "<...>")

<link to file and line with full SHA + line range>

2. <brief description> (bug due to <file and code snippet>)

<link to file and line with full SHA + line range>

Generated with [dev-loop](https://github.com/eumemic/dev-loop)
```

**If no issues found:**
```
### Code review

No issues found. Checked for bugs and CLAUDE.md compliance.

Generated with [dev-loop](https://github.com/eumemic/dev-loop)
```

**Link format**: `https://github.com/OWNER/REPO/blob/FULL_SHA/path/to/file.ext#L10-L15`
- Must use the full git SHA (not abbreviated)
- Use `#` after the file name
- Line range format: `L[start]-L[end]`
- Include at least 1 line of context before and after

### Step 8: Return Structured Results

After posting the comment, return a structured summary to the orchestrator:
- `issue_count`: number of issues above the confidence threshold
- `issues`: list of `{description, confidence, file, line}` objects
- `comment_url`: URL of the posted PR comment

This allows the orchestrator to decide whether to iterate.

## Notes

- Do NOT check build signal or attempt to build/typecheck. Those run separately in CI.
- Use `gh` to interact with GitHub, not web fetch.
- When linking to code, the repo name must match the repo being reviewed.
