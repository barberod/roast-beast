### How to Use the Statler-Waldorf Skill

Self-review dev branch changes with antagonistic pseudo-comments, lessons-learned checks, and automated fixes.

```
/statler-waldorf [--codebase:value] [--item-id:value] [--handle:value] [--quiet[:false|true|force]] [--mode:value] [--agent-attribution[:bool]] [--min-comments:N] [--user-mail:value] [--user-name:value]
```

| Param | Type | Default | What it does |
|-------|------|---------|--------------|
| `--codebase` | string | `project` | Repo to review: `project`, `personal`, or absolute path |
| `--item-id` | string | *(prompted)* | Work item ID (e.g., `20525`) |
| `--handle` | string | *(config)* | Developer handle for branch matching; `_` skips filtering |
| `--quiet` | `false` \| `true` \| `force` | `false` | `false`: normal. `true`: skip skill confirmations. `force`: skip all interruptions. |
| `--mode` | string | `default` | Scrutinize review lens: `default`, `harsh`, `frontend`, `cleancode`, `quick`, `academic` (or ID/symbol) |
| `--agent-attribution` | bool | `false` | Allow Co-Authored-By lines in commits |
| `--min-comments` | integer | `0` | Minimum pseudo-comments to produce (0-10); higher = more antagonistic |
| `--user-mail` | string | *(config)* | Override git email check; `_` skips |
| `--user-name` | string | *(config)* | Override git name check; `_` skips |

Booleans: `--quiet` = true, `--quiet:true` = true, `--quiet:false` = false, `--quiet:force` = force (max automation).
Omitted params use defaults from `config.json` > `"defaults"`.

**Identity parameters** (`--handle`, `--user-mail`, `--user-name`) fall back to the corresponding top-level config key (`handle`, `user-mail`, `user-name`) rather than the `defaults` object. They are never prompted for.

**Examples:**
- `/statler-waldorf --item-id:20525 --quiet`
- `/statler-waldorf --item-id:20525 --min-comments:5 --quiet:force`
- `/statler-waldorf --item-id:20525 --mode:harsh --min-comments:8`
- `/statler-waldorf --item-id:20525 --mode:frontend`
- `/statler-waldorf --codebase:personal --item-id:main`
- `/statler-waldorf` *(prompts for item-id, rest from config)*

**Pipeline:** Scrutinize > Retro > Evaluate > Formulate > Implement > Sanity Check > Glean > Finalize
