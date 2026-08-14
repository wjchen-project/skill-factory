---
name: git-commit
description: Triggered when the user is about to run git commit, asks to generate or normalize a commit message, inquires about commit message format, or any context involving Conventional Commits.
---

# Git Commit Skill

## Purpose

Enforce a consistent format and quality bar for git commit messages so that history is clear, traceable, and can be used to auto-generate changelogs.

> Branch management (e.g. branch naming, protected branches, pushing) is out of scope. The user handles branch-related decisions themselves; this skill focuses only on the commit message and its format.

## When to Trigger

- The user asks to run `git commit`, commit code, or write a commit message.
- The user provides a diff or a list of changes and asks for a commit message.
- The user wants to normalize an existing commit message.

## Commit Message Format

```
<type>(<scope>): <subject>

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

```
feat(auth): integrate JWT authentication

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

### Subject (required)

- No more than 50 characters (excluding type and scope).
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
feat(api)!: restructure user endpoints

## Changes
- User API: response shape changed from flat to nested object

## Details
- Nested structure leaves room for future extension

## References
- Issue: #200

BREAKING CHANGE: the `name` field on `/users` has moved to `profile.name`; clients must update accordingly.
```

## Execution Steps

1. Inspect staged and unstaged changes: `git status` and `git diff --stat`.
2. If there are unstaged changes, prompt the user to `git add` first.
3. Identify `type` and `scope` from the changes.
4. Generate the commit message using the template above.
5. Commit: `git commit -m "..."` (for multi-line messages use multiple `-m` flags or a heredoc).
6. Report the result (commit hash).

## Bad vs Good Examples

| Bad                                          | Issue                                    |
| -------------------------------------------- | ---------------------------------------- |
| `fix bug`                                    | Missing type and scope                   |
| `feat: add login`                            | Missing scope (when applicable)          |
| `feat: changed 3 files`                      | Listed by file, not by feature           |

| Good                                                    |
| ------------------------------------------------------- |
| `feat(auth): integrate JWT auth`                        |
| `fix(api): fix null pointer causing 500`                |
| `refactor(core): extract shared cache layer`            |

## Notes

- `type` must be lowercase.
- Do not mix multiple types in a single commit; split into multiple commits if needed.
- For multi-line messages in the shell, mind quote escaping; use `\n` carefully.
