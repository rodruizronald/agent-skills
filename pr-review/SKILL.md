---
name: pr-review
description: Review a GitHub pull request for bugs, security issues, and code style
---

# Review PR

Review a pull request.

## Steps

1. **Get the PR number:**
   If `$ARGUMENTS` is empty, use `AskUserQuestion` to ask: "What is the PR number? (e.g., 631)"
   Use this number as `<pr-number>` in all subsequent commands.

2. Get PR information:

   ```bash
   gh pr view <pr-number> --json title,body,headRefName,baseRefName,author
   ```

3. Ask the user: "Do you want to checkout the PR branch locally?"
   - Options: "Yes" / "No (review remote diff only)"

4. **If user selected "Yes" to checkout:**

   a. Ask: "Do you want to do a soft reset to see all PR changes as staged?"
   - Options: "Yes" / "No (just checkout)"

   b. Checkout the PR branch:

   ```bash
   git checkout <branch> && git pull
   ```

   c. **If user selected "Yes" to reset:**

   ```bash
   git reset --soft $(git merge-base main HEAD)
   git status
   ```

   Inform the user that all PR changes are now staged.

5. Get the list of changed files:

   ```bash
   gh pr diff <pr-number> --name-only
   ```

6. Get the full diff:

   ```bash
   gh pr diff <pr-number>
   ```

7. Read the modified files using the Read tool to understand the full context.

8. **Review the PR focusing on:**

   **Must check:**
   - Architecture: Does the solution follow existing patterns? Interfaces at boundaries, config style consistent with the app.
   - Breaking changes: Does this PR break existing APIs, configs, or behavior? Flag as 🔴 Critical.
   - Test coverage: Are new code paths tested? Flag missing tests as 🔴 Critical.
   - Error handling: Are errors checked and propagated correctly?
   - Nil/null safety: Can anything panic or crash?
   - Resource management: Are connections, files, channels closed?

   **Also check:**
   - Security: Auth, input validation, secrets, injection risks
   - Race conditions and concurrency issues
   - Edge cases: Empty arrays, zero values, boundary conditions
   - Code style: Typos, naming, conventions per CLAUDE.md

   **Assign severity based on impact**, not category. A nil pointer in critical path is 🔴, in error logging is 🟢.

9. **Save the review** to `dev/reviews/pr-<number>.md` using the format below. Create the directory if needed.

10. **Present findings to user** and provide the file path.

11. Ask if the user wants to dive deeper into any specific finding.

## Review Format

```markdown
# PR #<number> Review: <title>

**Author:** <author>
**Branch:** <branch>
**Date:** <YYYY-MM-DD>

---

## Findings

### 🔴 Critical

1. **<Issue Title>**
   - File: `<file>:<line>`
   - <Description>
   - Recommendation: <How to fix>

### 🟡 Medium

2. **<Issue Title>**
   - File: `<file>:<line>`
   - <Description>
   - Recommendation: <How to fix>

### 🟢 Minor

3. **<Issue Title>**
   - File: `<file>:<line>`
   - <Description>

---

## Positive Observations

- <Good patterns, well-tested code, etc.>

---

## Summary

| Severity    | Count |
| ----------- | ----- |
| 🔴 Critical | X     |
| 🟡 Medium   | X     |
| 🟢 Minor    | X     |
```
