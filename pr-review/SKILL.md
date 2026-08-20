---
name: pr-review
description: "Review pull requests against JIRA ticket requirements, then submit findings as a GitHub PR review with inline comments. Use when the user says 'review PR', 'review this code', 'PR review', 'code review', asks to check a branch against tickets, or wants to submit review comments on a pull request."
---

# PR Review

Structured code review workflow: gather context from git + JIRA, analyze against a standard checklist, compile findings by severity, and optionally submit as a GitHub PR review with inline comments.

## Workflow

### Phase 1: Identify scope

1. Determine the branch or PR to review. Check `git status`, open files, and user context.
2. Run `git log --all --oneline -- <file>` on focused files to find commits with JIRA ticket keys (pattern: `CP-NNNNN` or project-specific prefix).
3. Run `git diff main...<branch> --name-only` to get the full list of changed files.
4. Narrow to the files relevant to the ticket(s) — skip unrelated changes merged in from main.

### Phase 2: Gather requirements from JIRA

Use the Atlassian MCP to fetch each ticket:

```
getAccessibleAtlassianResources → get cloudId
getJiraIssue(cloudId, issueIdOrKey, responseContentFormat: "markdown")
```

Extract from each ticket:
- **Summary** and **description** (requirements, acceptance criteria)
- **Parent epic** (for broader context)
- **Status** (In Review, Done, etc.)
- **Ticket notes / comments** (often contain implementation decisions)

Fetch tickets in parallel when there are multiple.

### Phase 3: Read the code

Read all changed source files AND their test files from the branch. Use `git show <branch>:<path>` or `git diff main...<branch> -- <path>` as appropriate.

Also read:
- Configuration files (`application.properties`, config classes) for new properties
- Exception mappers / global handlers that interact with the changed code

### Phase 4: Analyze against checklist

Evaluate every changed file against these 7 areas. **Do not invent requirements** — anchor every finding to ticket ACs, code behavior, or established patterns.

#### 1. Correctness / Completeness
- Does the code implement ALL acceptance criteria from the ticket?
- Are there ACs that are partially implemented or missing entirely?
- Quote the specific AC text when citing gaps.

#### 2. Negative case handling
- Exceptions: are the right exception types thrown/caught?
- Timeouts: are all external calls guarded with explicit timeouts?
- Null safety: can any `.invoke()` / `.get()` NPE on unexpected nulls?
- Error classification: are errors mapped to the correct HTTP status codes?

#### 3. Test coverage
- Do tests match the **intended** behavior (from the ticket), not just current behavior?
- Are both happy path and error paths covered?
- Are edge cases tested (null input, empty collections, timeout, network error)?
- If there's a kill switch / config flag, are both enabled and disabled paths tested?

#### 4. Comments and documentation
- Is there sufficient class-level and method-level Javadoc for an agent to understand context quickly?
- Were useful comments stripped during a refactor without replacement?
- Don't flag missing comments on self-explanatory code.

#### 5. Logging
- **No double logging**: don't log an exception that will be caught and logged by a global exception mapper.
- **Correct log levels**: WARN/ERROR only for real problems. Expected behavior paths use INFO or DEBUG.
- **No PII**: emails, user IDs, tokens must be masked in logs.
- **Parameterized logging**: no string concatenation (`+`) in log statements.

#### 6. Performance
- Are there redundant operations (e.g., double-wrapping immutable collections, N+1 calls)?
- Stay within ticket scope — don't request new infrastructure (Redis, queues) unless the ticket calls for it.
- Flag hardcoded values that should be configurable on the hot path (timeouts, pool sizes).

#### 7. Constants, configuration, kill switches
- New features that could be disruptive MUST have a runtime config toggle (kill switch).
- Hardcoded magic strings/values on the critical path should be externalized to config.
- Duplicated constants across classes should be shared or extracted.

### Phase 5: Compile findings

Assign each finding a severity:

| Emoji | Severity | Meaning |
|-------|----------|---------|
| 🔴 | **CRITICAL** | Blocks merge. Incorrect behavior, missing AC, no kill switch for risky change. |
| 🟠 | **HIGH** | Should fix before merge. Double logging, PII leak, hardcoded timeout on hot path. |
| 🟡 | **MEDIUM** | Should fix, can be follow-up. Redundant work, duplicated constants, missing null guard. |
| 🟢 | **LOW** | Nice to have. Missing Javadoc, minor style, defensive depth limits. |

Present findings as a numbered list with:
- Severity and one-line title
- File path and line number(s)
- Explanation with ticket AC quotes where applicable
- Concrete suggested fix

End with a summary table: `| # | Severity | Category | File:Line | Description |`

### Phase 6: Submit to GitHub (ask first)

**Always ask the user before submitting.** Show the compiled findings and ask:
> "Ready to submit this as a PR review on PR #NNN? (REQUEST_CHANGES / COMMENT only)"

When approved:

1. Find the PR: `gh pr list --head <branch> --json number,url`
2. Get the head commit: `gh pr view <number> --json headRefOid`
3. Get diff hunk boundaries: `gh api repos/{owner}/{repo}/pulls/{number}/files --jq '.[].filename'` and inspect patches
4. Build review JSON (see [review-format.md](review-format.md))
5. **Validate line numbers** — every comment `line` must fall within a diff hunk for that file. Use `grep -n "^@@"` on each file's patch to find hunk ranges.
6. Submit: `gh api repos/{owner}/{repo}/pulls/{number}/reviews --input <file>.json`
7. Clean up the temp JSON file.

## Important rules

- **Never submit without user approval.**
- **Never approve a PR** — only use `REQUEST_CHANGES` or `COMMENT` events.
- Anchor findings to ticket requirements. Don't invent scope.
- When a finding spans multiple lines, comment on the first changed line within the diff hunk.
- If a line falls outside all diff hunks, move the comment to the nearest line inside a hunk and note the actual line in the comment body.

## Additional resources

- For GitHub review JSON format and line-mapping details, see [review-format.md](review-format.md)
