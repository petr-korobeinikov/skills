# skills

Personal collection of agent skills for Claude Code and other compatible agents.

## Install

All skills:

```bash
npx skills add petr-korobeinikov/skills -a claude-code
```

Single skill:

```bash
npx skills add petr-korobeinikov/skills --skill skill-new -a claude-code
```

Add `-g` for global (`~/.claude/skills/`)
instead of project (`.claude/skills/`).

## Skills

| Skill                                                      | What it does                                                                                                    | Triggers                                                                       |
|------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------|
| [`skill-new`](skills/skill-new/SKILL.md)                   | Scaffolds a new skill in the project's existing layout.                                                         | "создай скилл" / "create a skill"                                              |
| [`sembr`](skills/sembr/SKILL.md)                           | Reformats prose using [Semantic Line Breaks](https://sembr.org/); auto-applies to `.md` / `.rst` edits.         | "оформи по sembr" / "format with sembr" / `.md` / `.rst` edits                 |
| [`mise`](skills/mise/SKILL.md)                             | Configures [mise](https://mise.jdx.dev/) as the sole runtime/tool manager; aqua backend, fully pinned versions. | "поставь go" / "запинь версию" / "set up tools" / "migrate from nvm/asdf/brew" |
| [`roast`](skills/roast/SKILL.md)                           | Direct critical review of an artifact; surfaces problems, stops at findings (no fix proposals).                 | "прожарь" / "разнеси" / "найди дыры" / "roast" / "tear apart"                  |
| [`vcs-commit-msg`](skills/vcs-commit-msg/SKILL.md)         | Writes commit messages matching the project's existing style (git or other VCS); never AI-attributed.           | "сделай коммит" / "make a commit" / `git commit` / amend / reword              |
| [`vcs-identity-check`](skills/vcs-identity-check/SKILL.md) | Verifies VCS identity (name + email) before committing; recommends commit signing (GPG/SSH) if missing.         | "проверь автора" / "check git identity" / "настрой подпись коммитов"           |
| [`which`](skills/which/SKILL.md)                           | Forbids `which`; mandates POSIX `command -v` instead.                                                           | "есть ли команда" / "which foo" / "check if X is installed"                    |

## Layout

```
skills/
  <skill-name>/
    SKILL.md          # required: frontmatter + instructions
    scripts/          # optional: helper scripts
    references/       # optional: examples, templates
```

## Add a new skill

Easiest: ask the agent to use the `skill-new` skill ("создай новый скилл …") —
it scaffolds the folder and `SKILL.md` for you.

Manually:

1. Create `skills/<skill-name>/SKILL.md`.
2. Fill in frontmatter:
   `name` (matches the folder),
   `description` (what the skill does, 1–2 sentences),
   `when_to_use` (trigger phrases, file types, project shapes).
   Use YAML `>-` block scalar for `description` and `when_to_use`
   so lines stay ≤80 chars.
3. Write the instructions below the frontmatter.
