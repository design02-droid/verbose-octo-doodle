---
name: pr-review
description: Review a pull request or local diff for bugs, security issues, and test coverage gaps, and write the findings as markdown review comments. Use when asked to "review this PR", "review my changes", "check this diff", or given a PR number, branch, or URL to review.
---

# PR review

Review a change for three things only: **bugs**, **security issues**, and **test coverage**. Report the findings as markdown review comments anchored to file and line. Do not comment on style, naming, or formatting unless it causes a bug.

## 1. Get the change

Work out what to review from what the user gave you:

- **PR number or URL**: fetch the PR's diff and description with whatever GitHub access you have (GitHub MCP tools, or `gh pr diff <n>` and `gh pr view <n>`).
- **Branch**: `git diff $(git merge-base origin/main <branch>)...<branch>` (use the repo's real default branch).
- **Nothing given**: review the local changes, `git diff HEAD` plus untracked files from `git status`. If there are none, review the current branch against the default branch.

Read the PR description, if any, so you know what the change is meant to do. For every changed file, read enough of the surrounding code (callers, definitions, related tests) to judge the change; the diff alone is rarely enough.

## 2. Review

### Bugs
- Logic errors, wrong conditions, off-by-one, inverted checks.
- Null/undefined/empty handling, unhandled errors, swallowed exceptions.
- Race conditions, missing awaits, resource leaks (files, connections, locks).
- Broken contracts: changed signatures or return shapes whose callers were not updated.
- Behaviour that contradicts the PR description.

### Security
- Injection: SQL, shell, path traversal, template, XSS, unsafe deserialization.
- Missing or weakened authentication / authorization checks.
- Secrets, tokens, or credentials in code, config, logs, or error messages.
- Unvalidated input reaching sensitive sinks (file system, eval, network requests / SSRF, redirects).
- Insecure crypto or randomness, disabled TLS verification, overly broad permissions or CORS.
- New dependencies: unpinned, unmaintained, or with known vulnerabilities.

### Test coverage
- New or changed behaviour with no test exercising it.
- Bug fixes with no regression test.
- Tests that cannot fail (no assertions, assertions on mocks only, always-true conditions).
- Missing edge cases for the risky paths found above (errors, empty input, boundaries, permissions).
- If the project has a test command and running it is cheap and safe, run it and report failures with their output.

### Verify before reporting
For each candidate finding, re-read the code and confirm it is real: trace the input, check whether an earlier guard already handles it, and check the tests. Drop anything you cannot back up. Fewer accurate findings beat many speculative ones.

## 3. Output

Write the review in markdown in exactly this shape:

````markdown
## PR review: <PR title, branch, or "local changes">

**Summary:** <one or two sentences on what the change does and the overall verdict>

| Severity | Bugs | Security | Tests |
|---|---|---|---|
| Count | <n> | <n> | <n> |

### Comments

#### 🔴 [Security] `path/to/file.py:42` — <short title>
<What is wrong and why it matters, with the concrete input or scenario that triggers it.>

**Suggestion:**
```<lang>
<minimal fix>
```

#### 🟠 [Bug] `path/to/other.ts:118` — <short title>
...

#### 🟡 [Tests] `path/to/module.go` — <short title>
<Which behaviour is untested and what test to add.>

### Not reviewed
<Anything you skipped or could not check, e.g. generated files, a test suite you could not run. Omit if nothing.>
````

Rules for comments:

- Severity: 🔴 **critical** (exploitable security issue, data loss, crash on a common path), 🟠 **major** (incorrect behaviour, likely bug), 🟡 **minor** (coverage gap, edge case, low-risk issue).
- Order comments by severity, most severe first.
- Anchor every comment to `file:line` (or just `file` for coverage gaps) using the line numbers of the new version of the file.
- Each comment states the problem, the scenario that triggers it, and a concrete fix. Keep suggestions minimal.
- If there are no findings in a category, say so in the summary rather than inventing one.

Only post the comments to GitHub if the user asks; otherwise return the markdown.
