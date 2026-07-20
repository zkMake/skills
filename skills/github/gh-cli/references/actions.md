# GitHub Actions & CI — runs, workflows, cache, secrets, variables

All commands accept `-R, --repo OWNER/REPO` and (where listed) `--json`/`-q,
--jq`/`-t, --template`.

Contents: [run](#run) · [workflow](#workflow) · [cache](#cache) ·
[secret](#secret) · [variable](#variable)

<a id="run"></a>
## `gh run` — workflow runs

| Subcommand | Key flags |
|-----------|-----------|
| `list`/`ls` | `-w/--workflow`, `-b/--branch`, `-s/--status`, `-e/--event`, `-u/--user`, `-c/--commit`, `--created`, `-L/--limit` (def 20), `-a/--all`, `--json` |
| `view [id]` | `-j/--job <id>`, `--log`, `--log-failed`, `-v/--verbose`, `-w/--web`, `--exit-status`, `-a/--attempt`, `--json` |
| `watch <id>` | `--exit-status`, `-i/--interval` (def 3s), `--compact` |
| `rerun [id]` | `--failed`, `-j/--job <databaseId>`, `-d/--debug` |
| `cancel [id]` | `--force` |
| `download [id]` | `-n/--name` (repeatable), `-p/--pattern` (glob, repeatable), `-D/--dir` |
| `delete [id]` | — |

`--log-failed` is the fast path to "why did CI fail". For `rerun --job`, the ID
is the job's `databaseId`, not the web-URL number — fetch it from
`gh run view <id> --json jobs`.

```bash
gh run list --workflow ci.yml --branch main --status failure
gh run view --log-failed                       # failing step logs, latest run
gh run watch 9876543210 --exit-status --compact # block; nonzero on failure
gh run rerun 9876543210 --failed --debug
gh run download 9876543210 --pattern "coverage-*" --dir ./reports
gh run view 9876543210 --json jobs \
  --jq '.jobs[] | select(.conclusion=="failure") | {name, databaseId}'
```

<a id="workflow"></a>
## `gh workflow`

`list`/`ls` (`-a/--all`, `-L/--limit` def 50, `--json`),
`view [id|name|file]` (`-r/--ref`, `-y/--yaml`, `-w/--web`),
`run [id|name]`, `enable`, `disable`.

`gh workflow run` triggers a `workflow_dispatch` (file must declare
`on: workflow_dispatch`). Pass inputs with `-f key=value` (string),
`-F key=@file` (file/`@-` stdin), or `--json` to read the whole input object
from stdin. `-r/--ref` selects the branch/tag (default: default branch).

```bash
gh workflow run deploy.yml --ref main -f environment=production -f dry_run=false
echo '{"environment":"production","dry_run":"false"}' \
  | gh workflow run deploy.yml --ref main --json
gh workflow disable nightly.yml
```

<a id="cache"></a>
## `gh cache`

`list`/`ls` (`-k/--key`, `-r/--ref`, `-L/--limit` def 30, `-S/--sort
{created_at|last_accessed_at|size_in_bytes}`, `-O/--order`, `--json`),
`delete [id|key|--all]` (`-a/--all`, `-r/--ref`, `--succeed-on-no-caches`).

```bash
gh cache list --sort size_in_bytes --order desc --limit 5
gh cache delete --all --ref refs/pull/42/merge --succeed-on-no-caches
```

<a id="secret"></a>
## `gh secret`

Values are encrypted locally before upload. Scopes: repo (default),
environment (`-e/--env`), org (`-o/--org`), user/Codespaces (`-u/--user`). App
context with `-a/--app {actions|codespaces|dependabot|agents}`.

`list`/`ls` (`--json`), `set <name>`, `delete <name>`.

`gh secret set` reads the value from stdin if `-b/--body` is omitted. Org
secrets: `-v/--visibility {all|private|selected}` + `-r/--repos`. Bulk-load a
dotenv file with `-f/--env-file`.

```bash
gh secret set DEPLOY_KEY < ~/.ssh/id_deploy           # value from file/stdin
gh secret set TOKEN -b "$TOKEN"                        # inline
gh secret set NPM_TOKEN --org my-org --app dependabot \
  --visibility selected --repos api-service,frontend
gh secret set -f .env --env production                 # bulk from dotenv
gh secret list --json name,updatedAt
```

<a id="variable"></a>
## `gh variable`

Plaintext (unlike secrets). Same scoping: repo (default), `-e/--env`,
`-o/--org`. `list`/`ls`, `set <name>`, `get <name>`, `delete <name>`.

```bash
gh variable set NODE_ENV -b production
gh variable set -f .env --env staging
gh variable get NODE_ENV --json value --jq '.value'
gh variable list --json name,value
```
