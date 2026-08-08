---
name: js-deobfuscator
description: Triggered when the user supplies a path to an obfuscated JavaScript file (or a string of obfuscated JS they want analysed) and asks to "deobfuscate", "unobfuscate", "reverse obfuscation", "想看懂这段混淆代码", "反混淆", "还原这段代码" / "看这段代码在做什么", or invokes `/js-deobfuscate`. Use the `synchrony` CLI to clean javascript-obfuscator / obfuscator.io output and produce a human-readable version the user can review.
---

# JavaScript Deobfuscator Skill

## Purpose

Take an **obfuscated JavaScript file** (typical of [javascript-obfuscator](https://github.com/javascript-obfuscator/javascript-obfuscator) / obfuscator.io output) and produce a **cleaned, human-readable version** so the user can read the actual logic. Backed by the [`synchrony`](https://github.com/relative/synchrony) CLI (the `deobfuscator` npm package's binary, v2.4.6).

Out of scope: rewriting the logic, removing protections like anti-debug or environment checks, runtime debugging, source-map recovery, or deobfuscating non-JS languages.

## When to Trigger

Trigger when **all** of the following are true:

- The user provides a **local file path** ending in `.js` / `.mjs` / `.cjs` / `.jsx` / `.ts` / `.tsx` (or a snippet they want saved to a file first).
- The user signals intent to **understand / clean / reverse** obfuscation. Look for phrases like:
  - 中文：`反混淆`、`还原`、`解混淆`、`这段混淆代码在做什么`、`想看懂`、`帮我看看这段代码的逻辑`
  - English: `deobfuscate`, `unobfuscate`, `reverse this obfuscation`, `clean this JS`, `what does this obfuscated code do`
  - Explicit: `/js-deobfuscate <path>`

Do **not** trigger when:

- The user just wants to read, run, lint, or format unobfuscated code.
- The input is HTML/Python/other language (use the appropriate tool instead).
- The user only wants to discuss obfuscation theory — no file is supplied.

## Inputs

| Input        | Required | Notes                                                              |
| ------------ | -------- | ------------------------------------------------------------------ |
| `file_path`  | yes      | Absolute or cwd-relative path to the obfuscated source file.       |
| `--no-rename`| no       | Pass-through flag. By default `synchrony --rename` is enabled.     |
| `--output`   | no       | Override the default `<name>.cleaned.<ext>` output location.        |
| `--config`   | no       | Path to a custom synchrony config JSON (advanced; see synchrony docs). |

If the user pastes a code block instead of a path, write it to `<cwd>/.deobfuscate-input.<ext>` first, then proceed.

## Output

For every run, the skill must produce (or reference) these artifacts:

1. `<original>.cleaned.<ext>` — synchrony's default output, written **next to** the original file. This is the primary deliverable.
2. `<original>.original.<ext>` — a copy of the original, written **before** any transformation, so the diff stays meaningful and the user can recover.
3. A short terminal summary (see **Output Format** below).

If the user passed `--output`, use that path instead of the default and skip creating a `.original.` copy only when the original is already preserved elsewhere (e.g., under git).

## Execution Steps

1. **Pre-flight checks**. Run all of these; stop and report clearly if any fail.
   - `synchrony --version` — must succeed (≥ 2.x). If missing, tell the user to run `npm i -g deobfuscator` (or `pnpm i -g deobfuscator`).
   - `test -f "<file_path>"` — file must exist and be readable.
   - Detect extension. Allowed: `js mjs cjs jsx ts tsx`. If the extension is missing, ask the user or default to `.js` after confirming.
   - `wc -l "<file_path>"` — record line count for the summary.

2. **Back up the original** (unless the user explicitly opts out).
   - `cp -- "<file_path>" "<file_path_without_ext>.original.<ext>"`.
   - Skip this step only when `<file_path>` is tracked by git **and** the user asked not to duplicate.

3. **First pass — deobfuscate with renaming enabled** (this is the default and gives the best readability):
   ```bash
   synchrony --rename --output "<file_path_without_ext>.cleaned.<ext>" "<file_path>"
   ```
   Notes:
   - `--rename` rewrites `_0x...` / single-letter identifiers into readable words. It is recommended; only omit it if the user passed `--no-rename`.
   - `--output` is explicit (rather than relying on synchrony's default `<name>.cleaned.<ext>`) so the path is unambiguous in the summary.
   - **`--output` MUST come before the positional file path.** yargs treats any non-option argument after the file as another positional and re-appends `.cleaned.<ext>` to it, producing `<name>.cleaned.cleaned.<ext>`. If a path starts with `-`, prefix with `./` (e.g. `./-weird.js`); synchrony does not support `--` as an option terminator.
   - Capture **all output** (`2>&1`): synchrony prints per-transformer progress lines (e.g. `Running StringDecoder transformer`) to stdout, while AST errors go to stderr. Surface any `Caught an error while attempting to run AST visitor!` lines prominently.

4. **Sanity-check the result** before reporting success:
   - Confirm `<file_path>.cleaned.<ext>` exists and is non-empty.
   - Diff line counts: `diff <(wc -l < original) <(wc -l < cleaned)` — flag if the cleaned file is suspiciously *larger* than the original (often means the deobfuscation actually inflated the code, which is a transformer-failure smell).
   - Spot-check the first 20 lines and the last 20 lines for obviously broken syntax (mismatched braces, dangling commas, `undefined` tokens where identifiers used to be). If broken, see **Failure Handling** below.

5. **Optional second pass — module-aware re-parse**. If the file uses ESM syntax (`import` / `export`) and synchrony defaulted to `script`, re-run with:
   ```bash
   synchrony --rename --sourceType module --output "<file_path>.cleaned.<ext>" "<file_path>"
   ```
   Same for CommonJS-only files: `--sourceType script` can speed up parsing. Same caveat as step 3: `--output` before the positional.

6. **Report**. Print the **Output Format** below. Include the absolute paths of the original, backup, and cleaned files so the user can open them directly.

## Output Format

Always print this block in the terminal. Keep it tight — no prose paragraphs.

```
## JS Deobfuscation

**Source**: <absolute path to original>
**Backup**:  <absolute path to .original.>
**Output**:  <absolute path to .cleaned.>
**Lines**:   <original> → <cleaned> (Δ +/-N)
**Rename**:  enabled | disabled
**Mode**:    script | module | auto

### Transformations applied (from synchrony stderr)
<one-line list of transformers that ran, e.g.:
 "Simplify, MemberExpressionCleaner, StringDecoder, ControlFlow, DeadCode, Rename">

### Notes
- <bulleted observations worth the user's attention: e.g. "string array fully resolved", "control flow flattened", "dead branches removed", "obfuscation appears layered — cleaned file still contains _0x… identifiers (consider /js-deobfuscate on the cleaned output as a second pass)">

### Verdict
<one of: Clean | Partially cleaned | Could not deobfuscate>
```

### Verdict definitions

| Verdict             | Meaning                                                                                              |
| ------------------- | ---------------------------------------------------------------------------------------------------- |
| `Clean`             | Cleaned file is syntactically valid and clearly more readable than the original.                     |
| `Partially cleaned` | Some transformations applied but obfuscation remnants remain (e.g. nested obfuscation).              |
| `Could not deobfuscate` | synchrony threw or the output is malformed; offer to retry with `--loose` or a custom `--config`. |

## Failure Handling

- **`synchrony: command not found`** → print the install command and stop.
- **`Caught an error while attempting to run AST visitor!`** → surface the full stderr and the original command; suggest retrying with `--loose -l`. If still failing, suggest filing an issue at relative/synchrony with the original file and the full log.
- **Output file empty or syntactically broken** → fall back to: `synchrony --loose --rename --output ...`; if still broken, mark `Could not deobfuscate` and report.
- **Path contains spaces or non-ASCII** → quote with `"$path"` and prefer absolute paths to avoid cwd ambiguity.
- **Permission denied on output** → suggest `--output` to a writable directory.

## Principles

- **Non-destructive.** Always create the `.original.` backup before transforming. Never overwrite the input file in place.
- **One pass by default.** Don't loop synchrony over its own output unless the user asks — `Rename` already covers the readability win, and chained renames can mangle names.
- **Surface what changed.** The summary lists transformers that ran — the user needs to know whether `ControlFlow` was unflattened or skipped.
- **Honest verdict.** If synchrony silently produced garbage, say `Could not deobfuscate`. Don't claim success because a file was written.
- **Don't read the cleaned code for the user.** This skill produces a file; the user reads it. If they ask "what does it do", that's a separate analysis task — handle it normally with the cleaned file as context.
- **Respect `.gitignore` / VCS.** If the directory is a git repo, prefer committing nothing automatically and let the user stage what they want.

## Examples

### Example 1 — straightforward javascript-obfuscator output

User: "帮我反混淆 /tmp/sample.js，我想看它做了什么"

1. `synchrony --version` → `2.4.6` ✅
2. `cp /tmp/sample.js /tmp/sample.original.js`
3. `synchrony --rename --output /tmp/sample.cleaned.js /tmp/sample.js`
4. Verify non-empty, diff line counts.

```
## JS Deobfuscation

**Source**: /tmp/sample.js
**Backup**:  /tmp/sample.original.js
**Output**:  /tmp/sample.cleaned.js
**Lines**:   1 → 11 (Δ +10)
**Rename**:  enabled
**Mode**:    auto

### Transformations applied
Simplify, MemberExpressionCleaner, LiteralMap, DeadCode, Demangle, StringDecoder, Desequence, ControlFlow, Rename

### Notes
- String array `['\x48\x65\x6c\x6c\x6f']` fully decoded to `['Hello']`.
- Member expressions normalised (`console['log']` → `console.log`).
- Hex literals converted to decimal.

### Verdict
Clean
```

### Example 2 — obfuscation survived one pass

User pastes a file produced by a custom packer that synchrony can't fully crack. Run synchrony once, see remnants, mark honestly:

```
### Verdict
Partially cleaned

(consider running again on /tmp/sample.cleaned.js, or opening an issue with the original file)
```