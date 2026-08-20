# Skills

This repo contains three Claude Code skills/rules for software engineering workflows.

## 1. Jira Snyk Remediation

**File:** [jira-snyk-remediation.mdc](jira-snyk-remediation.mdc)

Resolves Jira tickets that report a Snyk/dependency vulnerability (e.g. `CP-54783`), end to end: reads the ticket, opens the advisory, finds the minimum safe version, applies the smallest safe dependency fix, updates `yarn.lock`, opens a PR against `main`, and monitors CI.

Key behaviors:
- **Fail-fast**: stops immediately (no improvising) if Jira access, the advisory link, the fix version, or GitHub access is unavailable. Tracks 14 checkpoints (`C0`–`C13`) covering everything from Node version alignment to PR creation and CI follow-up.
- **Scoped fixes**: defaults to only the `package.json`/workspace named in the ticket; nested monorepo manifests are only touched if the ticket or user explicitly says so.
- **Safe git flow**: does the work in a separate `git worktree` off `origin/main` rather than touching the user's current branch/worktree; one branch, one PR, no pushes to `main` or other branches.
- **CI loop**: reads PR checks ~5 minutes after the PR is opened, investigates failures immediately (not on a timer), and applies at most 2 fix/re-run attempts before stopping to report status.
- Triggers automatically on Jira/Snyk-style remediation requests that touch `package.json`, `yarn.lock`, or workflow files (see `globs` in the frontmatter).

## 2. Plan Mode

**File:** [plan-mode.agent.md](plan-mode.agent.md)

Turns a feature/task request into a structured implementation plan document instead of jumping straight to code.

Key behaviors:
- **Always asks clarifying questions first**, one at a time, up to 10, with lettered multiple-choice options and a `✓ [Recommended]` pick justified by patterns found in the actual codebase.
- Stops asking once there's enough clarity, the user says "proceed," or the limit is hit; can also fast-forward on "use defaults."
- **Analyzes the codebase** for existing conventions before recommending an approach.
- **Writes an actual markdown file** (never just prints the plan) to `.github/stories-and-plans/implementation-plans/implementation_plan_[feature_name].md`, structured into 3–6 phases, each with target files, test files, key code snippets, test cases, and technical notes, plus overall technical considerations and a checkbox-based success criteria list.
- **Plan only, never implements** — the assistant's job ends at producing the plan document, not writing the feature code.

## 3. PR Review

**Files:** [pr-review/SKILL.md](pr-review/SKILL.md), [pr-review/review-format.md](pr-review/review-format.md)

Reviews a pull request's diff against its JIRA ticket's requirements and can submit the findings as an inline GitHub PR review.

Key behaviors:
- **6-phase workflow**: identify scope from git/branch diff → pull ticket requirements from JIRA via the Atlassian MCP → read the changed source and test files → analyze against a 7-point checklist → compile severity-ranked findings → (optionally) submit to GitHub.
- **Checklist areas**: correctness/completeness vs. acceptance criteria, negative-case handling (exceptions, timeouts, null safety), test coverage, comments/docs, logging quality (no double-logging, no PII, parameterized logs), performance, and configuration/kill-switches.
- **Severity levels**: 🔴 CRITICAL (blocks merge), 🟠 HIGH, 🟡 MEDIUM, 🟢 LOW — each finding is anchored to a quoted ticket AC or established code pattern, never invented scope.
- **Never submits without asking the user first**, and only ever uses `REQUEST_CHANGES` or `COMMENT` — never `APPROVE`.
- `review-format.md` documents the exact GitHub review JSON payload, including how to map comment line numbers to valid diff hunks before submitting via `gh api .../pulls/{number}/reviews`.
