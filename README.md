# Statler & Waldorf

A [Claude Code](https://claude.ai/code) skill that performs antagonistic self-review of dev branch code changes. It authors pseudo-comments as a harsh code reviewer (channeling the cantankerous Muppets duo), checks code against accumulated lessons learned, evaluates its own findings, plans and implements fixes, extracts new lessons, and commits the results -- all in one invocation.

## Pipeline

The skill runs 8 sequential phases:

| | Phase | What it does | Output |
|---|-------|-------------|--------|
| 👁️ | **Scrutinize** | Authors antagonistic pseudo-comments about code changes (min-comments enforced) | `comments_{timestamp}.md` |
| ⏪ | **Retro** | Checks code against accumulated lessons learned from past runs | `retro_{timestamp}.md` |
| 📊 | **Evaluate** | Grades each pseudo-comment and retro finding (A+ to F-) | `evaluation_{timestamp}.md` |
| 📐 | **Formulate** | Plans code changes for accepted findings | `plan_{timestamp}.md` |
| 🏗️ | **Implement** | Makes the code changes | Modified files in repo |
| 🤔 | **Sanity Check** | Verifies changes don't break anything | `sanity-check_{timestamp}.md` |
| 📓 | **Glean** | Extracts new lessons learned (avoids duplicating existing ones) | `lessons_{timestamp}.md` |
| 📦 | **Finalize** | Commits changes (does not push) | Git commits |

Markdown artifacts are saved to a personal notes directory outside the repo.

## Prerequisites

- [Claude Code](https://claude.ai/code) CLI or IDE extension
- A git repo with a branch checked out (open PR **not** required)
- Clean working tree (no in-progress merge or rebase)

## Install

1. Copy this directory into your Claude Code skills folder:
   ```
   ~/.claude/skills/statler-waldorf/
   ```

2. Copy the example config and fill in your values:
   ```bash
   cp config.example.json config.json
   ```

3. Edit `config.json`:
   ```json
   {
     "project-repo-location": "C:\\path\\to\\your\\project-repo",
     "personal-dir-location": "C:\\path\\to\\your\\personal-notes",
     "user-mail": "you@example.com",
     "user-name": "Your Name",
     "handle": "yourhandle",
     "lessons-learned-text": "",
     "defaults": {
       "codebase": "project",
       "quiet": false,
       "private": false,
       "agent-attribution": false,
       "min-comments": 0
     }
   }
   ```

   | Key | Description |
   |-----|-------------|
   | `project-repo-location` | Local path to the repo you work in |
   | `personal-dir-location` | Path for notes/artifacts (must be outside the repo) |
   | `user-mail` | Must match `git config user.email` in your repo (or `_` to skip check; overridable via `--user-mail` param) |
   | `user-name` | Must match `git config user.name` in your repo (or `_` to skip check; overridable via `--user-name` param) |
   | `handle` | Short handle that appears in your branch names (optional; `_` to skip filtering; overridable via `--handle` param) |
   | `product-text` | Description of your product and tech stack |
   | `sanity-text` | Self-audit questions run after implementation |
   | `guidance-text` | Architectural rules and coding conventions |
   | `lessons-learned-text` | Accumulated lessons from past runs (start empty; grow over time) |
   | `defaults` | Default values for parameters (see Usage below) |

   See `config.example.json` for the full template with all keys.

## Usage

```
/statler-waldorf [--codebase:value] [--item-id:value] [--handle:value] [--quiet[:false|true|force]] [--private[:bool]] [--agent-attribution[:bool]] [--min-comments:N] [--user-mail:value] [--user-name:value]
```

Parameters use `--name:value` syntax, in any order. Booleans accept `--name`, `--name:true`, or `--name:false`. The `--quiet` parameter also accepts `--quiet:force` for maximum automation. Omitted parameters fall back to config defaults.

| Param | Type | Config Default | Effect |
|-------|------|----------------|--------|
| `--codebase` | string | `project` | Repo to review: `project`, `personal`, or absolute path |
| `--item-id` | string | *(none -- prompted)* | Work item identifier (e.g., `20525`) |
| `--handle` | string | *(config `handle` key)* | Developer handle for branch matching; `_` skips filtering |
| `--quiet` | `false` \| `true` \| `force` | `false` | `false`: pause for confirmations. `true`: skip skill confirmations. `force`: skip all interruptions including tool approvals. |
| `--private` | bool | `false` | Reserved for future use (currently no effect) |
| `--agent-attribution` | bool | `false` | Allow Co-Authored-By lines in git commits |
| `--min-comments` | integer | `0` | Minimum pseudo-comments SCRUTINIZE must produce (0-10). Higher values force more antagonistic review. |
| `--user-mail` | string | *(config `user-mail` key)* | Override git email check; `_` skips |
| `--user-name` | string | *(config `user-name` key)* | Override git name check; `_` skips |

Identity parameters (`--handle`, `--user-mail`, `--user-name`) fall back to the corresponding top-level config key rather than the `defaults` object. They are never prompted for.

**Examples:**
- `/statler-waldorf` -- prompts for item-id, uses config defaults for the rest
- `/statler-waldorf --item-id:20525` -- use item ID 20525, config defaults for the rest
- `/statler-waldorf --item-id:20525 --quiet --min-comments:5` -- quiet mode, at least 5 findings
- `/statler-waldorf --item-id:20525 --quiet:force --min-comments:10` -- maximum antagonism, zero interruptions
- `/statler-waldorf --codebase:personal --item-id:main` -- review personal codebase main branch
- `/statler-waldorf --handle:_ --user-mail:_` -- skip handle filtering and email check

When it finishes:
- Commits have been created locally -- you still need to **push**
- Consider adding new lessons from `lessons_{timestamp}.md` to `config.json` `lessons-learned-text` for future RETRO checks

## How it works

Each phase is defined by a markdown file (`SCRUTINIZE.md`, `RETRO.md`, `EVALUATE.md`, etc.) that contains instructions Claude follows at runtime. `SKILL.md` is the orchestrator that ties them together. There is no compiled code -- the entire skill is structured prompts.

## License

MIT
