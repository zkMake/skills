# Skill authoring

How SKILL.md files in this repo are structured; load before creating or editing any skill.

## Frontmatter

| Field | Rule |
| --- | --- |
| `name` | Required. Must match the directory name exactly. |
| `description` | Required. The trigger surface: what agents match against to decide when to invoke. Lead with what the skill does, then explicit trigger phrases ("Use when user says 'create a PR', 'open a pull request', …"). |
| `disable-model-invocation` | Set `true` for skills meant only for explicit `/slash` invocation. Most skills here set it; `optimize-audio` and `gh-cli` are model-invocable and omit it. |

## Body

- Imperative instructions addressed to the executing agent, typically a numbered workflow with concrete shell commands and skeletons.
- Existing skills run 76–217 lines; keep new ones in that range and self-contained.
- Push long reference material into a `references/` subdirectory and tell the agent in the body when to load each file (see `skills/github/gh-cli/SKILL.md` for the pattern).

## Adding a skill — checklist

1. Create `skills/<category>/<skill-name>/SKILL.md` (new category = new directory).
2. Frontmatter `name` matches directory name.
3. `description` includes trigger phrases; add `disable-model-invocation: true` unless the model should auto-trigger it.
4. Add a row to the matching category table in `README.md` with a working relative link (new category = new table + heading).

## Related

- `context/overview.md` for the category map and repo layout.
