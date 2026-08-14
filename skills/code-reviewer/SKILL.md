---
name: code-reviewer
description: "Triggered when the user asks to review, audit, or critique code; requests a technical/code review; asks whether an implementation is reasonable or elegant. The review scope is specified by the user (when no scope is given, defaults to the current git workspace: staged + unstaged changes). Review coverage is limited to architecture design, performance optimization, and code smells only."
---

# Code Reviewer Skill

## Purpose

Perform a **technical-only** code review focusing on **architecture design**, **performance optimization**, and **code smells**. Output structured findings in the terminal so the user can act on them before committing.

Out of scope: logic correctness/bug hunting, security, style nits (formatting, naming conventions, import order), and commit message formatting — handle those via other tools/skills.

## When to Trigger

- The user says "review my code", "check this diff", "is this implementation reasonable", "review <path>", etc.
- The user asks for feedback on architecture, design, performance, or code smells for a specified scope.
- The user invokes this skill explicitly by name.

Do **not** trigger for: bug hunting or correctness verification, security audits, commit message generation, or pure style/lint issues.

## Scope

The review scope is **specified by the user** and always takes priority.

- If the user names specific files, directories, code snippets, a branch, or a diff, review exactly that and nothing else.
- If the user gives no explicit scope, default to the current git workspace: review `git diff` (unstaged) and `git diff --cached` (staged) together as a single change set.
- When using the default scope and both diffs are empty, inform the user and stop.
- New files (`A`), modifications (`M`), deletions (`D`), and renames (`R`) are all in scope.
- Do not review committed history unless the user explicitly asks.

## Review Dimensions

Evaluate changes **only** along these three dimensions. Do not assess anything outside them (no correctness, bug, security, or style commentary). Skip a dimension if the change set is too small to be meaningful (e.g., a 3-line typo fix).

### 1. Architecture & Design
- Does the change fit the existing module boundaries, or does it leak concerns across layers?
- Are responsibilities reasonably distributed, or is one function/class doing too much?
- Are dependencies directionally correct (e.g., no cyclic imports, no upward references)?
- Will this change make future evolution harder (tight coupling, hidden shared state, etc.)?

### 2. Performance
- Unnecessary work in hot paths (N+1 queries, repeated I/O, recomputation inside loops)?
- Missing memoization/caching opportunities that are obvious and low-risk?
- Obvious memory or resource leaks (unclosed handles, growing collections)?

### 3. Code Smells
- Is there obvious duplication that should be extracted?
- Is the code overly long or complex (long functions/methods, deeply nested logic, too many parameters)?
- Is there dead code, unreachable code, or copy-paste leftovers?
- Are abstractions at the wrong level (over-engineered or under-abstracted)?
- Is naming so unclear that it obscures intent?

## Output Format

Print findings in the terminal using this exact structure. Be concise; do not write paragraphs of prose.

```
## Code Review

**Scope**: <e.g., "user-specified: src/api.ts" or "3 files, +84/-12 (2 staged, 1 unstaged)">
**Dimensions checked**: <list, e.g., "Architecture, Performance, Code smells">

### [Severity] <file>:<line> — <one-line summary>
<category>: <one of Architecture | Performance | Code smells>
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
| `Critical`   | Structural flaw that breaks module boundaries, causes a severe performance trap (e.g., unbounded queries in hot paths), or a code smell that will clearly lead to a broken contract. Must fix. |
| `Warning`    | Design risk, obvious performance trap, or significant code smell (large duplication, dead code). Should fix. |
| `Suggestion` | Minor architecture/performance/code-smell improvement, future-proofing. Nice to have. |

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

1. **Determine scope**: use the scope explicitly specified by the user. Only if no scope is given, run `git status` and `git diff --stat` to confirm there are staged and/or unstaged changes. If the default scope is empty, stop and tell the user there is nothing to review.
2. **Collect the code**: for a user-specified scope, read those files/snippets directly; for the default git scope, run `git diff` and `git diff --cached` (with `--stat` for file summaries). For binary files or very large diffs, summarize rather than dump full content.
3. **Read context**: for each changed file, read the relevant surrounding code (not just the diff) to judge architecture and intent. Use `git log --oneline -5 -- <file>` when context is unclear.
4. **Evaluate**: walk through the three review dimensions only (Architecture, Performance, Code smells). Do not evaluate correctness, bugs, or security.
5. **Cite**: every finding must include `file:line`. Use the **post-change** line numbers from the diff. For deletions, cite the nearest surviving line or the hunk header.
6. **Score severity**: classify each finding as Critical / Warning / Suggestion per the table above.
7. **Render**: print the output using the format defined in **Output Format**.
8. **Verdict**: end with LGTM, Changes recommended, or Blocking issues found.

## Principles

- **Technical only, and only the three dimensions**. Do not comment on correctness, bugs, security, formatting, whitespace, import order, or commit message style.
- **Cite specifics**. Vague feedback like "consider refactoring" without a file:line is not useful.
- **Be honest about uncertainty**. If you cannot confirm a design or performance issue from the code alone, say so ("cannot verify without runtime context") instead of fabricating a finding.
- **No false positives**. When in doubt, omit the finding. A short, accurate review beats a long, speculative one.
- **Respect existing patterns**. Only flag duplication or design issues when the existing codebase clearly avoids that pattern; don't impose a style that isn't already established.
- **No code rewrites in output**. Describe the fix in prose; do not paste full replacement implementations unless the change is trivial.

## Bad vs Good Findings

| Bad                                                                              | Issue                                          |
| -------------------------------------------------------------------------------- | ---------------------------------------------- |
| `Warning: src/api.ts — code is messy`                                             | No category, no line, no concrete suggestion   |
| `Critical: possible null pointer`                                                 | Out of scope (correctness), no file:line       |
| `Suggestion: rewrite using Strategy pattern`                                     | Over-engineering; the change doesn't warrant it |

| Good                                                                                  |
| ------------------------------------------------------------------------------------- |
| `[Critical] src/api.ts:42 — unbounded query loads full table`                          |
| `category: Performance`                                                                |
| `issue: \`listUsers()\` calls \`db.query("SELECT * FROM users")\` with no LIMIT; on a large table this returns the entire result set into memory.` |
| `suggestion: add a paginated parameter (limit/offset or cursor) and default it to a safe cap like 100.` |
