---
name: statler-waldorf
description: General code review skill that authors antagonistic pseudo-comments on dev branch changes, evaluates them, checks past lessons, and implements fixes.
---

# Statler-Waldorf

**When to use:** Invoke with `/statler-waldorf` to perform a self-review of dev branch code changes. The skill authors antagonistic pseudo-comments, checks code against accumulated lessons learned, evaluates the findings, plans and implements fixes, extracts new lessons, and commits the results — all in one invocation. Unlike rockem-sockem, this skill does **not** require an open PR.

**Usage:** `/statler-waldorf [--codebase:value] [--item-id:value] [--handle:value] [--quiet[:false|true|force]] [--mode:value] [--agent-attribution[:bool]] [--min-comments:N] [--user-mail:value] [--user-name:value]`

- Parameters use `--name:value` syntax and may appear in **any order**.
- Boolean parameters accept `--name:true`, `--name:false`, or bare `--name` (shorthand for `--name:true`).
- The `--quiet` parameter additionally accepts `--quiet:force` for maximum automation (see Step 4e).
- Any parameter not provided on the command line falls back to the value in `config.json` under the `"defaults"` key. If no default exists, the user is prompted.
- `--item-id` has no config default and will always be prompted if not passed.
- For `--codebase`, the value after the first colon is the full codebase value (important for Windows paths like `--codebase:C:\foo`).
- Unrecognized parameter names are rejected with an error listing valid names.
- Bare positional arguments (no `--` prefix) are rejected with a message showing the correct named syntax.
- Pass `--help` to display a quick reference and exit.

**Examples:**
- `/statler-waldorf --item-id:20525 --quiet --min-comments:5`
- `/statler-waldorf --item-id:20525` — codebase, other params use config defaults
- `/statler-waldorf --item-id:20525 --quiet:force --min-comments:10` — maximum antagonism, zero interruptions
- `/statler-waldorf --item-id:20525 --mode:harsh --min-comments:8` — harsh scrutiny with high minimum
- `/statler-waldorf --item-id:20525 --mode:frontend` — frontend-focused review
- `/statler-waldorf --item-id:20525 --mode:Q` — quick mode (by ID)
- `/statler-waldorf` — prompts for item-id, other params use config defaults

## Overview

This skill orchestrates an end-to-end pipeline of 8 sequential phases — each defined by its own frontmatter file in this skill directory — producing documentation artifacts in the developer's personal notes folder and making code changes + commits in the project repo.

The 8 phases: 🔍 **Scrutinize** → ⏪ **Retro** → 📊 **Evaluate** → 📐 **Formulate** → 🏗️ **Implement** → 🤔 **Sanity Check** → 📓 **Glean** → 📦 **Finalize**

---

## Instructions

### Step 1 — Load Configuration

Read `config.json` from this skill's directory. This file defines:

| Key | Description |
|-----|-------------|
| `project-repo-location` | Local path to the main codebase repo |
| `personal-dir-location` | Local path to the developer's personal files (outside the repo) |
| `user-mail` | Developer's git email |
| `user-name` | Developer's git name |
| `handle` | Short handle used in branch names (overridable via `--handle` param; set to `_` to skip handle filtering) |
| `product-text` | Description of your product (tech stack, architecture, frameworks) — injected into all phase files |
| `defaults` | Object with default parameter values: `codebase`, `quiet`, `mode`, `agent-attribution`, `min-comments`. Missing boolean keys are treated as `false`. Missing string keys trigger a user prompt. Missing `min-comments` defaults to `0`. Missing `mode` defaults to `default`. |

If `config.json` is missing or unreadable, stop and alert the user.

**Lessons learned source:** Read `LESSONSLEARNED.md` from this skill directory. If it does not exist or is empty, fall back to `LESSONSLEARNED.md.example`. If neither file exists or both are empty, set `{lessons-learned-text}` to empty string (this is expected on first run). This value is injected into RETRO.md and GLEAN.md.

**Sanity check rules source:** Read `SANITYCHECK-RULES.md` from this skill directory. If it does not exist or is empty, fall back to `SANITYCHECK-RULES.md.example`. If neither file exists or both are empty, stop and alert the user — sanity check rules are required. Set `{sanity-text}` to the full text content of whichever file was found. This value is injected into SANITYCHECK.md.

