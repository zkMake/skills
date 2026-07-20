# Overview

What this repo is, how it's laid out, and where everything lives.

## Purpose

Personal collection of agent skills for Claude Code and other agents. Grown organically for day-to-day use; published to keep them versioned, recoverable, and installable anywhere. Consumers install with `npx skills add zkMake/skills` and pick skills from an interactive prompt.

## Directory map

| Path | Responsibility |
| --- | --- |
| `skills/<category>/<skill-name>/` | One skill per directory, grouped by category |
| `skills/<category>/<skill-name>/SKILL.md` | The skill itself: YAML frontmatter + instructions |
| `skills/github/gh-cli/references/` | Per-topic reference sheets the skill loads on demand (`pr.md`, `issues.md`, `actions.md`, `repo-release.md`, `core.md`) |
| `README.md` | Install instructions + per-category tables listing every skill |
| `LICENSE` | MIT |

## Categories

| Category | Skills |
| --- | --- |
| `claude-md` | `bootstrap-claude-md`, `update-claude-md` |
| `github` | `create-pr`, `update-pr`, `gh-cli` |
| `media` | `optimize-audio` |
| `writing` | `study-writing-style` |

## Gotchas

- `skills/github/gh-cli/evals/` exists but is empty — placeholder, nothing consumes it yet.
- README tables and link paths are maintained by hand; keep them in sync with the directory tree.

## Related

- `~/.claude/CLAUDE.md` — inherited user conventions (Conventional Commits, `zubin/` branch prefix, gh CLI preference); not repeated here.
- Sibling context: `context/skill-authoring.md` for the SKILL.md format.
