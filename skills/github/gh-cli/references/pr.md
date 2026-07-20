# Pull requests — `gh pr`

All subcommands accept `-R, --repo OWNER/REPO`. Commands that take a PR target
(`view`, `checkout`, `merge`, `review`, `diff`, `checks`, `edit`, `close`,
`reopen`, `comment`, `ready`, `update-branch`) accept a `<number>`, `<url>`, or
`<branch>`; with no argument they default to the PR for the **current branch**.

Contents: [output flags](#json) · [create](#create) · [list](#list) ·
[status](#status) · [view](#view) · [checkout](#checkout) · [diff](#diff) ·
[checks](#checks) · [review](#review) · [merge](#merge) · [ready/close/reopen](#state) ·
[comment](#comment) · [edit](#edit) · [lock/unlock](#lock) · [update-branch](#update-branch) ·
[examples](#examples)

<a id="json"></a>
## Output flags (shared by list/view/status/checks)

- `--json <fields>` — comma-separated fields; run with `--json` and **no list**
  to print available field names.
- `-q, --jq <expr>` — filter the JSON (requires `--json`).
- `-t, --template <go-template>` — format JSON; helpers include `tablerow`,
  `tablerender`, `timeago`, `truncate`.

<a id="create"></a>
## `gh pr create`

Interactive unless enough flags given. Auto-creates a fork if you lack push
access.

| Flag | Meaning |
|------|---------|
| `-t, --title` / `-b, --body` / `-F, --body-file` | Title / body / body from file (`-`=stdin) |
| `-B, --base` / `-H, --head` | Target branch / source branch |
| `-d, --draft` | Create as draft |
| `-r, --reviewer` | Request reviewer (user or `org/team`); repeatable |
| `-a, --assignee` | Assign (`@me` allowed); repeatable |
| `-l, --label` / `-m, --milestone` / `-p, --project` | Add label / milestone / project |
| `-f, --fill` | Title+body from commits |
| `--fill-first` | Title+body from the first commit only |
| `--fill-verbose` | Body from full commit messages |
| `-T, --template` | Start body from a PR template file |
| `--no-maintainer-edit` | Disallow maintainer edits |
| `-w, --web` | Finish in browser |
| `--dry-run` | Print what would be created |

Non-interactive: supply `--title`+`--body` or `--fill`.

<a id="list"></a>
## `gh pr list` (`gh pr ls`)

Defaults: open PRs, limit 30.

`-s/--state {open|closed|merged|all}`, `-B/--base`, `-H/--head`, `-l/--label`,
`-A/--author`, `-a/--assignee`, `-d/--draft`, `-S/--search <query>`,
`-L/--limit`, `--json/--jq/--template`, `-w/--web`.

<a id="status"></a>
## `gh pr status`

PRs relevant to you (authored, assigned, review-requested, current branch).
`-c/--conflict-status` adds merge-conflict info. Supports `--json`.

<a id="view"></a>
## `gh pr view`

`-c/--comments`, `-w/--web`, `--json/--jq/--template`.

<a id="checkout"></a>
## `gh pr checkout` (`gh pr co`)

Check out a PR locally. No arg → pick from 10 most recent.
`-b/--branch <name>`, `--detach`, `-f/--force` (reset local to PR state),
`--recurse-submodules`.

<a id="diff"></a>
## `gh pr diff`

`--color {auto|always|never}`, `-e/--exclude <glob>`, `--name-only`,
`--patch`, `-w/--web`.

<a id="checks"></a>
## `gh pr checks`

`--watch` (refresh until done), `--fail-fast` (exit on first failure),
`-i/--interval <sec>` (default 10), `--required`, `-w/--web`, `--json`
(adds a `bucket` field: pass/fail/pending/skipping/cancel).
**Exit code 8 = checks still pending.**

<a id="review"></a>
## `gh pr review`

Exactly one verdict required (non-interactive): `-a/--approve`,
`-r/--request-changes`, or `-c/--comment`. Body via `-b/--body` or
`-F/--body-file`.

<a id="merge"></a>
## `gh pr merge`

Pick a strategy or you'll be prompted: `-m/--merge`, `-s/--squash`,
`-r/--rebase`.

| Flag | Meaning |
|------|---------|
| `--auto` / `--disable-auto` | Enable/disable auto-merge (merges when requirements pass) |
| `-d, --delete-branch` | Delete local + remote branch after merge |
| `--admin` | Bypass branch protections / merge queue (privileged) |
| `-b, --body` / `-F, --body-file` / `-t, --subject` | Merge commit message |
| `--match-head-commit <SHA>` | Abort unless PR head equals this SHA (safety) |

With a merge queue: passing checks → joins queue; pending → enables auto-merge;
`--admin` bypasses the queue.

<a id="state"></a>
## `gh pr ready` / `close` / `reopen`

- `gh pr ready` — mark draft as ready; `--undo` converts back to draft.
- `gh pr close` — `-c/--comment`, `-d/--delete-branch`.
- `gh pr reopen` — `-c/--comment`.

<a id="comment"></a>
## `gh pr comment`

Interactive by default. `-b/--body`, `-F/--body-file`, `-e/--editor`,
`-w/--web`, `--edit-last` (+`--create-if-none`), `--delete-last` (+`--yes`).

<a id="edit"></a>
## `gh pr edit`

Non-interactive; only specified fields change.
`-t/--title`, `-b/--body`, `-F/--body-file`, `-B/--base`,
`--add-assignee`/`--remove-assignee` (`@me`, `@copilot`),
`--add-reviewer`/`--remove-reviewer`, `--add-label`/`--remove-label`,
`-m/--milestone`/`--remove-milestone`, `--add-project`/`--remove-project`.

<a id="lock"></a>
## `gh pr lock` / `gh pr unlock`

`lock` takes `-r/--reason {off_topic|resolved|spam|too_heated}`.

<a id="update-branch"></a>
## `gh pr update-branch`

Sync PR branch with its base (merge commit by default; `--rebase` to rebase).

<a id="examples"></a>
## End-to-end examples

```bash
# Create from commits, request reviewer, label, draft
gh pr create --fill --reviewer octocat --label enhancement --draft

# Approve then squash-merge and clean up
gh pr review 42 --approve --body "LGTM"
gh pr merge 42 --squash --delete-branch

# Enable auto-merge on green
gh pr merge --auto --squash --delete-branch

# List PRs awaiting my review, number + title
gh pr list --search "review-requested:@me" --state open \
  --json number,title --jq '.[] | "\(.number)\t\(.title)"'

# Safe merge: verify head hasn't moved
SHA=$(gh pr view 99 --json headRefOid --jq '.headRefOid')
gh pr merge 99 --merge --match-head-commit "$SHA" --delete-branch

# Watch checks, fail fast (for CI scripts)
gh pr checks --watch --fail-fast
```
