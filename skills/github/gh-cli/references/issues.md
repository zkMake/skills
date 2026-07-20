# Issues, labels, projects, search

All commands accept `-R, --repo OWNER/REPO`. List/view commands support
`--json <fields>` / `-q, --jq` / `-t, --template` (run `--json` with no field
list to discover fields).

Contents: [issue](#issue) · [label](#label) · [project](#project) · [search](#search)

<a id="issue"></a>
## `gh issue`

Subcommands: `create`, `list`/`ls`, `status`, `view`, `edit`, `close`,
`reopen`, `comment`, `delete`, `transfer`, `pin`/`unpin`, `develop`,
`lock`/`unlock`. Most take `<number>` or `<url>`.

### `gh issue create` (`gh issue new`)
`-t/--title`, `-b/--body`, `-F/--body-file` (`-`=stdin), `-a/--assignee`
(`@me`), `-l/--label`, `-m/--milestone`, `-p/--project`, `-T/--template`,
`-e/--editor`, `-w/--web`. Non-interactive: provide title + body.

```bash
gh issue create -t "Login fails on Safari" -F repro.md -l bug -a @me
gh issue create -t "Add dark mode" -l enhancement -p "Q3 Roadmap"
```

### `gh issue list` (`gh issue ls`)
`-s/--state {open|closed|all}`, `-a/--assignee`, `-A/--author`, `-l/--label`,
`-m/--milestone`, `-S/--search`, `--mention`, `--app`, `-L/--limit` (def 30),
`--json/--jq/--template`, `-w/--web`.

```bash
gh issue list --state closed --label bug --limit 50
gh issue list --assignee @me --milestone v1.2
gh issue list --search "no:assignee is:open" --json number,title,url
```

### `gh issue view`
`-c/--comments`, `-w/--web`, `--json` (fields: `assignees`, `author`, `body`,
`labels`, `milestone`, `number`, `state`, `title`, `url`, …).

### `gh issue edit` (accepts multiple numbers for bulk edit)
`-t/--title`, `-b/--body`, `-F/--body-file`,
`--add-assignee`/`--remove-assignee` (`@me`,`@copilot`),
`--add-label`/`--remove-label`, `-m/--milestone`/`--remove-milestone`,
`--add-project`/`--remove-project`.

```bash
gh issue edit 42 --add-label priority:high --remove-label bug
gh issue edit 10 20 30 --add-label triage    # bulk
```

### `gh issue close`
`-r/--reason {completed|"not planned"|duplicate}`, `-c/--comment`,
`--duplicate-of <number|url>`.

### `gh issue comment`
`-b/--body`, `-F/--body-file`, `-e/--editor`, `-w/--web`, `--edit-last`
(+`--create-if-none`), `--delete-last` (+`--yes`).

### `gh issue develop` — create/list a linked branch
`-n/--name`, `-b/--base`, `--branch-repo`, `-c/--checkout`, `-l/--list`. The
linked branch becomes the base for `gh pr create`.

```bash
gh issue develop 123 --name feat/new-login --checkout
gh issue develop --list 123
```

### `gh issue lock` / `unlock`
`lock` takes `-r/--reason {off_topic|resolved|spam|too_heated}`.

<a id="label"></a>
## `gh label`

`list`/`ls`, `create`, `edit`, `delete`, `clone`.

```bash
gh label list --json name,color,description
gh label create bug --description "Something isn't working" --color E99695
gh label create needs-review --color 0075ca --force   # update if exists
gh label edit bug --name defect --color B60205
gh label delete wontfix --yes
gh label clone cli/cli                # copy labels from another repo into cwd
gh label clone cli/cli --force        # overwrite existing
```

Flags: `create`/`edit` take `-c/--color` (6-char hex), `-d/--description`,
`-f/--force` (create) / `-n/--name` (edit). `list` takes `-S/--search`,
`-L/--limit`, `--sort {created|name}`, `--order {asc|desc}`.

<a id="project"></a>
## `gh project` (Projects v2)

Requires the `project` scope: `gh auth refresh -s project`. Most subcommands
take the project **number** (from the project URL) plus `--owner <login>`
(`@me` for yourself). JSON output via `--format json` (not `--json`).

Subcommands: `list`/`ls`, `create`, `view`, `edit`, `close`, `copy`, `delete`,
`link`/`unlink`, `mark-template`; items: `item-list`, `item-create` (draft),
`item-add` (existing issue/PR by `--url`), `item-edit`, `item-delete`,
`item-archive`; fields: `field-list`, `field-create`, `field-delete`.

```bash
gh project list --owner "@me"
gh project create --owner monalisa --title "Q4 Roadmap"
gh project item-list 1 --owner "@me" --query "is:open label:bug -status:Done"
gh project item-create 1 --owner "@me" --title "Spike: auth refactor"
gh project item-add 1 --owner monalisa --url https://github.com/org/repo/issues/42
gh project field-create 1 --owner "@me" --name Priority \
  --data-type SINGLE_SELECT --single-select-options "P0,P1,P2"
gh project link 1 --owner myorg --repo myorg/api
```

`item-edit` updates one field per call for real items: `--id`, `--project-id`,
`--field-id`, then one of `--text`/`--number`/`--date`/
`--single-select-option-id`/`--iteration-id` (or `--clear`). For draft items use
`--title`/`--body`.

<a id="search"></a>
## `gh search`

`issues`, `prs`, `repos`, `commits`, `code`. Freeform `[<query>]` uses GitHub
search syntax; flags AND with it. On Unix, separate a query containing leading
hyphens with `--`: `gh search issues -- "foo -label:wontfix"`. All support
`-L/--limit` (def 30), `--sort`, `--order`, `--json/--jq/--template`, `-w/--web`.

```bash
# Issues
gh search issues --assignee @me --state open --label bug
gh search issues "memory leak" --repo cli/cli --match title,body
gh search issues --no-assignee --label "good first issue" --owner myorg

# PRs (issues flags + --base/--head/--draft/--merged/--review/--checks/--reviewed-by)
gh search prs --review approved --assignee @me --repo myorg/api
gh search prs --base main --draft --owner myorg

# Repos (--language/--topic/--stars/--forks/--license/--visibility/--match)
gh search repos "kubernetes ingress" --topic cncf --stars ">100"
gh search repos --owner myorg --language go --sort stars

# Commits / code
gh search commits --author monalisa --hash abc123
gh search code "useState" --owner myorg --language typescript --extension tsx
```
