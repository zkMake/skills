---
name: study-writing-style
description: Studies a specific author's blog and produces a detailed writing-style guide markdown file at writing-styles/<author-slug>.md, capturing their voice, article skeleton, sentence-level moves, code presentation, emoji habits, anti-patterns, and a worked before/after rewrite, with verbatim quotes throughout. Use whenever the user wants to study, analyze, capture, mimic, or build a style guide for a named writer or blogger (e.g. "study TKDodo's writing style", "create a style guide for Julia Evans", "I want to rewrite my articles to sound like Dan Luu", "analyze how Maggie Appleton writes"), especially when they provide an author name plus a link to that author's blog index or archive. Also trigger when a user mentions wanting to match someone's voice for their own posts and references a blog URL. The skill produces a reusable reference document, not a rewritten article.
disable-model-invocation: true
---

# Study Writing Style

This skill turns a writer's blog into a detailed, example-rich style guide that another agent (or the user) can use to rewrite articles in that writer's voice. The output is a single markdown file at `writing-styles/<author-slug>.md`, dense with verbatim quotes from the source.

## When to use this skill

Trigger this skill when the user asks to study, analyze, or capture how a specific author writes, and is pointing at a real corpus (a blog index, an archive page, a list of posts). The goal is always to produce a reference document that someone could hand to a model and say "rewrite my draft in this voice". This skill does NOT rewrite the user's own articles; that's a separate step done with the output of this skill in context.

If the user only gives you an author's name with no link, ask for the blog/archive URL before starting. You cannot do this well from training data alone — the whole point is to ground every observation in verbatim text the author wrote.

## What you need from the user

Before starting, confirm you have:

1. **Author name** — exactly how they want it spelled in the guide.
2. **Blog index URL** — the page that lists their posts. Archive page, "/blog/all", "/posts", or similar.
3. **Output location** (optional) — defaults to `writing-styles/<author-slug>.md` in the current working directory's project root. `<author-slug>` is the author's handle or kebab-cased name (`tkdodo`, `julia-evans`, `dan-luu`).
4. **Project writing rules** (auto-detected) — read the nearest `CLAUDE.md` (project, then user-global) for any writing-style constraints (no em-dashes, no emojis, sentence length caps). You will call these out in the guide as substitution rules when the studied author violates them.

## The process

### 1. Fetch the index, then choose 6–10 varied articles

Use `WebFetch` on the index URL to get the list of posts. Pick a varied sample — do not just take the top 10 chronologically. You want range, because a writer's voice shows up differently across genres. Aim for a mix of:

- A strong-opinion / "hot take" post ("Please Stop Using X", "X Should Not Exist")
- A deep technical dive that explains a mechanism
- A principles or process piece (naming, refactoring, career advice)
- A personal or year-in-review piece (these expose the most natural voice)
- A tutorial or "how to" piece
- A short post (under 5 min read) and a long post (over 10 min read)

Six is the floor; ten is plenty. More than that and you start hitting context limits with diminishing returns on insight.

### 2. Read each article VERBATIM

This is the part that goes wrong by default. `WebFetch`'s default behavior is to summarize, which is useless here — you need the actual prose to study sentence rhythm, hedges, transitions, parenthetical asides. Use this exact prompt shape:

> Output the FULL article body as plain text VERBATIM. Do NOT summarize. Include intro, all paragraphs, headings, transitions, parenthetical asides, conclusion. Preserve sentence structure word-for-word. Skip code blocks but indicate where they appear.

If `WebFetch` still summarizes (it sometimes does on the first call), fall back to `curl -s <url>` via `Bash` and read the raw HTML — the prose is in `<p>` tags inside the `<article>` element. Strip the boilerplate mentally; you don't need to parse perfectly, you need quotable lines.

Run multiple `WebFetch` calls in parallel in a single tool-use block to keep this fast.

### 3. Take notes as you read

While reading, mentally (or in a scratch buffer) track:

- **Opener moves**: How does each article start? Pattern across pieces?
- **Hedges and intensifiers**: What words signal opinion? ("I think", "imo", "in my experience", "honestly", "frankly", "to be clear")
- **Transitions**: How does the writer move between sections? Rhetorical questions? Section headings? Connective phrases?
- **One-liner punchlines**: Sentences set off on their own line, often the thesis of a section.
- **Sentence-level tics**: Em-dashes? Semicolons? Parentheticals? Sentence fragments?
- **Section-level structure**: Hook → definition → example → problem → solution → caveat? Or something else?
- **Code presentation**: Block size, filename titles, naive-then-fixed pattern, prose-in-between density.
- **Emoji**: Which ones, where, how often? Headings or only in prose? Sign-off only?
- **Sign-off**: Does the writer end the same way every time?
- **Things they DON'T do**: No throat-clearing? No top-level TL;DR? No filler verbs like "delve" or "leverage"?
- **Coined phrases / branded patterns**: Names they invent for ideas and reuse ("The Latest Ref Pattern", "I'm team consistency").

### 4. Detect project writing rules

Read the project's `CLAUDE.md` (and the user-global one at `~/.claude/CLAUDE.md` if present) for any writing constraints. Common ones to look for:

- Em-dash prohibition
- Emoji restrictions
- Sentence/paragraph length caps
- Required or banned vocabulary
- Voice rules (no second-person, no "we", etc.)