**Guidance source:** Read `GUIDANCE.md` from this skill directory. If it does not exist or is empty, fall back to `GUIDANCE.md.example`. If neither file exists or both are empty, stop and alert the user — guidance is required. Set `{guidance-text}` to the full text content of whichever file was found. This value is injected into FORMULATE.md and IMPLEMENT.md.

**Parse and resolve named parameters.** After loading config, parse the invocation arguments using these rules:

1. Split the argument string on spaces.
2. **Help check.** If any token is `--help`, read `HELP.md` from this skill directory and display its contents to the user. Then **stop** — do not continue with the rest of the skill.
3. Each token must start with `--`. If any token lacks the `--` prefix, stop and alert the user that this skill uses named parameters, and show the correct syntax.
4. For each `--` token, split on the **first** colon (`:`) to get the parameter name and value. If there is no colon, the token is a bare boolean flag (value = `true`). Splitting on the first colon is critical for Windows paths (e.g., `--codebase:C:\foo` yields name=`codebase`, value=`C:\foo`).
5. Validate each parameter name against the allowed set: `codebase`, `item-id`, `handle`, `quiet`, `mode`, `agent-attribution`, `min-comments`, `user-mail`, `user-name`. If unrecognized, stop and alert the user with the list of valid names.
6. Reject duplicate parameter names.
7. For boolean parameters (`agent-attribution`), the value must be `true`, `false`, or absent (bare flag = `true`). For `quiet`, the value must be `true`, `false`, `force`, or absent (bare `--quiet` = `true`). For string parameters (`handle`, `user-mail`, `user-name`), any non-empty value is accepted. For `min-comments`, the value must be a string parseable as an integer from 0 to 10 inclusive; bare `--min-comments` (no value) is not allowed — the user must provide a number. For `mode`, the value must be a valid mode name (`default`, `harsh`, `frontend`, `cleancode`, `quick`, `academic`), a valid mode ID (`D`, `H`, `F`, `C`, `Q`, `A` — case-insensitive), or a valid mode symbol (`📋`, `🔥`, `🎨`, `🧹`, `⚡`, `🎓`). Bare `--mode` (no value) is not allowed — the user must provide a mode. Normalize the resolved mode to its lowercase name form (e.g., `H` becomes `harsh`, `🔥` becomes `harsh`).

**Resolve each parameter** using this precedence: command-line value > `defaults` from config > prompt user. Store the resolved values for use in subsequent steps.

**Special resolution for identity parameters:** `--handle`, `--user-mail`, and `--user-name` resolve as: command-line value > corresponding top-level config key (`handle`, `user-mail`, `user-name`). These are **not** in the `defaults` object and are **never** prompted for — if absent from both command line and config, `handle` defaults to empty; `user-mail` and `user-name` remain unset (the Step 2 check will fail unless bypassed with `_`).

| Scenario | Resolved value |
|---|---|
| `--quiet` (bare) | `true` |
| `--quiet:true` | `true` |
| `--quiet:false` | `false` |
| `--quiet:force` | `force` |
| not passed, config default is `true` | `true` |
| not passed, config default is `false` | `false` |
| not passed, config default is `"force"` | `force` |
| not passed, no config default | prompt user |

### Step 2 — Check Requirements

Validate all of the following. If any check fails, notify the user with a clear explanation and prompt them to fix the issue and try again.

**(a)** `project-repo-location` exists and contains accessible git metadata (i.e., it is a git repository).

**(b)** `personal-dir-location` exists and is accessible.

**(c)** If the resolved `user-mail` is `_`, skip this check entirely. Otherwise, the git user email configured in the repo at `project-repo-location` must match the resolved `user-mail` value.

**(d)** If the resolved `user-name` is `_`, skip this check entirely. Otherwise, the git user name configured in the repo at `project-repo-location` must match the resolved `user-name` value.

**(e)** `handle` is resolved (from `--handle` param or config `handle` key). It may be empty — this is allowed, but branch matching in Step 4 will fall back to `item-id` only. If its value is `_`, branch matching will also use `item-id` only (equivalent to empty for matching, but skips confirmation prompts).

