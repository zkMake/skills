---
name: bootstrap-claude-md
description: Generate a new CLAUDE.md scoped to the current app, package, or directory by reviewing its code, dependencies, and architecture. Use when user says "create claude.md", "bootstrap claude.md", "write claude.md", "scaffold claude.md", or asks to author a CLAUDE.md for a specific app/package/sub-directory in a monorepo. Use update-claude-md instead when one already exists.
disable-model-invocation: true
---

# Bootstrap CLAUDE.md

Author a new CLAUDE.md that orients a future agent to the current working scope: what it is, how it's wired, what's load-bearing, and the conventions a contributor must respect. Optimized for both human skim and LLM ingestion.

## Output modes

Before drafting, ask the user which mode to use:

- **Flat (default)** — single `CLAUDE.md` containing every section. Best for leaf packages, small apps, anything that fits comfortably in ~150 lines.
- **Progressive disclosure** — slim `CLAUDE.md` orientation index that lists must-know rules + a table of links into `context/*.md` files for everything else. Best for large apps with multiple subsystems, where loading the whole file every turn is wasteful.

Quote the two options to the user and wait for their pick. If they don't choose, default to flat. Skip this prompt only if the user already named the mode in their request (e.g. "bootstrap a progressive claude.md").

## Scope rules

- **Target path**: `<cwd>/CLAUDE.md` unless the user names a different directory. Resolve to an absolute path before writing.
- **If the file already exists**: do NOT silently overwrite. Stop and ask the user how to proceed (see step 1). The user picks: route to `update-claude-md` to refresh in place, or delete the existing file and start fresh from this skill. Only proceed once they've chosen.
- **Never touch** `~/.claude/CLAUDE.md` or sibling packages' CLAUDE.md files.
- **Inherit, don't duplicate**: walk up to the repo root and read every ancestor `CLAUDE.md`. Anything already covered there (global naming rules, lint config, monorepo tooling) should NOT be repeated — only note local deviations.

## Workflow

### 1. Confirm scope

```bash
pwd
git rev-parse --show-toplevel
ls CLAUDE.md 2>/dev/null && echo "EXISTS"
```

State the target path back to the user in one line before doing research.

**If `CLAUDE.md` already exists at the target path**, do not proceed. Surface it and ask the user to pick:

- **Update in place** — hand off to the `update-claude-md` skill, which audits the existing file against current code and refreshes it. This preserves any hand-authored sections.
- **Delete and start fresh** — remove the existing file (`rm <path>`) and bootstrap a brand new one with this skill. Loses any hand-authored content.

Quote the existing path back to the user, name both options, and wait for their explicit choice. Don't assume — even a stale-looking CLAUDE.md may have content the user wants to keep. If they pick "update", stop this skill and invoke `update-claude-md`. If they pick "delete and start fresh", confirm the deletion, then continue with step 2.

### 2. Read ancestor CLAUDE.md files

Walk from `cwd` up to repo root. Collect each `CLAUDE.md`. Note inherited rules so this file stays additive, not redundant.

### 3. Survey the scope

In parallel where possible:

- `package.json` — name, scripts, deps, exports, bin, peer/optional deps, workspace links
- `tsconfig.json` / build config (`vite.config`, `tsup.config`, `turbo.json` task entry, etc.)
- Top-level directory tree (depth 2–3)
- README/docs in the directory
- Entry points named in `package.json` (`main`, `module`, `exports`, `bin`, framework-specific like `app/`, `src/index.ts`, `src/main.tsx`, route trees)
- Public API barrel files (`src/index.ts`) and any `types.ts`, `schema.ts`, `constants.ts`
- Test setup files (vitest/playwright config) — only enough to know the runner

Read enough source to describe each major subsystem in one sentence. Don't read every file.

### 4. Identify the load-bearing decisions

Before drafting, list internally:

- What problem does this scope solve? (one sentence)
- What's the public surface? (exports, routes, CLI commands, components consumed by siblings)
- What architectural choices would surprise a new contributor? (state lib, data flow, codegen, runtime boundary, plugin system, etc.)
- What conventions are enforced here that differ from ancestor CLAUDE.md?
- What's load-bearing but easy to break? (generated files, schema sources of truth, build order)

If you can't answer these from the survey, read more source — don't guess.

### 5. Draft using this skeleton

Keep sections short. Use tables for anything list-shaped (deps, scripts, files) — they parse cleanly for both humans and LLMs. Drop sections that don't apply rather than padding.

#### Flat mode skeleton

