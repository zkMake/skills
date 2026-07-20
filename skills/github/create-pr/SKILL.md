---
name: create-pr
description: Create a GitHub pull request with a Conventional Commits title and detailed summary body. Use when user says "create a PR", "open a pull request", "submit PR", or "make a PR".
disable-model-invocation: true
---

# Create PR

## Workflow

### 1. Gather context

Run these commands to understand the branch:

```bash
git branch --show-current
git log --oneline origin/master..HEAD
git diff origin/master...HEAD --stat
git diff origin/master...HEAD
```

### 2. Push if needed

```bash
git status -sb
git rev-list --count @{u}..HEAD 2>/dev/null
```

If no upstream or unpushed commits exist, push:

```bash
git push -u origin HEAD
```

Do NOT force-push or amend commits.

### 3. Compose the PR title

Format: `type(scope): description`

- **type**: `feat` `fix` `refactor` `chore` `build` `docs` `test` `perf` `ci` `style` `hotfix`
- **scope**: affected package or app (e.g. `platform-typing`, `virtual-room-three`)
- **description**: imperative mood, lowercase, total title under 70 chars

Derive type and scope from the diff and commit log. Use the dominant type when commits span multiple types.

### 4. Compose the PR body

<pr-body-template>
## Summary

- <what changed and why — one bullet per logical group of changes>

## Commits

- `type(scope):` description
- (list each commit from git log)
</pr-body-template>

Omit the **Commits** section if there is only one commit.

Group Summary bullets by change type when there are 4+ mixed-type commits.

### 5. Create the PR

Always assign `zkmake` and apply the existing `WIP` label.

```bash
gh pr create \
  --base master \
  --assignee zkmake \
  --label WIP \
  --title "type(scope): description" \
  --body "$(cat <<'EOF'
## Summary

- bullet 1
- bullet 2

## Commits

- `type(scope):` description
EOF
)"
```

### 6. Return the PR URL

Output the URL returned by `gh pr create`.
