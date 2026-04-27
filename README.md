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

- **`skill-new`** —
  scaffolds a new skill in the current project's existing layout.
  Triggers: "создай скилл" / "create a skill".
- **`sembr`** —
  reformats prose using Semantic Line Breaks
  ([sembr.org](https://sembr.org/)), strictly.
  Two modes: explicit reformat on request,
  and default-on-edit for `.md` / `.rst` files
  (new prose in sembr, in-place edits match the existing paragraph,
  untouched paragraphs left alone).
  Triggers: "оформи по sembr" / "format with sembr",
  or any edit to a Markdown / reST file.
- **`mise`** —
  configures [mise](https://mise.jdx.dev/) as the sole runtime/tool
  manager,
  enforces aqua as the default backend,
  and requires fully pinned versions.
  Triggers: "поставь go" / "запинь версию" / "set up tools" /
  "migrate from nvm/asdf/brew".
- **`roast`** —
  direct critical review of an artifact (file, plan, idea, skill);
  surfaces substantive problems first and stops at findings.
  Triggers: "прожарь" / "разнеси" / "найди дыры" / "roast" /
  "tear apart".
- **`vcs-commit-msg`** —
  writes commit messages (git or any other popular VCS) that
  match the project's existing style, inferred from history;
  asks the user for any dimension that can't be detected;
  always shows the message before committing.
  Never mentions an AI agent or model in the composed message
  (no co-author trailer, no «Generated with …», no inline
  reference).
  Triggers: "сделай коммит" / "оформи сообщение коммита" /
  "make a commit" / "write a commit message",
  or any `git commit` / amend / reword flow.
- **`vcs-identity-check`** —
  verifies the VCS author identity (name + email) is set correctly
  before committing,
  and recommends configuring commit signing (GPG or SSH)
  when it isn't already set up;
  pairs with `vcs-commit-msg`.
  Triggers: "проверь git config" / "проверь автора" /
  "настрой подпись коммитов" / "check git identity" /
  "verify commit author" / "configure commit signing".
- **`which`** —
  forbids using `which` to check command availability or resolve a
  binary path; mandates the POSIX builtin `command -v` instead,
  with idioms for existence checks, path resolution, and edge cases.
  Triggers: "есть ли команда" / "проверь, установлен ли" /
  "which foo" / "check if X is installed" / "is X on PATH",
  or any shell snippet probing a command.

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
