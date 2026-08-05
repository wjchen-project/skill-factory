---
name: code-reviewer
description: Triggered when the user asks to review, audit, or critique code in the current git workspace; requests a technical/code review; asks whether an implementation is reasonable or elegant; or any context involving pre-commit code quality checks on staged and unstaged changes.
---

# Code Reviewer Skill

## Purpose

Perform a **technical-only** code review of the current git workspace (staged + unstaged changes), covering architecture, elegance, and logic correctness. Output structured findings in the terminal so the user can act on them before committing.

Out of scope: style nits (formatting, naming conventions, import order) and commit message formatting — handle those via other tools/skills.

## When to Trigger

- The user says "review my code", "check this diff", "is this implementation reasonable", "find bugs in the changes", etc.
- The user asks for feedback on architecture, design, or refactoring suggestions for current workspace changes.
- The user invokes this skill explicitly by name.

Do **not** trigger for: full-project static audits with no diff context, commit message generation, or pure style/lint issues.

## Scope

- Review `git diff` (unstaged) and `git diff --cached` (staged) together as a single change set.
- If both are empty, inform the user and stop.
- New files (`A`), modifications (`M`), deletions (`D`), and renames (`R`) are all in scope.
- Do not review committed history unless the user explicitly asks.

## Review Dimensions

Evaluate changes along these dimensions. Skip a dimension if the change set is too small to be meaningful (e.g., a 3-line typo fix).

### 1. Architecture & Design
- Does the change fit the existing module boundaries, or does it leak concerns across layers?
- Are responsibilities reasonably distributed, or is one function/class doing too much?
- Are dependencies directionally correct (e.g., no cyclic imports, no upward references)?
- Will this change make future evolution harder (tight coupling, hidden shared state, etc.)?

### 2. Implementation Elegance
- Is there obvious duplication that should be extracted?
- Are there simpler/clearer ways to express the same logic?
- Are abstractions at the right level (not over-engineered, not under-abstracted)?
- Is naming precise and intent-revealing (within reason; naming is not the focus)?

### 3. Logic Correctness & Robustness
- Are edge cases handled (null/empty/zero/boundary values)?
- Are error paths complete (exceptions caught, resources released, transactions rolled back)?
- Is concurrency safe (shared state, locking, atomicity, ordering)?
- Are there off-by-one, type coercion, overflow, or division-by-zero risks?
- Does the code match its apparent intent (no dead code, no copy-paste leftovers)?

### 4. Performance
- Unnecessary work in hot paths (N+1 queries, repeated I/O, recomputation inside loops)?
- Missing memoization/caching opportunities that are obvious and low-risk?
- Obvious memory or resource leaks (unclosed handles, growing collections)?

### 5. Security (only when relevant)
- Input not validated or sanitized at trust boundaries?
- Injection risks (SQL, command, path traversal, XSS)?
- Secrets or sensitive data logged or exposed?
- Missing authorization checks on new endpoints/handlers?

Skip dimension 5 if the change set has no I/O, no auth boundary, and no sensitive data.

## Output Format

Print findings in the terminal using this exact structure. Be concise; do not write paragraphs of prose.

```
## Code Review

**Scope**: <e.g., "3 files, +84/-12 (2 staged, 1 unstaged)">
**Dimensions checked**: <list, e.g., "Architecture, Elegance, Logic, Performance">

### [Severity] <file>:<line> — <one-line summary>
<category>: <one of Architecture | Elegance | Logic | Performance | Security>
<issue>: <what is wrong, in 1-2 sentences, citing the specific construct>
<suggestion>: <concrete fix or alternative, in 1-2 sentences>

(repeat for each finding)

### Summary
- Critical: <n>
- Warning: <n>
- Suggestion: <n>
- Files reviewed: <list>

### Verdict
<one of: LGTM | Changes recommended | Blocking issues found>
```

### Severity Levels

| Level        | Meaning                                                                  |
| ------------ | ------------------------------------------------------------------------ |
| `Critical`   | Correctness bug, data loss, security hole, broken contract. Must fix.    |
| `Warning`    | Likely bug, performance trap, fragile design, missing edge case. Should fix. |
| `Suggestion` | Elegance, readability, minor optimization, future-proofing. Nice to have. |

If no issues are found, output:

```
## Code Review

**Scope**: <...>
**Dimensions checked**: <...>

No issues found.

### Verdict
LGTM
```

## Execution Steps

1. **Pre-check**: run `git status` and `git diff --stat` to confirm there are staged and/or unstaged changes. If both are empty, stop and tell the user there is nothing to review.
2. **Collect diff**: run `git diff` and `git diff --cached` (with `--stat` for file summaries). For binary files or very large diffs, summarize rather than dump full content.
3. **Read context**: for each changed file, read the relevant surrounding code (not just the diff) to judge architecture and intent. Use `git log --oneline -5 -- <file>` when context is unclear.
4. **Evaluate**: walk through the five review dimensions. Skip dimension 5 unless I/O or auth is involved.
5. **Cite**: every finding must include `file:line`. Use the **post-change** line numbers from the diff. For deletions, cite the nearest surviving line or the hunk header.
6. **Score severity**: classify each finding as Critical / Warning / Suggestion per the table above.
7. **Render**: print the output using the format defined in **Output Format**.
8. **Verdict**: end with LGTM, Changes recommended, or Blocking issues found.

## Principles

- **Technical only**. Do not comment on formatting, whitespace, import order, or commit message style.
- **Cite specifics**. Vague feedback like "consider refactoring" without a file:line is not useful.
- **Be honest about uncertainty**. If you cannot confirm a bug from the diff alone, say so ("cannot verify without runtime context") instead of fabricating a finding.
- **No false positives**. When in doubt, omit the finding. A short, accurate review beats a long, speculative one.
- **Respect existing patterns**. Only flag duplication or design issues when the existing codebase clearly avoids that pattern; don't impose a style that isn't already established.
- **No code rewrites in output**. Describe the fix in prose; do not paste full replacement implementations unless the change is trivial.

## Bad vs Good Findings

| Bad                                                                              | Issue                                          |
| -------------------------------------------------------------------------------- | ---------------------------------------------- |
| `Warning: src/api.ts — code is messy`                                             | No category, no line, no concrete suggestion   |
| `Critical: possible null pointer`                                                 | No file:line, no specific construct cited      |
| `Suggestion: rewrite using Strategy pattern`                                     | Over-engineering; the change doesn't warrant it |

| Good                                                                                  |
| ------------------------------------------------------------------------------------- |
| `[Critical] src/api.ts:42 — unbounded query loads full table`                          |
| `category: Performance`                                                                |
| `issue: \`listUsers()\` calls \`db.query("SELECT * FROM users")\` with no LIMIT; on a large table this returns the entire result set into memory.` |
| `suggestion: add a paginated parameter (limit/offset or cursor) and default it to a safe cap like 100.` |