**(f)** All of the following files exist in this skill directory and are non-empty:
- `HELP.md`
- `SCRUTINIZE-DEFAULT.md`
- `SCRUTINIZE-HARSH.md`
- `SCRUTINIZE-FRONTEND.md`
- `SCRUTINIZE-CLEANCODE.md`
- `SCRUTINIZE-QUICK.md`
- `SCRUTINIZE-ACADEMIC.md`
- `RETRO.md`
- `EVALUATE.md`
- `FORMULATE.md`
- `IMPLEMENT.md`
- `SANITYCHECK.md`
- `GLEAN.md`
- `FINALIZE.md`
- At least one of `LESSONSLEARNED.md` or `LESSONSLEARNED.md.example`
- At least one of `SANITYCHECK-RULES.md` or `SANITYCHECK-RULES.md.example`
- At least one of `GUIDANCE.md` or `GUIDANCE.md.example`

### Step 3 — Resolve Codebase

Prepare to work with the current code in **one** branch of **one** codebase's git repo.

The codebase is determined by the resolved value of `--codebase` (from the command line, then the config default). If neither provides a value, prompt the user. The codebase can be one of the following values: `project`, `personal`, or an **absolute path** to a directory. When using `project` or `personal`, the codebase can also be specified by its corresponding symbol, by its corresponding ID, or by one of its aliases.

The mapping of codebases, IDs, symbols, locations, and aliases is as follows:

| ID  | Symbol | Codebase | Location | Aliases |
|-----|--------|----------|----------|---------|
| A | 🏢 | project | `project-repo-location` | `work`, `product`, `theirs` |
| Z | 🏠 | personal | `personal-dir-location` | `dev`, `notes`, `mine` |

**Passing a path:** The `--codebase` value may also be an absolute path to a directory (e.g., `--codebase:C:\Users\DavidBarbero\.claude\skills\statler-waldorf`). When a path is passed, validate: (1) the path exists, (2) it is accessible, (3) it contains a git repository. If any check fails, stop and alert the user. The resolved path may or may not match `project-repo-location` or `personal-dir-location`; the skill does not check for overlap.

### Step 4 — Resolve Item ID and Verify Branch

**(a)** If `--item-id` was provided (from the command line), use it. Otherwise, ask the user for an `item-id`. Either way, validate:

- Only letters, numbers, hyphens, and underscores allowed
- No spaces
- No punctuation besides hyphens and underscores
- No emoji
- Minimum 5 characters, maximum 24 characters

**Exception:** The literal value `main` is allowed as an `item-id` despite being shorter than 5 characters — but **only** when the selected codebase is `personal` (🏠) or an absolute path. The `main` item-id is **never** permitted when the codebase is `project` (🏢), because you must not work directly on the main branch of the business codebase. If the user passes `main` with the project codebase, explain the restriction and prompt for a different item ID.

If validation fails, explain which rule failed and prompt again.

**(b) Find the designated branch.** If `item-id` is `main`, the designated branch is `main`. Skip the search logic below and proceed directly to step (c).

Otherwise, if `handle` is non-empty and is not `_`, search for branches whose name contains **both** `handle` and `item-id`. If `handle` is empty or `_`, search for branches containing `item-id` only. If exactly one match exists, that is the "designated branch." If multiple matches exist, list them and ask the user to select one. If none exist, stop and alert the user. If `handle` was empty and a single match was found, confirm the branch with the user before proceeding. If `handle` was `_`, no confirmation is needed — the user explicitly opted out of handle filtering.

**(c) Confirm the designated branch is checked out.** If it is not the currently checked-out branch, stop and alert the user.

**(d) Confirm the branch is ready for edits.** The working tree must not have an in-progress merge, rebase, or other blocking state. If it does, stop and alert the user.

**(e) Establish quiet mode.** The `quiet` parameter resolves to one of three levels: `false`, `true`, or `force`. If it was not resolved at all (neither command line nor config default), ask the user: "Allow all edits for this run? (no / yes / force)" If they confirm with "yes", quiet is `true`. If they respond with "force", quiet is `force`. If they decline (or do not respond affirmatively), quiet is `false`.

The three levels are:

