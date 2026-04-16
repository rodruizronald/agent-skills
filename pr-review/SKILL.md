---
name: pr-review
description: Review a GitHub pull request for breaking changes, potential bugs, and specification compliance. Does not run tests or linters unless validating a suspected issue.
---

# Review PR

Reviews a pull request focusing exclusively on **breaking changes, potential bugs, and specification compliance**. This skill is **read-only against the PR** — it must never push commits, modify source files, or post review comments on GitHub. CI/CD handles test and lint execution. The only local execution allowed is running a targeted test to validate a suspected bug or breaking change.

> **Philosophy**: Critical issues only. No style nitpicking, no naming suggestions, no "nice to have" improvements. Every finding must represent a real risk to production.

## When to use

- User asks to review a PR
- User types `/skill:pr-review`
- Before opening a PR to catch issues early

## Required Inputs

1. **PR number** — the pull request to review
2. **Jira ticket description** — the spec this change implements

If `$ARGUMENTS` contains a PR number, use it. Otherwise, ask: "What is the PR number?"
Always ask for the Jira ticket description if not provided.

---

## Steps

### 1. Gather PR Context

```bash
gh pr view <pr-number> --json title,body,headRefName,baseRefName,author
```

Extract the PR title, description, branch name, and author.

### 2. Collect Jira Ticket

If the Jira ticket description was not provided, ask:
"Please paste the Jira ticket description (acceptance criteria and requirements)."

### 3. Get Changed Files and Diff

```bash
gh pr diff <pr-number> --name-only
gh pr diff <pr-number>
```

### 4. Read Full File Context

For each modified file, read the **complete file** (not just the diff) to understand:

- The package and its role in the architecture
- Existing patterns and conventions
- Related interfaces, structs, and error types
- How the changed code integrates with the rest

### 5. Specification Compliance

Extract concrete requirements from the PR description and Jira ticket. For each requirement:

| Status       | Meaning                                               |
| ------------ | ----------------------------------------------------- |
| ✅ Covered   | Requirement fully implemented and verifiable in code  |
| ⚠️ Partial   | Implementation exists but incomplete                  |
| ❌ Missing   | No implementation found                               |
| 🔀 Unplanned | Code change not tied to any requirement (scope creep) |

Map every requirement to specific files and lines. Flag `❌ Missing` and `⚠️ Partial` as 🔴 Critical findings.

### 6. Critical Code Review

Review each changed file for the categories below. **Only report issues that would cause incidents, security vulnerabilities, data corruption, or specification violations in production.**

#### 6.1 Breaking Changes

- Public API signature changes without backward compatibility
- Changed HTTP status codes or response shapes that break clients
- Removed or renamed exported symbols consumed by other packages
- Changed database schema without migration
- Modified interface contracts that break existing implementations
- Changed JSON struct tags that break API consumers

#### 6.2 Concurrency Safety

- Shared state accessed from multiple goroutines without synchronization
- Goroutine leaks (started without shutdown mechanism)
- Missing `context.Context` propagation in goroutines
- Channel operations that can deadlock
- Race conditions on maps (concurrent read/write)

#### 6.3 Error Handling (Critical Only)

- Errors silently discarded in critical paths (DB writes, HTTP calls, file I/O)
- `panic()` in library/service code
- Missing error return from functions that can fail
- Error wrapping that loses context (`%v` instead of `%w`)
- Log-and-return (violates project error handling rules)

#### 6.4 Resource Management

- `http.Response.Body` not closed after HTTP calls
- Database rows/statements not closed
- Context cancellation functions not called (`defer cancel()`)
- File handles not closed
- Missing `defer` for cleanup

#### 6.5 Security

- SQL injection (string concatenation in queries)
- Hardcoded secrets, API keys, passwords
- Missing authentication/authorization checks on new endpoints
- Sensitive data in logs
- Path traversal via unsanitized user input
- Missing input validation on handler boundaries

#### 6.6 Data Integrity

- Missing transactions for multi-step operations that must be atomic
- N+1 query patterns in loops
- Incorrect NULL handling
- GORM audit bypass (`.Exec()`, `.UpdateColumn()` on audited tables)
- Missing `audit.AuditMap(ctx)` when raw SQL is unavoidable

#### 6.7 Context & Timeout Handling

- Missing `context.Context` as first parameter in I/O functions
- Not respecting context cancellation in long-running operations
- Using `context.Background()` where request context should propagate
- Default HTTP client with no timeout

### 7. Validate Suspected Issues (Only If Needed)

If during review you identify a **suspected bug or breaking change** that cannot be confirmed by reading code alone, you may run a **targeted test** to validate the assumption:

```bash
go test -run TestSpecificFunction -count=1 -timeout 30s ./path/to/package/...
```

Rules:

