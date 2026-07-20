# Core: global behavior, auth, `gh api`, config, alias, extension

Contents:
- [Global flags & repo resolution](#global)
- [Environment variables](#env)
- [Exit codes](#exit-codes)
- [`gh auth`](#auth)
- [`gh api` (REST + GraphQL)](#api)
- [`gh config`](#config)
- [`gh alias`](#alias)
- [`gh extension`](#extension)

<a id="global"></a>
## Global behavior & flags

Signature: `gh <command> <subcommand> [flags]`

Universal flags: `--help`, `--version`, and `-R, --repo [HOST/]OWNER/REPO`.

**Repo resolution order:** `--repo`/`-R` flag → `$GH_REPO` → the `git remote` of
the current directory. Pass `-R OWNER/REPO` whenever you operate on a repo other
than the cwd.

<a id="env"></a>
## Environment variables

| Variable | Purpose |
|----------|---------|
| `GH_TOKEN` / `GITHUB_TOKEN` | Auth token for github.com / `*.ghe.com` (`GH_TOKEN` wins) |
| `GH_ENTERPRISE_TOKEN` / `GITHUB_ENTERPRISE_TOKEN` | Token for GHES hosts |
| `GH_HOST` | Default hostname when not inferable |
| `GH_REPO` | Override current repo (`[HOST/]OWNER/REPO`) |
| `GH_EDITOR` | Editor for authoring (then `GIT_EDITOR`, `VISUAL`, `EDITOR`) |
| `GH_PAGER` / `PAGER` | Terminal pager |
| `GH_BROWSER` / `BROWSER` | Browser for `--web` |
| `GH_PROMPT_DISABLED` | Set to disable interactive prompting |
| `GH_FORCE_TTY` | Force terminal-style output (column count / %) |
| `GH_DEBUG` | Truthy = verbose stderr; `api` = HTTP-level detail |
| `GH_CONFIG_DIR` | Override config directory |
| `NO_COLOR` / `CLICOLOR` / `CLICOLOR_FORCE` | ANSI color control |

<a id="exit-codes"></a>
## Exit codes

`0` success · `1` error · `2` cancelled mid-run · `4` auth required.
Some commands add their own (e.g. `gh pr checks` returns `8` while pending).

<a id="auth"></a>
## `gh auth`

| Subcommand | Use |
|-----------|-----|
| `login` | Authenticate to a host (browser OAuth or `--with-token`) |
| `logout` | Remove stored creds locally (does not revoke on GitHub) |
| `status` | Show active account, host, token state |
| `refresh` | Add/remove OAuth scopes on stored creds |
| `token` | Print the stored token |
| `switch` | Change active account for a host |
| `setup-git` | Configure git to use `gh` as credential helper |

```bash
gh auth login                              # interactive browser flow
gh auth login -h enterprise.internal       # enterprise host
gh auth login --with-token < token.txt     # token from file (CI)
gh auth status                             # who am I, what scopes
gh auth status --show-token                # reveal token
gh auth refresh -s project,delete_repo     # grant extra scopes
gh auth token                              # print token (for piping)
gh auth switch --hostname github.com --user monalisa
gh auth setup-git                          # use gh creds for git push/pull
```

Key `login` flags: `-h/--hostname`, `-p/--git-protocol {ssh|https}`,
`-s/--scopes`, `-w/--web`, `--with-token` (stdin), `--insecure-storage`.
Common scopes to refresh into: `project` (Projects v2), `delete_repo`,
`write:org`, `read:public_key`, `workflow`.

<a id="api"></a>
## `gh api` — authenticated REST & GraphQL

`gh api <endpoint> [flags]`. Endpoint is a REST path (`repos/{owner}/{repo}/...`)
or the literal `graphql`. `{owner}`, `{repo}`, `{branch}` auto-fill from the
current repo.

| Flag | Meaning |
|------|---------|
| `-X, --method` | HTTP method (defaults GET; auto-POST when params added) |
| `-f, --raw-field key=value` | Static string param |
| `-F, --field key=value` | Typed param: auto-casts bool/int/null, expands `{placeholder}`, `@file`, `@-` (stdin) |
| `-H, --header key:value` | Request header |
| `-q, --jq <expr>` | Filter response with jq (jq need not be installed) |
| `-t, --template <tmpl>` | Go-template the JSON |
| `--paginate` | Fetch all pages |
| `--slurp` | With `--paginate`, wrap pages in one JSON array |
| `--input <file>` | Request body from file (`-` = stdin); fields become query params |
| `--cache <dur>` | Cache response (e.g. `1h`, `3600s`) |
| `--hostname` | Target host (default github.com) |
| `-i, --include` | Include status line + headers |

Nested params: `key[subkey]=value`, arrays `key[]=a key[]=b`,
file content `files[name][content]=@path`.

GraphQL: endpoint `graphql`; all `-f`/`-F` params except `query` and
`operationName` become GraphQL variables.

```bash
# REST: read a single field
gh api repos/{owner}/{repo} --jq '.default_branch'

# REST: list all releases' tags, paginated
gh api repos/{owner}/{repo}/releases --paginate --jq '.[].tag_name'

# REST: POST a comment
gh api repos/{owner}/{repo}/issues/123/comments -f body='Looks good!'

# REST: PATCH with a typed field
gh api repos/{owner}/{repo} -X PATCH -F has_issues=false

# Request body from stdin
echo '{"title":"x"}' | gh api repos/{owner}/{repo}/issues --input -

# GraphQL with variables
gh api graphql -F owner='{owner}' -F name='{repo}' -f query='
  query($owner:String!,$name:String!){
    repository(owner:$owner,name:$name){ releases(last:3){ nodes{ tagName } } }
  }'

# GraphQL cursor pagination
gh api graphql --paginate -f query='
  query($endCursor:String){
    viewer{ repositories(first:100,after:$endCursor){
      nodes{ nameWithOwner } pageInfo{ hasNextPage endCursor } } } }'
```

<a id="config"></a>
## `gh config`

`get` / `set` / `list` / `clear-cache`. All support `-h/--host` for per-host
settings.

Common keys: `editor`, `git_protocol` (`https`|`ssh`), `pager`, `prompt`
(`enabled`|`disabled`), `browser`, `color_labels`, `spinner`, `telemetry`.

```bash
gh config set editor "code --wait"
gh config set git_protocol ssh --host github.com
gh config set prompt disabled        # never prompt interactively
gh config get git_protocol
gh config list
```

<a id="alias"></a>
## `gh alias`

`set` / `list` / `delete` / `import`. Expansion can use positional placeholders
`$1`, `$2` (else extra args append). Prefix with `!` or pass `--shell` to run
through the shell (enables pipes/redirection). Pass `-` to read expansion from
stdin.

```bash
gh alias set pv 'pr view'                              # gh pv 123
gh alias set epicsBy 'issue list --author="$1" --label="epic"'
gh alias set --shell igrep 'gh issue list --label="$1" | grep "$2"'
gh alias set bugs '!gh issue list --label=bug | head -5'
gh alias list
gh alias delete pv
gh alias import aliases.yml --clobber
```

<a id="extension"></a>
## `gh extension`

(`gh ext`) `install` / `list` / `remove` / `upgrade` / `exec` / `search` /
`browse` / `create`. Extension repos are named `gh-<name>`.

```bash
gh extension install owner/gh-my-ext
gh extension install owner/gh-my-ext --pin v1.2.0
gh extension list
gh extension upgrade --all
gh extension remove my-ext
gh extension search dashboard --limit 10
gh extension exec label --list     # disambiguate when name clashes with core
gh extension create --precompiled=go myext
```
