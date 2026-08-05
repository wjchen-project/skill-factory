---
name: git-commit
description: Triggered when the user is about to run git commit, asks to generate or normalize a commit message, inquires about commit message format, or any context involving Conventional Commits and the branch-name constraint.
---

# Git Commit Skill

## Purpose

Enforce a consistent format and quality bar for git commit messages so that history is clear, traceable, and can be used to auto-generate changelogs.

## When to Trigger

- The user asks to run `git commit`, commit code, or write a commit message.
- The user provides a diff or a list of changes and asks for a commit message.
- The user wants to normalize an existing commit message.

## Commit Message Format

```
<type>(<scope>): <subject> (<branch-name>)

## Changes
- <feature>: <description>
- <feature>: <description>

## Details
(Optional, elaborates on each feature above.)

## References
- Issue: #xxx
- Related: xxx
```

### Example

Branch `feat/user-auth`:

```
feat(auth): integrate JWT authentication (feat/user-auth)

## Changes
- Login: add JWT signing and verification middleware
- User: add registration endpoint and password hashing
- Interceptor: return 401 for unauthenticated requests

## References
- Issue: #123
```

## Specification

### Type (required)

Follow [Conventional Commits](https://www.conventionalcommits.org/):

| Type     | Description                                                              |
| -------- | ------------------------------------------------------------------------ |
| feat     | A new feature                                                            |
| fix      | A bug fix                                                                |
| docs     | Documentation only                                                      |
| style    | Formatting changes that do not affect code meaning (whitespace, etc.)   |
| refactor | Code change that neither fixes a bug nor adds a feature                  |
| perf     | Performance improvement                                                  |
| test     | Add or correct tests                                                    |
| build    | Changes to build system or external dependencies (npm, yarn, etc.)      |
| ci       | Changes to CI configuration files and scripts                            |
| chore    | Other changes that don't modify src or test files                        |
| revert   | Reverts a previous commit                                                |

### Scope (optional)

Indicates the area of the codebase affected. Use a noun, e.g. `auth`, `api`, `ui`, `deps`, `config`.

### Branch Name (required)

- Placed at the end of the subject line, wrapped in half-width parentheses: `(<branch-name>)`.
- Full position: `<type>(<scope>): <subject> (<branch-name>)`.
- Obtain via `git branch --show-current`.

### Subject (required)

- No more than 50 characters (excluding type, scope, and branch name).
- Use the project's primary language.
- Start with a verb; describe the core goal of the commit, no granular details.

### Changes (required)

- Group by **feature boundary**, not by file.
- Do **not** list files (e.g. "modified a.ts, b.ts" is forbidden).
- One line per feature, format: `- <feature>: <description>`.

### Details (optional)

Further elaboration on each feature's implementation. May be omitted.

### References (optional)

- `Issue: #xxx` or `Task: xxx`.
- Related PRs, documents, or design files.

## Breaking Changes

If the commit introduces breaking changes, both of the following are required:

1. Append `!` after the type, e.g. `feat(api)!: ...`.
2. Add a `BREAKING CHANGE: ...` entry in the body explaining the cause and migration path.

Example:

```
feat(api)!: restructure user endpoints (feat/api-v2)

## Changes
- User API: response shape changed from flat to nested object

## Details
- Nested structure leaves room for future extension

## References
- Issue: #200

BREAKING CHANGE: the `name` field on `/users` has moved to `profile.name`; clients must update accordingly.
```

## Execution Steps

1. Get the current branch: `git branch --show-current`.
2. Inspect staged and unstaged changes: `git status` and `git diff --stat`.
3. If there are unstaged changes, prompt the user to `git add` first.
4. Identify `type` and `scope` from the changes.
5. Generate the commit message using the template above.
6. Commit: `git commit -m "..."` (for multi-line messages use multiple `-m` flags or a heredoc).
7. Report the result (commit hash, branch name).

## Bad vs Good Examples

| Bad                                              | Issue                                |
| ------------------------------------------------ | ------------------------------------ |
| `fix bug`                                        | Missing type, scope, and branch name |
| `feat: add login`                                | Missing branch name                  |
| `feat: changed 3 files`                          | Listed by file, not by feature       |
| `feat: integrate login [feat/login]`             | Branch name should be at the end, in parentheses |
| `feat: xxx [main]`                               | Should not commit directly to `main` |

| Good                                                    |
| ------------------------------------------------------- |
| `feat(auth): integrate JWT auth (feat/login)`           |
| `fix(api): fix null pointer causing 500 (fix/user-crash)` |
| `refactor(core): extract shared cache layer (refactor/store)` |

## Notes

- Do not commit directly on protected branches (`main` / `master`); warn the user if detected.
- `type` must be lowercase.
- Do not mix multiple types in a single commit; split into multiple commits if needed.
- For multi-line messages in the shell, mind quote escaping; use `\n` carefully.