- **Only run tests to validate a specific suspected issue** — never as a general check
- Run the minimum scope needed (single test or single package)
- Document why the test was run and what it confirmed
- If no suspected issues need validation, skip this step entirely

### 8. Save Review

Create the directory and save the review:

```bash
mkdir -p dev/reviews
```

Write to `dev/reviews/pr-<number>.md` using the output format below.

### 9. Present Findings

Show the verdict banner and a summary of findings. Provide the file path to the full review.

### 10. Offer Deep Dive

Ask: "Want me to dive deeper into any specific finding?"

---

## Severity Definitions

Only the following qualify as reportable issues:

| Severity    | Criteria                                                                                                                                    |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| 🔴 Critical | Blocks merge. Bugs, security flaws, data corruption, breaking changes, missing spec requirements. Would cause a production incident.        |
| 🟡 Medium   | Doesn't block merge but creates risk. Missing error handling in non-critical paths, potential edge cases, test gaps for new critical paths. |
| 🟢 Minor    | Low risk but worth noting. Non-obvious behavior, potential future issues, minor contract inconsistencies.                                   |

**Severity is based on production impact, not category.** A nil pointer in a critical path is 🔴; in error logging it's 🟢.

---

## What Is NOT Reported

- Style preferences, naming suggestions
- Import ordering
- Minor refactors or "could be cleaner" comments
- Missing comments or documentation
- Variable naming conventions
- Formatting issues (linter handles this)
- General test coverage (CI handles this)

---

## Output Format

Write to `dev/reviews/pr-<number>.md`:

````markdown
# PR #<number> Review: <title>

**Author:** <author>
**Branch:** <branch> → <base>
**Date:** <YYYY-MM-DD>
**Jira:** <ticket-key or "provided via description">

---

## Specification Compliance

### Requirements Mapping

| #   | Requirement        | Status     | Evidence             |
| --- | ------------------ | ---------- | -------------------- |
| 1   | [requirement text] | ✅ Covered | `file.go:42`         |
| 2   | [requirement text] | ❌ Missing | Not found in changes |

**Coverage:** X/Y requirements met

### Scope Creep

| File   | Change        | Why Unplanned |
| ------ | ------------- | ------------- |
| [file] | [description] | [explanation] |

---

## Findings

### 🔴 Critical

1. **<Issue Title>**
   - **File:** `path/to/file.go:line`
   - **Category:** Breaking Change | Concurrency | Error Handling | Resource Leak | Security | Data Integrity | Spec Violation
   - **Code:**
     ```go
     // problematic code
     ```
   - **Problem:** [What is wrong and what breaks in production]
   - **Fix:**
     ```go
     // corrected code
     ```

### 🟡 Medium

2. **<Issue Title>**
   - **File:** `path/to/file.go:line`
   - **Problem:** [Description]
   - **Recommendation:** [How to fix]

### 🟢 Minor

3. **<Issue Title>**
   - **File:** `path/to/file.go:line`
   - **Note:** [Description]

---

## Positive Observations

- [Good patterns, solid error handling, well-structured code, etc.]

---

## Summary

| Severity    | Count |
| ----------- | ----- |
| 🔴 Critical | X     |
| 🟡 Medium   | X     |
| 🟢 Minor    | X     |

## Verdict

**[APPROVED | CHANGES REQUESTED]**

[If CHANGES REQUESTED: list required actions before merge]
````

---

## Verdict Rules

Start the chat response with the appropriate banner:

**Critical issues or spec violations found:**

```
🔴 CHANGES REQUESTED — [N] critical issues found
[One-line summary of the worst issue]
```

**Spec violations found:**

```
⚠️ SPECIFICATION VIOLATION — [N] requirements missing or partial
```

**Everything clean:**

```
✅ APPROVED — No critical issues found
All [N] requirements met.
```

---

## Behavioral Guidelines

1. **Critical only**: Do not report style, naming, minor refactors, or "nice to have" improvements
2. **Read-only**: Never push commits, modify source files, post GitHub review comments, or alter the PR in any way. Local review file output is the only write allowed
3. **Evidence-based**: Every finding must have a file, line number, and code snippet
4. **Impact-driven**: Explain what breaks in production if the issue ships
5. **Concrete fixes**: Every 🔴 Critical issue includes a working code fix
6. **Spec-first**: The Jira ticket and PR description are the source of truth for scope
7. **Trust the author**: Assume the developer is competent; only flag genuine risks
8. **Praise good work**: Acknowledge solid patterns — reviews should not be purely negative
9. **No blanket execution**: Never run `go test ./...`, `go build ./...`, `go vet ./...`, or any linter. CI/CD owns that
10. **Targeted validation only**: Only run a specific test when you have a concrete suspected bug that cannot be confirmed by reading code alone
