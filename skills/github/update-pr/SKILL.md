---
name: update-pr
description: Update the current GitHub PR title and summary to reflect recent changes on the branch. Use when user says "update pr", "update the pr", "refresh pr", "update pr title", "update pr summary", or "sync pr".
disable-model-invocation: true
---

# Update PR

## Workflow

### 1. Detect the current PR

```bash
gh pr view --json number,url,baseRefName
```

If no PR exists for the current branch, inform the user and stop.

### 2. Gather context

Using the base branch from step 1:

```bash
git log --oneline origin/<base>..HEAD
git diff origin/<base>...HEAD --stat
git diff origin/<base>...HEAD
```

### 3. Compose the updated PR title

Format: `type(scope): description`

- **type**: `feat` `fix` `refactor` `chore` `build` `docs` `test` `perf` `ci` `style` `hotfix`
- **scope**: affected package or app (e.g. `platform-typing`, `virtual-room-three`)
- **description**: imperative mood, lowercase, total title under 70 chars

Derive type and scope from the diff and commit log. Use the dominant type when commits span multiple types.

### 4. Compose the updated PR body

<pr-body-template>
## Summary

- <what changed and why -- one bullet per logical group of changes>

## Commits

- `type(scope):` description
- (list each commit from git log)
</pr-body-template>

Omit the **Commits** section if there is only one commit.

Group Summary bullets by change type when there are 4+ mixed-type commits.

### 5. Update the PR

```bash
gh pr edit <number> \
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

Output the URL from step 1.