```md
# <package or app name>

<One-paragraph purpose: what this scope is, who consumes it, why it exists.>

## Stack

| Concern | Choice | Notes |
| --- | --- | --- |
| Runtime | … | … |
| Framework | … | … |
| State | … | … |
| Build | … | … |
| Test | … | … |

## Entry points

- `<path>` — <what runs / is exported here>
- …

## Directory map

| Path | Responsibility |
| --- | --- |
| `src/…` | … |
| `…` | … |

Only include paths a contributor needs to know. Skip generated dirs unless the contract matters.

## Architecture

<2–6 short bullets or a small diagram in fenced code. Cover data flow, module boundaries, and any non-obvious wiring (codegen, plugin registration, runtime split). Name the files where each decision lives.>

## Key decisions

- **<decision>** — <what + why, one line>. See `<file>`.
- …

Only the choices that would surprise someone or that we'd defend in review. Not every dep.

## Conventions (local)

Only deviations from / additions to ancestor CLAUDE.md. If none, omit the section.

## Commands

| Script | Purpose |
| --- | --- |
| `bun run …` | … |

Pull from `package.json` scripts. Skip ones that just wrap turbo passthroughs unless the local form matters.

## Gotchas

- <non-obvious thing that bites people: build order, generated file, env var, peer dep version pin, etc.>
- …

Omit if there are none worth flagging — don't invent.

## Related

- Parent: `<../CLAUDE.md>` — <what's inherited>
- Siblings consumed: `<…>`
- Siblings consuming this: `<…>`
```

#### Progressive disclosure skeleton

Writes one slim `CLAUDE.md` plus a `context/` folder of focused topic files. The root file is the orientation index — short, always loaded. The `context/*.md` files are deep dives loaded on demand.

**Root `CLAUDE.md`** — keep it tight (target ~30–60 lines). Structure:

```md
# <package or app name>

Orientation index for coding tasks on <one-line purpose>. The deep content lives in `context/*.md` — load only what's relevant to the task at hand. Extend the "Task-specific plan" section at the bottom for the actual change.

## Must-know (always load this much)

- **Stack.** <one line>
- **Run from `<path>`.** <key dev/test/build commands>
- **<load-bearing rule>.** <one line, e.g. schema source of truth, singleton access, grid spacing helper>
- **<load-bearing rule>.** …
- **Conventions.** <commit style, branch prefix, import style — only the non-obvious ones>

Aim for 4–8 bullets. Each one is something every task touches or every contributor breaks. If a rule only matters for one subsystem, push it into the matching `context/*.md`.

## Where to read deeper (load on demand)

| Section | File | When to load |
| --- | --- | --- |
| <topic> | `context/<slug>.md` | <trigger: file touched / task type> |
| … | … | … |

## Task-specific plan

(Extend below for the task at hand. Keep orientation section above unchanged.)
```

**`context/*.md` files** — one per subsystem or concern. Each file is self-contained: a contributor loading just that file plus the root should be able to work on that subsystem. Suggested set (drop any that don't apply, add others as needed):

| File | Contents |
| --- | --- |
| `context/overview.md` | What the app is, full conventions list, all key deps, shared-pkg API surface, anything onboarding-shaped |
| `context/directory-map.md` | Annotated `src/` tree — the "Directory map" table from flat mode, expanded |
| `context/lifecycle.md` | Runtime lifecycle / bootstrap order / dispose paths (if non-trivial) |
| `context/architecture.md` | Data flow, module boundaries, plugin or codegen wiring — the "Architecture" + "Key decisions" sections from flat mode |
| `context/<subsystem>.md` | One file per load-bearing subsystem (audio, state machine, schema, build pipeline, …). Each names the files it covers |
| `context/conventions.md` | Full convention checklist for non-trivial changes (deviations from + additions to ancestor CLAUDE.md) |
| `context/events.md` (etc.) | Reference tables that are too long for the root but still need to be findable |

Each `context/*.md` starts with a one-line summary of what's inside and ends with a short "Related" pointer to sibling context files if the topic spans more than one.

**Choosing what goes where.** The split test: would a contributor need this rule on _every_ task, regardless of which subsystem they're in? If yes, it belongs in the root "Must-know" bullets. If it only matters when touching a specific subsystem, it belongs in that subsystem's `context/*.md`. Don't duplicate — link instead.

**Related** pointer (parent CLAUDE.md, siblings) goes in `context/overview.md`, not the root.

### 6. Quality bar before writing

- Every claim must be checkable against a file or `package.json` field. No vibes.
- No time-sensitive language ("recently", "as of April 2026", "new"). Write current truth.
- No meta-commentary ("generated by skill", "TODO: fill in").
- Don't restate ancestor CLAUDE.md content. Reference it under **Related** instead.
- Don't add a "Task-specific plan" section — that's the user's space, leave it absent so they can add it when needed.
- Keep total length proportional to scope. A leaf utility package may be ~40 lines; a top-level app may be ~150. If you're padding past that, you're adding noise.

### 7. Write and report

Use `Write` to create the file(s). For progressive mode, create `context/` first (`mkdir -p <scope>/context`), then write the root `CLAUDE.md` and each `context/*.md`. Then end with:

- **Created**: `<path>` (<line count> lines) — list every file written
- **Inherited from**: list of ancestor CLAUDE.md files surveyed
- **Open questions**: anything you couldn't verify and chose to omit (so user can fill in)
- **Next**: remind the user that `update-claude-md` keeps it fresh as the code changes

Do not commit.
