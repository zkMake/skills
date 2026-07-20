---
name: gh-cli
description: >-
  Deep operational knowledge of the GitHub CLI (`gh` command) sourced from the
  official manual at cli.github.com/manual. Use this skill WHENEVER a task
  involves GitHub via the terminal: opening/merging/reviewing pull requests,
  filing or editing issues, creating releases, triggering or watching GitHub
  Actions workflow runs, downloading run artifacts, managing labels/secrets/
  variables/projects, forking/cloning/creating repos, or hitting the GitHub
  REST/GraphQL API with `gh api`. Trigger even when the user doesn't say "gh"
  explicitly but clearly wants a GitHub operation done from the command line
  (e.g. "open a PR for this branch", "merge #42 and delete the branch", "what
  CI checks are failing", "make a release from this tag", "set a repo secret",
  "rerun the failed jobs"). Covers correct non-interactive flags for scripting,
  `--json`/`--jq`/`--template` output, and the `gh api` escape hatch for
  anything the porcelain commands don't expose.
---

# GitHub CLI (`gh`)

This skill makes you fluent with `gh`: pick the right subcommand, drive it
non-interactively, and reach for `gh api` when a porcelain command can't do the
job. Detailed per-command flag references live in `references/` — load the one
that matches the task instead of guessing flag names.

## Operating principles

**Run non-interactively by default.** Most `gh` subcommands drop into an
interactive prompt when required fields are missing. In an agent session that
prompt hangs. So always supply the flags that make a command complete on its
own: `--title`/`--body` for create commands, a merge strategy (`--squash` etc.)
for `gh pr merge`, `--yes` for destructive commands. If a body is long or
multi-line, write it to a temp file and pass `--body-file <file>` (or `-F -` to
read stdin) rather than fighting shell quoting.

**Prefer `gh` over raw `git` for anything server-side.** Creating PRs, reading
checks, merging, releases, issues — these are GitHub operations, not git
operations. Reserve `git` for local history work (commit, rebase, branch).

**Scripting → `--json` + `--jq`.** Whenever you need to read a value out of `gh`
to use it programmatically, request JSON and filter it. Don't parse the
human-formatted table output. To discover which fields exist, run the command
with `--json` and no field list — it prints the available field names. Example:
`gh pr view 42 --json state,mergeable --jq '.state'`.

**`gh api` is the escape hatch.** The porcelain commands cover the common 90%.
For everything else (arbitrary REST endpoints, GraphQL, settings the CLI doesn't
surface), use `gh api`. `{owner}`/`{repo}`/`{branch}` placeholders auto-fill
from the current repo. See `references/core.md`.

**Targeting a different repo:** every command accepts `-R OWNER/REPO` (or
`--repo`). Without it, `gh` infers the repo from the current directory's git
remote, or from `$GH_REPO`. When the user names a repo that isn't the cwd, pass
`-R`.

**Check auth first if anything 401s.** `gh auth status` shows the active account
and token scopes. Missing scope is the usual cause of permission errors — fix
with `gh auth refresh -s <scope>` (e.g. `project`, `delete_repo`, `write:org`).

## Command map — which reference to load

| Task | Commands | Reference |
|------|----------|-----------|
| Auth, `gh api` (REST/GraphQL), config, aliases, extensions, env vars, exit codes | `gh auth`, `gh api`, `gh config`, `gh alias`, `gh extension` | `references/core.md` |
| Pull requests: create, review, merge, checkout, diff, checks, edit | `gh pr *` | `references/pr.md` |
| Issues, labels, Projects v2, search | `gh issue`, `gh label`, `gh project`, `gh search` | `references/issues.md` |
| GitHub Actions: runs, workflows, cache, secrets, variables | `gh run`, `gh workflow`, `gh cache`, `gh secret`, `gh variable` | `references/actions.md` |
| Repos, releases, gists, browse, status, codespaces, rulesets, attestation | `gh repo`, `gh release`, `gh gist`, `gh browse`, `gh status`, `gh codespace` | `references/repo-release.md` |

If you already know the exact flags for a routine command (`gh pr create
--fill`, `gh pr checks`), just run it. Load the reference when you're unsure of a
flag name, need a less-common subcommand, or want the scripting/JSON details.

## High-frequency workflows

These are the patterns that come up constantly. Each links to deeper coverage.

**Open a PR for the current branch** (fills title/body from commits, no prompt):
```bash
gh pr create --fill
# add reviewers/labels/draft as needed:
gh pr create --fill --reviewer octocat --label enhancement --draft
```

**See why CI is red, then read the failing log:**
```bash
gh pr checks                       # status of all checks on current branch's PR
gh run view --log-failed           # failing step logs for the latest run
gh run watch <run-id> --exit-status # block until done; nonzero exit on failure
```

**Approve and merge, cleaning up the branch:**
```bash
gh pr merge <number> --squash --delete-branch
```

**File an issue non-interactively:**
```bash
gh issue create --title "Login fails on Safari" --body-file repro.md --label bug
```

**Cut a release with auto-generated notes and assets:**
```bash
gh release create v1.2.3 --generate-notes ./dist/*.tar.gz
```

**Trigger a workflow_dispatch with inputs:**
```bash
gh workflow run deploy.yml --ref main -f environment=production
```

**Read a value out of the API for scripting:**
```bash
gh api repos/{owner}/{repo} --jq '.default_branch'
```

## Gotchas worth remembering

- **Confirm before destructive/outward-facing actions.** Merging, closing,
  deleting, force-syncing, and changing repo visibility are hard to undo and
  visible to others. Unless the user already authorized it, confirm first —
  and never add `--yes`/`--admin`/`--force` on your own initiative.
- **`@me`** works as the current user in `--assignee`, `--author`, and search
  filters — handy for "my PRs", "issues assigned to me".
- **Body text with special characters:** prefer `--body-file` / `-F -` over
  inline `--body` to avoid the shell mangling backticks, `$`, quotes, newlines.
- **Job IDs for `gh run rerun --job` are `databaseId`**, not the number in the
  web URL. Get them via `gh run view <id> --json jobs`.
- **`gh pr checks` exit code 8** means checks are still pending (not failed) —
  useful in scripts that distinguish "running" from "broken".
- **Auto-merge vs immediate merge:** `gh pr merge --auto` queues the merge for
  when checks pass; without `--auto` it merges now (and errors if not mergeable).
