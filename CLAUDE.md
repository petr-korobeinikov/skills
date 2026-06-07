# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code)
when working with code in this repository.

## What this repo is

A personal collection of **agent skills** —
plain-markdown capability packs for Claude Code and other agents
in the [vercel-labs/skills](https://github.com/vercel-labs/skills) ecosystem.
Not application code:
there is no build, no test suite, no runtime dependencies.
Treat the repo as a published content package.

## Layout

```
skills/
  <skill-name>/
    SKILL.md          # required
    scripts/          # optional, only if the skill needs helpers
    references/       # optional, only if the skill needs examples/templates
```

Each skill is a folder with at least `SKILL.md`.
Frontmatter `name` must match the folder name.

## SKILL.md frontmatter rules

- `name` — lowercase, hyphen-separated, unique within `skills/`.
- `description` —
  what the skill does, in one or two sentences.
  Keep it concise;
  trigger conditions go in `when_to_use`.
- `when_to_use` —
  trigger conditions for the agent's relevance matching:
  phrases the user might say, file types, project shapes.
  Include both Russian and English triggers when relevant —
  this user works in both.
  The agent matches against `description` + `when_to_use` combined,
  so concrete triggers here are critical.
- Use YAML block scalar `>-` for `description` and `when_to_use`
  so frontmatter lines stay readable (≤80 chars).
  `>-` folds newlines into spaces and strips the trailing newline,
  yielding a parsed value identical to a single-line scalar.

## Skills must stay portable

Skills are installed into arbitrary projects via
`npx skills add petr-korobeinikov/skills`.
So:

- Never hardcode this repo's name (`petr-korobeinikov/skills`)
  or any specific path inside `SKILL.md` files.
- Skills that create files should detect the host project's existing layout
  (`skills/<name>/`, `.claude/skills/<name>/`,
  or flat `<name>/SKILL.md` at root)
  and follow it.
  Default to `skills/<name>/` if no existing skills are found.
- Do not introduce build tooling, `package.json`, lockfiles,
  or runtime dependencies —
  skills are plain markdown and shipped as-is.

## Formatting

All Markdown files in this repo (`SKILL.md`, `README.md`, `CLAUDE.md`)
must be formatted using Semantic Line Breaks (sembr) —
see `skills/sembr/SKILL.md` for the rules and https://sembr.org/ for the spec.
This is a repo-level convention:
it does NOT propagate when skills are installed elsewhere,
so `skill-new` stays neutral about formatting in host projects.

## Adding or editing a skill

Prefer invoking the `skill-new` skill (`skills/skill-new/`) —
it handles layout detection and template generation.
Manual path: create `skills/<name>/SKILL.md`,
fill frontmatter per the rules above,
write the procedure in the body.
Do not add a per-skill README;
`SKILL.md` is the documentation.

After adding or updating any skill,
update the **Skills** section in `README.md` to reflect the change
(name, one-line description, primary triggers).

## Distribution commands (for reference)

Skills target Claude Code only —
we do not maintain compatibility with other agents.
The standard install flags reflect that:

- `-a claude-code` —
  lands skills in the host's `.claude/skills/`
  instead of spreading copies across other agent configs.
- `--copy` —
  writes plain file copies instead of symlinks.
  Symlinks couple the host repo to this checkout
  and clutter `git status` in the consumer project.
- `-y` —
  skips interactive prompts when installing the full set.

```bash
# install all skills
npx skills add petr-korobeinikov/skills --skill '*' --copy -a claude-code -y

# install one skill
npx skills add petr-korobeinikov/skills --skill <name> --copy -a claude-code
```

Add `-g` for global (`~/.claude/skills/`) instead of project (`.claude/skills/`).
Consumers track versions via `.skill-lock.json` in their own project —
this repo does not maintain a lockfile.