Where the studied author violates a project rule, the guide must call it out with a substitution example (see the tkdodo.md template). The point: a future agent rewriting the user's article in this author's voice needs to know what to copy and what to translate.

### 5. Write the style guide

Output to `writing-styles/<author-slug>.md`. Use this exact section structure — it is what makes the guide usable. Each section is filled with verbatim quotes from the articles you read.

```markdown
# <Author Name> Writing Style Guide

<One paragraph: who this person is, where their blog lives, and what this document is for.>

## TL;DR

<3–5 sentences capturing the essence of their voice. End with the single most important "if you remember nothing else" instruction.>

## Voice and tone

### <Subhead: a named tonal trait, e.g. "First-person, conversational, direct">

<Prose explanation, then 2–4 verbatim quoted openers or representative sentences as blockquotes.>

### <Another tonal trait>
...

## Article structure

<List the skeleton as numbered stages: Hook, Definition, Baseline example, Pivot, Problem sections, Real-world example, Solution, Caveats, Learnings, Sign-off — adapt to what the author actually does. For each stage, give a verbatim example and explain when/why they use it.>

## Sentence-level moves

### The highlighted one-liner
### Rhetorical questions as transitions
### Parentheticals and asides
### Italics for emphasis
### Em-dashes / semicolons / punctuation tics
### Sentence fragments
### Lists, used <sparingly | heavily | for X>
### Coined / branded phrases

<Each subsection: 1 sentence of explanation + 2–3 verbatim quotes.>

## Code presentation

<Only include if the author writes about code. Cover block size, titles, the running-example pattern, how much prose surrounds each block, comment density inside vs outside code.>

## Emoji usage

<Which emoji they reach for, how often, where they appear (headings, sign-off, punchlines), and what they signal.>

## Things they do NOT do

<Bullet list of conspicuous absences. This is as important as what they do.>

## Project rule substitutions

<For each project-CLAUDE.md rule the author violates, give a before/after substitution. Example:

> **No em-dashes (project rule).** The author uses them liberally. Substitute with parentheses, semicolons, or split into two sentences.
>
> Author: "This becomes a nightmare to navigate — and eventually, no one will dare to remove a single memoization."
> Adapted: "This becomes a nightmare to navigate; eventually, no one will dare to remove a single memoization."

If no rules conflict, write "No substitutions needed — the author's style is compatible with this project's writing rules.">

## A compressed checklist for re-writing an article in this voice

<10-step numbered list. The last step should be "pass over the draft and apply project substitutions" if any apply.>

## Worked example: rewriting a flat paragraph in this voice

**Before (generic technical prose):**

> <2–3 sentences of bland AI-flavored writing on a topic the author has actually covered.>

**After (<author> voice):**

> <Rewrite using the moves you identified. Make it obviously theirs. Then in one sentence below, note what changed and why.>

## Source articles consulted

<Bulleted list of the article titles you actually read. No URLs needed if the index URL is in the intro paragraph.>
```

### 6. Quality bar before declaring done

Before finishing, check:

- **Quote density**: At least 2–3 verbatim quotes in every major section. If a section is all prose with no quotes, you're paraphrasing the author's style instead of demonstrating it.
- **Specificity**: No claim like "the author is engaging" or "the writing is clear". Every observation is mechanical: "opens with a personal admission", "uses italics on the single word that flips the meaning of the sentence", "lists are 3–5 noun phrases, no trailing periods".
- **No hallucinated quotes**: Every blockquote must be copy-paste from an article you actually fetched. If you're tempted to write a quote from memory, fetch the article again instead.
- **Worked example is in voice**: The "After" rewrite should be unmistakably the author's. If you read it back and it sounds like generic competent prose, redo it.
- **Project rules called out**: If `CLAUDE.md` has writing rules and the author violates them, the substitutions section must exist and have at least one before/after.
- **Length**: 200–350 lines is the right range. Under 150 lines means you're being too abstract; over 400 means you're padding.

## Why this structure

The output is a tool, not an essay. Someone (likely a model) will read this guide while rewriting an article and try to apply it. That's why every section is concrete, every claim has a verbatim example, and the worked before/after at the end shows what success looks like in miniature. Abstract observations like "the author is conversational" are useless to the rewriter; specific moves like "ends sections with a rhetorical question on its own line" are immediately applicable.

The verbatim-quotes rule is the single most important constraint. A style guide that paraphrases the author has already lost the very thing it's supposed to capture — the actual surface texture of how they write.

## Common failure modes

- **WebFetch summarized instead of giving raw prose.** Reissue with the explicit "VERBATIM, do not summarize" prompt, or fall back to `curl`.
- **All sample articles are the same genre.** You'll miss range. Re-pick to cover opinion, technical, personal, principles.
- **Quotes are too long.** Trim to the smallest excerpt that demonstrates the move. One sentence is usually enough.
- **Writing about the topic, not the style.** If a section talks about what the author *thinks* about React, you're off-track. Talk about how they *write* about React.
- **Skipped the project CLAUDE.md.** The guide is supposed to be usable in *this* project. Project rules matter.

## Example output

A reference example produced from this skill exists at `writing-styles/tkdodo.md` in the zubin.dev repo. When in doubt about depth, density of quotes, or section length, match that file.
