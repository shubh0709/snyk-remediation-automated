# GitHub PR Review JSON Format

## Review payload structure

```json
{
  "commit_id": "<head SHA from gh pr view>",
  "event": "REQUEST_CHANGES",
  "body": "## Review summary\n\n<overview paragraph>\n\n### Summary\n| Severity | Count |\n|--|--|\n| 🔴 CRITICAL | N |\n| 🟠 HIGH | N |\n| 🟡 MEDIUM | N |\n| 🟢 LOW | N |",
  "comments": [
    {
      "path": "src/main/java/com/example/Foo.java",
      "line": 42,
      "side": "RIGHT",
      "body": "🔴 **CRITICAL — Title**\n\nExplanation with ticket quote.\n\n**Fix:** concrete suggestion."
    }
  ]
}
```

## Field reference

| Field | Value | Notes |
|-------|-------|-------|
| `commit_id` | HEAD SHA of the PR | Get from `gh pr view <n> --json headRefOid` |
| `event` | `REQUEST_CHANGES` or `COMMENT` | Never use `APPROVE` |
| `body` | Markdown summary | Shown at top of review |
| `comments[].path` | Relative file path | Must match a file in the PR diff |
| `comments[].line` | Line number in the NEW file | Must be within a diff hunk |
| `comments[].side` | `RIGHT` for new code, `LEFT` for deleted code | Almost always `RIGHT` |
| `comments[].body` | Markdown comment | Prefix with severity emoji |

## Mapping line numbers to diff hunks

GitHub rejects comments on lines outside diff hunks. Before submitting:

1. Get each file's patch:
   ```bash
   gh api repos/{owner}/{repo}/pulls/{number}/files \
     --jq '.[] | select(.filename == "path/to/File.java") | .patch' \
     | grep "^@@"
   ```

2. Parse hunk headers (`@@ -old_start,old_count +new_start,new_count @@`):
   - New file lines covered: `new_start` through `new_start + new_count - 1`

3. For each comment, verify `line` falls within a hunk range.

4. If a line falls in a gap between hunks:
   - Move to the nearest line inside a hunk (prefer the next hunk's first line)
   - Add "(applies to line N above/below)" in the comment body

## Submitting

```bash
gh api repos/{owner}/{repo}/pulls/{number}/reviews --input review.json
```

Always request `full_network` permission for `gh` commands. Clean up the temp JSON after submission.

## Comment body format

```markdown
🔴 **CRITICAL — One-line title**

Ticket [CP-NNNNN](https://certifyos.atlassian.net/browse/CP-NNNNN) states:
> "Quoted acceptance criteria text"

Explanation of the gap between the AC and the code.

**Fix:** Concrete suggestion with code snippet if helpful.
```

Combine related findings on the same line into a single comment to reduce noise.
Group by the highest severity when combining.
