# skills

Orientation index for coding tasks on this personal agent-skills collection (published as `zkMake/skills`, installed via `npx skills add zkMake/skills`). The deep content lives in `context/*.md` — load only what's relevant to the task at hand. Extend the "Task-specific plan" section at the bottom for the actual change.

## Must-know (always load this much)

- **Pure markdown repo.** No package.json, build step, or tests; the repo tree is the distribution — skill directories are copied verbatim into consumers' `~/.claude/skills/`.
- **One skill per directory** at `skills/<category>/<skill-name>/SKILL.md`, following the [Agent Skills](https://agentskills.io/) format.
- **`description` frontmatter is the trigger surface.** It's what agents match against to decide when to invoke — the most load-bearing field in the repo. Write it with explicit trigger phrases.
- **Three names must agree**: directory name, frontmatter `name`, and the README table link path. The installer keys off the directory path.
- **`README.md` tables are hand-maintained.** Adding, renaming, or moving a skill requires updating the matching category table.

## Where to read deeper (load on demand)

| Section | File | When to load |
| --- | --- | --- |
| Repo purpose, layout, category map, related files | `context/overview.md` | Onboarding, adding a category, touching README |
| SKILL.md authoring format and conventions | `context/skill-authoring.md` | Creating or editing any skill |

## Task-specific plan

(Extend below for the task at hand. Keep orientation section above unchanged.)

### Add tailwind-to-stylex skill (2026-08-24)

New `styling` category. `skills/styling/tailwind-to-stylex/`: SKILL.md drives a 7-phase migration (Baseline → Tooling → Tokens → Reset → Migrate → Lift → Teardown); `references/` holds one sheet per phase. Design decisions grilled + settled: ground-truth CSS from the project's own compiled Tailwind (never memorized tables), literal-then-lift, screenshot + computed-style diff gates, `light-dark()` tokens, sanctioned plain-CSS escape hatches, Astro hybrid (islands = StyleX, `.astro` = scoped styles on bridge vars, babel+postcss wiring primary). README + overview.md tables updated.
