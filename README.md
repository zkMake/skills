# skills

Personal collection of [agent skills](https://www.skills.sh/) for Claude Code and other agents. Grown organically for day-to-day use; published here to keep them versioned, recoverable, and installable anywhere.

## Install

```bash
npx skills add zkMake/skills
```

Pick the skills you want from the interactive prompt, or install everything.

## Skills

### CLAUDE.md

| Skill                                                        | Description                                                                                    |
| ------------------------------------------------------------ | ---------------------------------------------------------------------------------------------- |
| [bootstrap-claude-md](skills/claude-md/bootstrap-claude-md) | Author a new CLAUDE.md scoped to the current app, package, or directory from scratch.          |
| [update-claude-md](skills/claude-md/update-claude-md)       | Audit and refresh the nearest CLAUDE.md against recent code changes so it stays accurate.      |

### GitHub

| Skill                                        | Description                                                                                             |
| -------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| [create-pr](skills/github/create-pr)         | Create a GitHub PR with a Conventional Commits title and detailed summary body.                          |
| [update-pr](skills/github/update-pr)         | Update the current PR title and summary to reflect recent changes on the branch.                         |
| [gh-cli](skills/github/gh-cli)               | Deep operational knowledge of the GitHub CLI, sourced from the official manual, with reference sheets.   |

### Media

| Skill                                            | Description                                                              |
| ------------------------------------------------ | ------------------------------------------------------------------------ |
| [optimize-audio](skills/media/optimize-audio)   | Shrink audio files by re-encoding with ffmpeg at lower bitrates/channels. |

### Writing

| Skill                                                            | Description                                                                                  |
| ---------------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| [study-writing-style](skills/writing/study-writing-style)       | Study an author's blog and produce a detailed, quote-grounded writing-style guide.           |

## Layout

Each skill is a directory containing a `SKILL.md` (frontmatter: `name`, `description`) plus optional `references/` files, following the [Agent Skills](https://agentskills.io/) format:

```
skills/<category>/<skill-name>/SKILL.md
```

## License

MIT