| Level | Behavior |
|-------|----------|
| `false` | **Normal.** The skill may pause to confirm significant actions with the user as it normally would. |
| `true` | **Quiet.** The skill proceeds without pausing for skill-level confirmations — it will not ask the user to approve individual edits, file writes, or git operations. The runtime environment (e.g., Claude Code's tool-permission prompts) may still interrupt. |
| `force` | **Force.** The skill proceeds without pausing for **any** reason. Do **NOT** prompt the user for permission to run any command. Execute all bash commands, git commands, file writes, file reads, and any other tool operations without pausing, without asking, and without waiting for approval. The **only** permitted interruption is a genuine, unrecoverable error that prevents correct execution. The user has explicitly pre-authorized every tool call and shell command for the duration of this run by selecting force mode. |

### Step 5 — Set Time-Bound Variables and Ensure Directories

**(a-f) Capture time values once** (these are reused for the entire run, never refreshed mid-run):

| Placeholder | Format | Example |
|---|---|---|
| `{year}` | 4-digit year | `2026` |
| `{month}` | 2-digit month (zero-padded) | `03` |
| `{day}` | 2-digit day (zero-padded) | `26` |
| `{hour}` | 2-digit hour, 24h (zero-padded) | `14` |
| `{minutes}` | 2-digit minutes (zero-padded) | `07` |
| `{timestamp}` | `{year}{month}{day}-{hour}{minutes}` | `20260326-1407` |

**(g) Safety check:** Verify that `personal-dir-location` is NOT inside `project-repo-location`. If it is, stop and alert the user.

**(h) Resolve agent-attribution-text.** If `--agent-attribution` resolved to `true` (from command line or config default), set `{agent-attribution-text}` to: "Agent attribution is allowed. You may include Co-Authored-By lines in commit messages to credit AI agents that contributed to the changes." Otherwise (default), set it to: "Agent attribution is not allowed. Do not include Co-Authored-By lines or any other agent attribution in commit messages. Strip any existing agent attribution."

**(m) Resolve min-comments.** Store the resolved `--min-comments` value as `{min-comments}` for injection into the scrutinize mode file.

**(n) Resolve mode.** The mode is determined by the resolved value of `--mode` (from the command line, then the config default `defaults.mode`). If neither provides a value, default to `default`. Normalize the value to its lowercase name form using the mapping table below. If the resolved value does not match any valid name, ID, or symbol, stop and alert the user with the list of valid modes.

| ID | Symbol | Mode | Frontmatter File |
|----|--------|------|-----------------|
| D | 📋 | default | `SCRUTINIZE-DEFAULT.md` |
| H | 🔥 | harsh | `SCRUTINIZE-HARSH.md` |
| F | 🎨 | frontend | `SCRUTINIZE-FRONTEND.md` |
| C | 🧹 | cleancode | `SCRUTINIZE-CLEANCODE.md` |
| Q | ⚡ | quick | `SCRUTINIZE-QUICK.md` |
| A | 🎓 | academic | `SCRUTINIZE-ACADEMIC.md` |

Store the resolved mode name as `{mode}` for display purposes and to determine which file to read in Step 6.

**(i-l) Ensure personal subdirectories exist.**

First, derive the **folder name** from `{item-id}`. If `{item-id}` starts with `pbi` or `bug` (case-insensitive) AND the remainder after stripping that prefix consists entirely of digits (with the total original string being at least 5 characters), use only the numeric part as the folder name. Otherwise, use `{item-id}` unchanged. Always derive from the **original user-provided** item-id (from the command line or prompt), never from a branch-name segment discovered during Step 4(b). Store the result as `{folder-name}` and use it in the directory path below and in all subsequent output-path references where the subdirectory is needed.

Examples: `pbi20525` → `20525`; `bug12345` → `12345`; `BUG90210` → `90210`; `trapper-keeper` → `trapper-keeper`; `pbitools` → `pbitools` (remainder is not all digits); `bugbear` → `bugbear`; `wsl2` → `wsl2`; `20314` → `20314`.

Create the full path if any segment is missing:
```
{personal-dir-location}/notes/{year}/{month}/{folder-name}/
```

**Special case when `item-id` is `main`:** Do not use `main` as the subdirectory name. Instead, derive the name from the resolved codebase path by taking its final meaningful path segment — skipping trailing segments like `src`, `source`, `app`, or `root` that do not identify the codebase. For example, if the codebase path is `C:\StudentFirstSIS`, the subdirectory name is `StudentFirstSIS`. If the path is `C:\my-project\src`, the subdirectory name is `my-project` (skipping `src`). The resulting path would be:

```
{personal-dir-location}/notes/{year}/{month}/{codebase-name}/
```

### Steps 6–13 — Execute Frontmatter Phases

Execute each phase **in order**. For each phase:

1. Read the corresponding frontmatter file from this skill directory.
2. Replace all `{placeholders}` with their resolved values (from config, user input, and time variables).
3. Understand the tasks described in the file.
4. Execute the tasks accurately and comprehensively.
5. Produce and save any output files as mandated by the frontmatter file.

**If any phase encounters a problem that prevents 100% accurate execution, stop and alert the user.**

| Step | | Phase | Frontmatter File | Output |
|------|---|-------|------------------|--------|
| 6 | 🔍 | Scrutinize | `SCRUTINIZE-{mode}.md` | `comments_{timestamp}.md` |
| 7 | ⏪ | Retro | `RETRO.md` | `retro_{timestamp}.md` |

**Note on Step 6:** The Scrutinize phase reads `SCRUTINIZE-{mode}.md` where `{mode}` is the resolved mode name from Step 5(n) (e.g., `SCRUTINIZE-DEFAULT.md`, `SCRUTINIZE-HARSH.md`). All other phases (Steps 7–13) are unaffected by the mode — they always read their fixed frontmatter file.
| 8 | 📊 | Evaluate | `EVALUATE.md` | `evaluation_{timestamp}.md` |
| 9 | 📐 | Formulate | `FORMULATE.md` | `plan_{timestamp}.md` |
| 10 | 🏗️ | Implement | `IMPLEMENT.md` | Code changes in the repo |
| 11 | 🤔 | Sanity Check | `SANITYCHECK.md` | `sanity-check_{timestamp}.md` |
| 12 | 📓 | Glean | `GLEAN.md` | `lessons_{timestamp}.md` |
| 13 | 📦 | Finalize | `FINALIZE.md` | Git commits (not pushed) |

All markdown output files are saved to: `{personal-dir-location}/notes/{year}/{month}/{folder-name}/`

### Step 14 — Process Complete

The process is finished. Inform the user:

1. New commits have been created on the designated branch, but the user must still **push** them.
2. All artifacts have been saved to the personal notes directory.

All time-bound and run-scoped variables are now unset. A fresh `/statler-waldorf` invocation will set its own values.

---

## Important Notes

- **One-shot time values.** Time-bound variables are captured once at Step 5 and reused for the entire run. They are not refreshed mid-run.
- **Isolation.** `personal-dir-location` must never be inside `project-repo-location`. The skill checks this and stops if violated.
- **Fail-safe.** On any step failure, the skill stops and alerts the user rather than continuing with partial or incorrect work.
- **Attribution controlled by parameter.** Agent attribution (Co-Authored-By lines) is only included when `--agent-attribution` resolves to `true`. The default is `false` — all agent attribution is stripped.
- **No push.** The skill creates commits but never pushes them. The user pushes manually.
- **No PR required.** Unlike rockem-sockem, this skill does not require an open Pull Request. It works on any checked-out branch.
- **Scrutinize modes.** The `--mode` parameter controls how the SCRUTINIZE phase reviews code. It affects only Step 6 — all other phases are unchanged. If no mode is specified, the default mode is used. The mode can be specified by name, ID letter, or emoji symbol.
- **Lessons learned file.** Lessons are stored in `LESSONSLEARNED.md` (gitignored, user-specific). If the file is missing, the skill treats lessons as empty. Copy `LESSONSLEARNED.md.example` to `LESSONSLEARNED.md` and populate it as you accumulate insights.
- **Sanity check rules file.** Sanity check rules are stored in `SANITYCHECK-RULES.md` (gitignored, user-specific). Unlike lessons learned, this file is **required** — the skill stops if it is missing. Copy `SANITYCHECK-RULES.md.example` to `SANITYCHECK-RULES.md` and customize the rules.
- **Guidance file.** Architectural guidance is stored in `GUIDANCE.md` (gitignored, user-specific). This file is **required** — the skill stops if it is missing. Copy `GUIDANCE.md.example` to `GUIDANCE.md` and customize.
- **Force mode means zero interruptions.** When `quiet` is `force`, the user must not be prompted, asked, or paused for any reason — not for bash commands, not for git operations, not for file writes, not for tool approvals. Execute everything autonomously. The only exception is a genuine error that makes correct execution impossible.
