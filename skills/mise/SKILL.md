---
name: mise
description: >-
  Configures mise as the sole runtime and CLI tool manager for a project.
  Every mise.toml carries [settings] disable_backends = ["ubi", "asdf"]
  and lockfile = true, pins every version fully, commits mise.lock,
  routes Python CLIs through pipx and Node CLIs through npm, and rejects
  competing managers (brew/nvm/asdf/volta/fnm/pyenv/ubi).
when_to_use: >-
  Use when the user sets up a project, asks to install or pin a runtime
  or CLI tool (Go, Node, Python, Terraform, kubectl, etc.), or when
  brew install / nvm / asdf / volta / fnm / .tool-versions / .nvmrc /
  .mise.toml show up in the repo. Triggers: «настрой проект», «поставь
  go», «добавь node», «нужен terraform», «запинь версию», «mise»,
  «aqua», «set up tools», «pin a runtime», «add a CLI»,
  «migrate from nvm/asdf/brew/volta».
---

# mise

mise is the only allowed runtime and tool manager for projects this skill
applies to.
Every project pins all tools in `mise.toml` at the repo root,
records resolved versions in a committed `mise.lock`,
disables the `ubi` and `asdf` backends,
routes Python CLIs through pipx
and Node CLIs through npm,
and uses aqua (or mise's built-in core) for everything else.

## Hard rules

1. **mise is the only manager.**
   No `brew install`,
   `nvm`,
   `asdf`,
   `volta`,
   `fnm`,
   `pyenv`,
   `rbenv`,
   `tfenv`,
   `goenv`,
   ad-hoc installers,
   or curl-pipe scripts.
2. **Every `mise.toml` carries this `[settings]` block:**

   ```toml
   [settings]
   disable_backends = ["ubi", "asdf"]
   lockfile = true
   ```

   `disable_backends` removes `ubi` and `asdf` from the resolution
   chain project-wide;
   core runtimes (go, node, python, etc.) are unaffected because mise
   keeps their aqua and core entries.
   `lockfile = true` makes mise read and write `mise.lock`.
3. **Backend per tool:**
   - Python CLIs (e.g. `black`, `ruff`, `poetry`) → `pipx:` —
     mise's pipx backend uses `uvx` under the hood and isolates each
     CLI in its own venv,
     so no aqua-bundled Python sneaks in.
   - Node / JS CLIs (e.g. `typescript`, `prettier`, `eslint`) → `npm:` —
     mise's npm backend installs through a mise-managed Node,
     so no aqua-bundled Node sneaks in.
   - Everything else
     (runtimes, single-binary CLIs like `terraform`, `kubectl`) →
     bare name,
     resolved by mise to aqua or core.
   - Rust CLI not in aqua → `cargo:`.
     Ruby CLI not in aqua → `gem:`.
   - `ubi` and `asdf` are out of bounds (see rule 2).
4. **Versions are fully pinned.**
   No `latest`,
   no `lts`,
   no major-only (`20`),
   no minor-only (`20.11`).
   Always a complete version (`20.11.1`, `1.22.4`, `1.7.5`).
   `mise.lock` is committed alongside `mise.toml`.
5. **The project config is `mise.toml`** at the repo root.
   No `.mise.toml`,
   no `.tool-versions`,
   no `.nvmrc`,
   no `.python-version`,
   no `.ruby-version`.
   If `.mise.toml` already exists,
   do not edit it in place —
   run the migration procedure.
6. **Only the `[tools]` and `[settings]` sections are in scope.**
   `[env]`,
   `[tasks]`,
   and other sections — handle separately.
7. **Local setup only.**
   CI integration is out of scope for this skill.

## Confirming what mise will do

Run `mise registry <tool>` to see mise's backend chain for a tool.
With `ubi` and `asdf` disabled,
mise falls through to the next entry in the chain
(typically aqua or core).
If nothing remains,
mise refuses to install —
that is the expected, visible failure,
and the trigger for the **Tool not available** procedure below.

## Clarify before acting

Ask the user when any of these are unclear:

- which exact version to pin —
  never invent one,
  never write `latest`;
- which backend to use when neither pipx nor npm applies and aqua does
  not have the tool;
- whether to delete legacy config files
  (`.nvmrc`, `.tool-versions`, `Brewfile`, etc.) during migration;
- whether to make an exception for `asdf` —
  see **Tool not available** below.

If you cannot reach the network to resolve "the latest" version,
ask the user for an exact version instead of guessing.

## Verifying memory against docs

mise changes settings, registry entries, and backend behavior between
releases.
If anything below — backend syntax, settings keys, registry contents,
trust behavior, lockfile semantics — looks unfamiliar or stale,
verify at https://mise.jdx.dev/ before writing config or running
commands.
Do not guess.

## Procedure: new project setup

1. Confirm there is no existing `mise.toml` or `.mise.toml`.
   If `.mise.toml` exists,
   stop and run the migration procedure.
2. Create `mise.toml` at the repo root:

   ```toml
   [settings]
   disable_backends = ["ubi", "asdf"]
   lockfile = true

   [tools]
   ```

3. For every runtime and CLI the project uses,
   resolve a complete version
   (ask the user if unspecified)
   and add an entry per the **Backend per tool** rule.
   Examples:

   ```toml
   [tools]
   node      = "20.11.1"
   go        = "1.22.4"
   terraform = "1.7.5"
   "pipx:black"     = "24.4.2"
   "npm:typescript" = "5.4.5"
   ```

4. Run `mise trust` to authorize the file.
5. Initialize the lockfile and install:

   ```sh
   touch mise.lock && mise install
   ```

6. Verify with `mise ls` that the pinned versions are active.
7. Commit both `mise.toml` and `mise.lock`.

## Procedure: add or pin a tool

1. Verify `mise.toml` exists at the repo root with the required
   `[settings]` block.
   If `[settings]` is missing or incomplete,
   add it.
2. Resolve the exact version to pin
   (ask if unspecified).
3. Pick the backend per the **Backend per tool** rule.
   Run `mise registry <tool>` when in doubt about which backend mise
   will pick.
4. Add the entry under `[tools]`.
5. Run `mise trust`.
   In normal mode trust persists by path;
   in paranoid mode mise hashes the file and re-trust is required
   after each modification —
   running `mise trust` after every write covers both modes safely.
6. Run `mise install` to materialize the version and refresh
   `mise.lock`.
7. Verify with `mise ls`,
   then commit `mise.toml` and `mise.lock`.

## Procedure: migrate from another manager

Trigger when any of these are present:
`.mise.toml`,
`.nvmrc`,
`.tool-versions`,
`.python-version`,
`.ruby-version`,
`Brewfile` containing runtime or CLI tools,
`volta` or `fnm` config,
README lines like `nvm install …` or `brew install <runtime>`.

1. List every tool managed by the other system,
   resolving any ranges or `latest` markers to a concrete pinned
   version.
2. Present a migration plan to the user:
   the proposed `mise.toml` (with `[settings]` and `[tools]`),
   files that will be deleted,
   docs lines that need rewriting.
   For an existing `.mise.toml`:
   propose renaming to `mise.toml` and adding the required
   `[settings]` block —
   do not edit `.mise.toml` in place.
3. **Wait for explicit confirmation** before making changes.
4. After confirmation:
   write `mise.toml`,
   delete the obsolete config files,
   update docs to reference `mise install`.
5. `mise trust`,
   `touch mise.lock && mise install`,
   `mise ls`.
6. Commit `mise.toml` and `mise.lock`.

## Procedure: tool not available in any allowed backend

If `mise registry <tool>` shows only `ubi` or `asdf`,
or shows nothing at all:

1. Stop.
2. Surface the situation to the user:
   tool name,
   the backends mise lists for it,
   the project policy that disables `ubi` and `asdf`.
3. Offer alternatives in this order:
   - pick a different tool that has an aqua / core / pipx / npm /
     cargo / gem option;
   - install through a language-native manager
     (`pipx`, `npm`, `cargo`, `gem`)
     if the tool's language matches one;
   - as a last resort,
     ask the user to approve removing `asdf` from `disable_backends`
     for this project.
4. If the user approves the asdf override:
   keep `ubi` disabled,
   write the tool entry,
   and add a TOML comment on the entry recording why
   (tool name, what other backends were checked, why none worked).
   Never re-enable `ubi`.

## Reporting

After changes,
state which `mise.toml` was written or updated,
which tools were added or pinned (with backend and version),
which legacy files were removed,
the state of `mise.lock` (created or refreshed),
and the commands the user should run next
(`mise trust`, `mise install`, `mise ls`).
