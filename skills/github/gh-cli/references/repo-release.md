# Repos, releases, gists, browse, and misc

All commands accept `-R, --repo OWNER/REPO`; list/view commands support
`--json`/`-q, --jq`/`-t, --template`.

Contents: [repo](#repo) · [release](#release) · [gist](#gist) · [browse](#browse) ·
[status](#status) · [codespace](#codespace) · [ruleset](#ruleset) · [attestation](#attestation)

<a id="repo"></a>
## `gh repo`

### `create [name]` (`gh repo new`)
Interactive with no args; non-interactive needs a name + visibility flag.
`--public`/`--private`/`--internal`, `-c/--clone`, `-s/--source <dir>`,
`-r/--remote`, `--push`, `-d/--description`, `-h/--homepage`,
`-p/--template OWNER/REPO`, `--include-all-branches`, `-l/--license`,
`-g/--gitignore`, `--add-readme`, `--disable-issues`, `--disable-wiki`,
`-t/--team`.

```bash
gh repo create my-project --public --clone
gh repo create my-org/svc --private --source=. --remote=origin --push
gh repo create my-project --public --template cli/cli
```

### `clone <repo> [dir] [-- <gitflags>]`
Omitting `OWNER/` defaults to your account. For forks, auto-adds an `upstream`
remote (`--no-upstream` to skip). Pass extra git flags after `--`.
```bash
gh repo clone cli/cli
gh repo clone cli/cli -- --depth=1
```

### `fork [repo] [-- <gitflags>]`
No arg forks the current repo. `--clone`, `--remote`, `--remote-name`,
`--org`, `--fork-name`, `--default-branch-only`.
```bash
gh repo fork cli/cli --clone --org my-org --fork-name cli-mirror
```

### `view [repo]`
`-b/--branch`, `-w/--web`, `--json name,description,url,...`.

### `list [owner]` (`gh repo ls`)
`--archived`/`--no-archived`, `--fork`/`--source`, `-l/--language`, `--topic`,
`--visibility {public|private|internal}`, `-L/--limit` (def 30), `--json`.

### `edit [repo]`
`-d/--description`, `-h/--homepage`, `--visibility` (needs
`--accept-visibility-change-consequences`), `--default-branch`,
`--enable-issues`/`--enable-wiki`/`--enable-projects`/`--enable-discussions`,
`--enable-merge-commit`/`--enable-squash-merge`/`--enable-rebase-merge`,
`--enable-auto-merge`, `--delete-branch-on-merge`, `--allow-forking`,
`--allow-update-branch`, `--add-topic`/`--remove-topic`, `--template`,
`--enable-secret-scanning`, `--enable-advanced-security`. Disable a bool with
`--<flag>=false`.
```bash
gh repo edit --enable-squash-merge --enable-merge-commit=false --delete-branch-on-merge
gh repo edit --add-topic cli --add-topic automation
```

### `delete [repo]`
Needs `delete_repo` scope (`gh auth refresh -s delete_repo`). `--yes` only
applies when the repo is given explicitly; otherwise it always prompts.

### Other
`rename [new-name]` (`-y/--yes`), `archive`/`unarchive` (`-y`),
`set-default [repo]` (`-v/--view`, `-u/--unset` — sets which remote `gh` uses),
`sync [dest]` (`-b/--branch`, `-s/--source`, `--force`),
`deploy-key list|add <keyfile>|delete` (`add`: `-t/--title`, `-w/--allow-write`),
`license list|view <id>`, `gitignore list|view <name>`.

```bash
gh repo set-default owner/repo
gh repo sync                       # fast-forward fork from parent
gh repo deploy-key add ~/.ssh/deploy.pub --title "CI" --allow-write
```

<a id="release"></a>
## `gh release`

### `create [tag] [files...]`
Append `#label` to an asset path for a display label. Assets accept globs.
`-t/--title`, `-n/--notes`, `-F/--notes-file` (`-`=stdin), `--generate-notes`,
`--notes-start-tag`, `--notes-from-tag`, `-d/--draft`, `-p/--prerelease`,
`--latest`, `--target <branch|SHA>`, `--verify-tag`, `--fail-on-no-commits`,
`--discussion-category`.

```bash
gh release create v1.2.3 --generate-notes ./dist/*.tar.gz
gh release create v1.2.3 --generate-notes --notes-start-tag v1.2.0
gh release create v2.0.0-rc1 --prerelease --target release/v2
gh release create v1.2.3 './dist/app-linux.tar.gz#Linux binary'
```

### Other
`list`/`ls` (`--exclude-drafts`, `--exclude-pre-releases`, `-L/--limit`,
`-O/--order`, `--json`),
`view [tag]` (no tag = latest; `-w/--web`, `--json assets,tagName,url,...`),
`edit <tag>` (same flags as create + `--tag` to rename; publish a draft with
`--draft=false`),
`download [tag]` (`-p/--pattern` glob, `-A/--archive {zip|tar.gz}`, `-D/--dir`,
`-O/--output` (`-`=stdout), `--skip-existing`, `--clobber`),
`upload <tag> <files...>` (`--clobber`),
`delete <tag>` (`--cleanup-tag`, `-y/--yes`),
`delete-asset <tag> <name>` (`-y`).

```bash
gh release download v1.2.3 -p '*.deb' -p '*.rpm' --dir /tmp/rel
gh release edit v1.0.0 --draft=false --latest
gh release upload v1.2.3 checksums.txt --clobber
```

<a id="gist"></a>
## `gh gist`

`create`, `list`/`ls`, `view`, `edit`, `clone`, `delete`, `rename`.
Default visibility is secret; `-p/--public` to publish.

```bash
gh gist create hello.py --public --desc "Hello in Python"
cat notes.txt | gh gist create -f notes.txt    # from stdin, named
gh gist list --secret --limit 50
gh gist view <id> --filename hello.py --raw
gh gist edit <id> --add new.py
gh gist clone <id> -- --depth=1
gh gist delete <id> --yes
```

<a id="browse"></a>
## `gh browse [number|path|sha]`

Opens the repo (or a resource) in the browser. A number → issue/PR; a `path`
→ that file in the tree; `path:line` → that line.
`-b/--branch`, `-c/--commit`, `-n/--no-browser` (print URL), `--blame`,
`-s/--settings`, `-w/--wiki`, `-p/--projects`, `-r/--releases`, `-a/--actions`.

```bash
gh browse 217                          # issue/PR #217
gh browse main.go:312 --blame
gh browse --commit 77507cd --no-browser   # print URL only
gh browse --settings
```

<a id="status"></a>
## `gh status`

Aggregated activity: assigned issues/PRs, review requests, mentions, repo
activity. `-o/--org <name>`, `-e/--exclude owner/name` (repeatable).

<a id="codespace"></a>
## `gh codespace` (`gh cs`)

`list`/`ls`, `create` (`-R repo`, `-b branch`, `-m machine`,
`-d display-name`), `code` (open in VS Code), `ssh` (`-c name`, `--config`),
`delete` (`-c`, `--all`, `--days N`, `-f`), `ports`, `stop`, `rebuild`,
`logs`, `edit`, `view`, `cp`, `jupyter`.

```bash
gh cs create -R cli/cli -b main -m basicLinux32gb
gh cs ssh -c my-codespace
gh cs delete --all --days 7
```

<a id="ruleset"></a>
## `gh ruleset` (`gh rs`)

Read-only (create/edit on github.com). `list`, `view <id>` (`-w/--web`),
`check [branch]` (which rules apply to a branch name).

<a id="attestation"></a>
## `gh attestation` (`gh at`)

Verify/download artifact attestations (SLSA provenance). `verify`, `download`,
`trusted-root`.
```bash
gh attestation verify dist/app.bin --owner my-org
gh attestation verify oci://ghcr.io/owner/image:tag --owner my-org --format json
```
