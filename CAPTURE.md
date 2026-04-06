# 📸 CAPTURE

You are a coding assistant helping a software engineer build and maintain an enterprise-grade product. Prioritize accuracy, professionalism, and thoroughness.

{product-text}

## Background Information

### Locations

#### Notes and Analysis

Work in the following directory: `{personal-dir-location}\notes\{year}\{month}\{folder-name}`

#### Codebase

The codebase is in the following directory: `{codebase}`

## Task

Capture the git diff information for the designated branch, starting from `{starting-hash}` (inclusive) through HEAD.

### Resolving the Starting Point

If `{starting-hash}` is `_`, determine the fork point automatically:

```bash
git merge-base main HEAD
```

Use the resulting hash as the starting point.

If `{starting-hash}` is set to a specific commit hash, use that hash as the starting point for the diff.

### Gathering Diff Information

Collect the following information:

1. **Commit log** — All non-merge commits from the starting point to HEAD, by any author:

```bash
git log {start}..HEAD --no-merges --format="%h %an <%ae> %s" --date=short
```

2. **File-level summary** — A stat overview of all changes:

```bash
git diff {start}..HEAD --stat
```

3. **Full diff** — The complete diff of all changes:

```bash
git diff {start}..HEAD
```

### Filtering

- **Ignore DbContext changes.** Exclude any files whose name contains `DbContext` (e.g., `AcademicsDbContext.cs`, `StudentFirstDbContext.cs`). These files contain auto-generated scaffolding that is not meaningful for review.
- **Include migration files, except Designer.cs.** Migration files (under `Migrations/` directories) should be included — they represent intentional schema changes. However, exclude any file named `*.Designer.cs` within migration directories, as these are auto-generated snapshots that are not meaningful for review.
- **Ignore merge commits.** The commit log uses `--no-merges` to exclude merge commits. Commits that enter the branch via a merge from `main` are not part of the branch's own work.
- **Filter by author.** If the value for `{user-name}` is not `_`, then you have the opportunity to use the value of `{user-name}` as a filter.

### Output

Save a markdown file in the dir `{personal-dir-location}\notes\{year}\{month}\{folder-name}`. Name the file `diffs_{timestamp}.md`.

The file should contain the following sections:

```markdown
# Diff Capture

**Branch:** {designated-branch}
**Starting point:** {start-hash}
**Captured:** {timestamp}

## Commits

{commit log, one line per commit}

## File Summary

{stat output}

## Full Diff

```diff
{full diff content, excluding filtered files}
```​
```

Keep the output factual and unedited. Do not summarize, interpret, or omit any diff content (except the filtered exclusions). The SCRUTINIZE and RETRO phases will interpret this information later.
