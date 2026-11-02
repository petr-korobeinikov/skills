---
name: mise
description: >-
  Configures mise as the sole runtime and CLI tool manager for a project.
  Every mise.toml carries [settings] disable_backends = ["ubi", "asdf"]
  and lockfile = true, pins every version fully, commits mise.lock and
  .mise/locks/, routes Python CLIs through pypi (each on its own private
  Python, in its own venv, from a locked wheel-only graph) and Node CLIs
  through npm, and rejects competing managers
  (brew/nvm/asdf/volta/fnm/pyenv/ubi/pip/pipx/uv tool).
when_to_use: >-
  Use when the user sets up a project, asks to install or pin a runtime
  or CLI tool (Go, Node, Python, Terraform, kubectl, etc.), or when
  brew install / nvm / asdf / volta / fnm / .tool-versions / .nvmrc /
  .mise.toml show up in the repo. Triggers: «настрой проект», «поставь
  go», «добавь node», «нужен terraform», «запинь версию», «mise»,
  «aqua», «поставь python cli», «pipx», «uv tool», «питон не видит
  пакет», «поставилось не в тот python», «set up tools», «pin a runtime»,
  «add a CLI», «wrong python», «migrate from nvm/asdf/brew/volta».
---

# mise

mise is the only allowed runtime and tool manager for projects this skill
applies to.
Every project pins all tools in `mise.toml` at the repo root,
records resolved versions in a committed `mise.lock`
and dependency graphs in a committed `.mise/locks/`,
disables the `ubi` and `asdf` backends,
routes Python CLIs through `pypi:`
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
   `pip`, `pipx`, or `uv tool` for a CLI,
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
   - Python CLIs (e.g. `black`, `ruff`, `poetry`) → `pypi:` —
     each CLI gets its own locked dependency graph,
     its own venv,
     and its own private Python,
     so no aqua-bundled Python sneaks in
     (see **Python CLIs: total isolation** below).
     `pipx:` is another name for the same backend and is not deprecated,
     so existing entries may keep it, under the same rules;
     write new ones as `pypi:`.
     mise treats `pipx:x` and `pypi:x` as different tools:
     changing the prefix means a new install and a new lock entry.
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
   `mise.lock` and `.mise/locks/` are committed alongside `mise.toml`.
5. **The project config is `mise.toml`** at the repo root.
   No `.mise.toml`,
   no `.tool-versions`,
   no `.nvmrc`,
   no `.python-version`,
   no `.ruby-version`.
   If `.mise.toml` already exists,
   do not edit it in place —
   run the migration procedure.
6. **Only the `[tools]` and `[settings]` sections are in scope,**
   plus the Python guard block and venv entry in `[env]` (see below).
   Other `[env]` entries,
   `[tasks]`,
   and other sections — handle separately.
7. **Local setup only.**
   CI integration is out of scope for this skill.
8. **`mise.lock` and `.mise/locks/` are mise-generated, never hand-edited.**
   Only a mise command may write them —
   normally `mise lock`.
   Never create, edit, or patch them by hand
   or with any non-mise tool:
   not the versions,
   not the checksums,
   not the platform entries,
   not the dependency graphs.
   To change what the lockfile records,
   edit `mise.toml` and run `mise lock`;
   the lockfile follows.

## Python CLIs: total isolation

A Python package is installed into exactly one interpreter
and run by that same interpreter.
"Installed into one Python, ran another, package not found"
must be impossible.
This takes mise 2026.10.24 or newer;
if `mise --version` is older,
ask the user to upgrade it first.

Every `pypi:` entry names its Python,
and `uv` is pinned next to it:

```toml
[tools]
uv = "0.13.6"
"pypi:black" = { version = "24.4.2", python = "3.14.10" }
```

That gives each tool:

- **A private interpreter.**
  The `python` option takes a full version.
  mise installs that runtime without adding it to the shell's `PATH`,
  and a `python` pin in `[tools]` cannot move the tool off it.
  Without the option the tool sits on the project's `python`
  and moves with every bump of it,
  or on the first Python on `PATH`,
  or, when that one does not fit the package,
  on a CPython that uv downloads for itself.
- **A venv of its own,**
  at `installs/pypi-<tool>/<version>~uv~<hash>/`,
  with system and user site-packages off and no `pip` inside.
  Only the tool's commands reach `PATH`,
  never the venv's `python`.
  Repos that lock the same graph share one venv.
- **A locked dependency graph.**
  `mise lock` writes `pyproject.toml` and `uv.lock`,
  with a hash for every wheel,
  to `.mise/locks/pypi-<tool>/<version>~<hash>/`,
  and records the path and a digest in `mise.lock`.
  A `--locked` install replays it with `uv sync --frozen --no-build`:
  wheels only,
  so no package code runs at install time.

`uv` is what locks and installs these tools.
0.12.10 is the oldest version that can lock a graph,
not a version to aim for:
releases before 0.13.4 carry known advisories,
so pin a current one.
`pipx.uvx = true` in `[settings]` is obsolete —
uv is already the default installer once it is installed —
so delete it on sight.
Pin `python` in `[tools]` only if the project itself runs Python;
the tools do not need it.

### Guard block

Any `mise.toml` with a `pypi:` tool or a `python` pin
also carries this block,
an exception to rule 6:

```toml
[env]
PYTHONPATH = false               # would shadow a tool's own packages
PYTHONHOME = false               # would redirect or break the interpreter
PIP_REQUIRE_VIRTUALENV = "true"  # pip refuses to install outside a venv
UV_PYTHON_DOWNLOADS = "never"    # uv cannot fetch a CPython of its own
```

A venv does not block `PYTHONPATH`:
a package found there takes over a tool's imports.
A stray `pip install` lands in the runtime
shared by every project on that Python.
mise applies the block in an activated shell,
in `mise x`,
and through shims;
only a tool started by its full install path escapes it.

### Keeping tools isolated

- **Run a tool by its command** (`black`),
  with no wrapper and never as `python -m black`.
  The shell's `python` does not have the package and must not:
  `No module named black` from it is the isolation working,
  not something to fix by installing the package again.
- **Never install by hand.**
  No `pip install`,
  `uv pip install`,
  `pipx install`,
  or `uv tool install` for a CLI,
  and nothing into a tool's venv.
  A plugin goes through the tool's `with` option,
  a companion with its own commands through `expose`;
  both are pinned with `==` and locked with the tool:

  ```toml
  "pypi:ansible-core" = { version = "2.21.5", python = "3.14.10", expose = ["ansible-lint==26.9.0"] }
  ```

- **No `uvx_args` or `pipx_args`.**
  They cannot be locked:
  the tool gets a version-only entry
  and a `--locked` install refuses it.
  `python`, `with`, and `expose` replace them.
- **A library the project's own code imports is not a tool.**
  `pypi:` makes nothing importable;
  see **Projects written in Python** below.

To see which Python a tool really runs on:

```sh
grep -E '^(home|version_info)' "$(mise where pypi:<tool>)/.mise-uv/.venv/pyvenv.cfg"
```

`home` must be `installs/python/<the pinned version>/bin`.
No such file means the tool was not installed from a graph.
With shims on `PATH`,
`command -v python3` names the shim
even when it falls through to another Python;
ask `python3 -c 'import sys; print(sys.executable)'` instead.

### Projects written in Python

Pin `python` and let mise keep the project's venv active:

```toml
[tools]
python = "3.14.10"

[env]
_.python.venv = { path = ".venv", create = true }
```

In a terminal with `mise activate`,
`python script.py` then runs in that venv with no wrapper,
and mise creates the venv on first use.
Without uv in the project,
`pip` is the venv's too,
so `pip install` lands where `python` looks.
With uv in the project the venv has no `pip`:
declare packages in `pyproject.toml` and run `uv sync`.
A `pypi:` tool in the same project stays on its own Python and venv.
Add `.venv/` to `.gitignore`.

Shims alone do not activate the venv,
so a shell without `mise activate`, such as an agent's,
gets `No module named …` from `python`;
call `.venv/bin/python` there.

### Projects and the machine

Projects do not see each other:
each gets only the tools in its own `mise.toml`,
and two projects can pin one tool at different versions and Pythons.

The machine's global config is inherited on purpose:
its tools (`docker`, say) stay available in every project,
and a project's entry overrides a global one.
That does not weaken the isolation above:
a global Python CLI has its own venv like any other,
and a global `python` cannot move a tool that names its own.
It only decides what `python3` means in a project with no pin,
so keep `python` itself out of the global config.
A `mise.toml` in a parent directory is inherited the same way;
keep none above a project that must stand alone.

### When `mise lock` leaves no graph

`mise lock` exits 0 even when it could not write a graph,
so its exit code proves nothing.
A locked entry in `mise.lock` carries a `uv` line:

```toml
[[tools."pypi:black"]]
version = "24.4.2"
backend = "pypi:black"
specifiers = ["24.4.2"]
uv = { path = ".mise/locks/pypi-black/24.4.2~87de8a48", digest = "sha256:…" }
```

An entry with only `version`, `backend`, and `specifiers` has no graph.
The output of `mise lock` says why:

| Output contains                                 | Cause                                                            | Fix                                                     |
|-------------------------------------------------|------------------------------------------------------------------|---------------------------------------------------------|
| `uv >= 0.12.10 is unavailable`                  | uv is not installed yet                                          | Step 1 of **lock and install**, then lock again         |
| `Python dependency locks require uv >= 0.12.10` | The pinned uv is too old                                         | Pin a current uv                                        |
| `No interpreter found`                          | No `python` option, or that runtime is not installed             | Add the option, step 1 of **lock and install**, re-lock |
| `has no usable wheels`                          | A dependency has no wheel for that Python                        | Step down the Python (see below)                        |
| `uvx_args or pipx_args`                         | These options cannot be locked                                   | Replace them with `python`, `with`, `expose`            |
| `exclude-newer`                                 | A release is younger than `minimum_release_age` (24h by default) | Wait, or pin an older version                           |
| `uses lockfile format version`                  | An older lockfile; formats 0 and 1 hold no graphs                | `mise lock --upgrade`                                   |

A brand-new Python is the usual cause of `has no usable wheels`:
the graph installs wheels only,
so every dependency must publish one for that interpreter.
PyYAML 6.0.3, for example, ships wheels up to `cp314`,
so cookiecutter 2.7.1 stays on Python 3.14.10 rather than 3.15.2.
Never answer this by allowing source builds.
Take the newest Python the whole graph has wheels for
(`mise latest python@3.14` resolves a minor series)
and record the reason next to the tool:

```toml
# 3.14, not 3.15: PyYAML 6.0.3 ships no cp315 wheels.
"pypi:cookiecutter" = { version = "2.7.1", python = "3.14.10" }
```

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
- which backend to use when neither pypi nor npm applies and aqua does
  not have the tool;
- whether the project's own code relies on `PYTHONPATH` —
  the guard block unsets it;
- whether to delete legacy config files
  (`.nvmrc`, `.tool-versions`, `Brewfile`, etc.) during migration;
- whether to make an exception for `asdf` —
  see **Tool not available** below.

If you cannot reach the network to resolve "the latest" version,
ask the user for an exact version instead of guessing.

## Verifying memory against docs

mise changes settings, registry entries, and backend behavior between
releases.
If anything here — backend syntax, settings keys, registry contents,
trust behavior, lockfile semantics — looks unfamiliar or stale,
verify at https://mise.jdx.dev/ before writing config or running
commands.
Do not guess.

## Procedure: lock and install

The other procedures end here.
`mise lock` runs no package code,
so a problem surfaces before anything is installed.

1. If `[tools]` has `pypi:` tools,
   install what locking needs:
   uv, and one `python@<version>` per distinct `python` option.

   ```sh
   mise install uv python@3.14.10
   ```

   Without uv, `mise lock` records no graph.
   Without the runtime, it has no interpreter to resolve against.
   The WARN `python installed but not activated` is expected here:
   the runtime stays private to the tool.
2. Write the lockfile and the graphs:

   ```sh
   mise lock
   ```

   If it warns that the lockfile uses an older format,
   run `mise lock --upgrade`.
3. Check that every `pypi:` entry in `mise.lock` has a `uv` line.
   If one does not,
   stop and fix the cause (see **When `mise lock` leaves no graph**).
4. Install strictly:

   ```sh
   MISE_IGNORED_CONFIG_PATHS="$HOME/.config/mise" mise install --locked
   ```

   The variable keeps the machine's global tools out of this one command:
   `--locked` would demand lock entries for them too
   and fail with `is not in the lockfile`.
   They stay available in the project as usual.
   `--locked` fails on a missing graph, a missing sidecar, or an edited graph.
   Plain `mise install` does not:
   with no usable graph it falls back to `uv tool install`,
   which resolves afresh and may build source distributions,
   running their code.
   It also accepts an edited graph and rewrites the digest.
   `mise x` installs a missing tool the plain way too,
   so run this before starting any tool.
5. Verify with `mise ls`.
   A `python` row with no config file is a tool's private runtime.
6. Commit `mise.toml`, `mise.lock`, and `.mise/locks/`.
   `.mise/locks/` must not be git-ignored:
   a fresh clone needs it for the strict install in step 4,
   which is all that clone has to run after `mise trust`.

Plain `mise lock` reuses a recorded graph.
To pull in dependency fixes for an unchanged tool version,
run `mise lock --bump pypi:<tool>`,
then the strict install in step 4.

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
   uv        = "0.13.6"
   "pypi:black"     = { version = "24.4.2", python = "3.14.10" }
   "npm:typescript" = "5.4.5"
   ```

   With a `pypi:` tool or a `python` pin,
   add the guard block from **Python CLIs: total isolation**.
   A project written in Python also gets the venv entry
   from **Projects written in Python**.

4. Run `mise trust` to authorize the file.
5. Run **Procedure: lock and install**.

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
   A `pypi:` tool follows **Python CLIs: total isolation**:
   the `python` option,
   `uv` pinned,
   the guard block in place.
   A `python` pin in a project written in Python
   follows **Projects written in Python**.
5. Run `mise trust`.
   In normal mode trust persists by path;
   in paranoid mode mise hashes the file and re-trust is required
   after each modification —
   running `mise trust` after every write covers both modes safely.
6. Run **Procedure: lock and install**.

## Procedure: migrate from another manager

Trigger when any of these are present:
`.mise.toml`,
`.nvmrc`,
`.tool-versions`,
`.python-version`,
`.ruby-version`,
`Brewfile` containing runtime or CLI tools,
`volta` or `fnm` config,
`pipx.uvx`, `uvx_args`, or `pipx_args` in `mise.toml`,
README lines like `nvm install …`,
`brew install <runtime>`,
or `pipx install …`.

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
   For a Python CLI on `uvx_args` or `pipx_args`:
   propose the entry from **Python CLIs: total isolation**
   and dropping `pipx.uvx`.
3. **Wait for explicit confirmation** before making changes.
4. After confirmation:
   write `mise.toml`,
   delete the obsolete config files,
   update docs to reference the strict install in **lock and install**.
5. `mise trust`,
   then **Procedure: lock and install**,
   committing the deletions with it.
   `mise prune --yes pipx:<tool>` removes an install left behind
   by a prefix change.

## Procedure: tool not available in any allowed backend

If `mise registry <tool>` shows only `ubi` or `asdf`,
or shows nothing at all:

1. Stop.
2. Surface the situation to the user:
   tool name,
   the backends mise lists for it,
   the project policy that disables `ubi` and `asdf`.
3. Offer alternatives in this order:
   - pick a different tool that has an aqua / core / pypi / npm /
     cargo / gem option;
   - install through a language-native backend
     (`pypi`, `npm`, `cargo`, `gem`)
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
which Python each `pypi:` tool runs on and whether its graph is locked,
which legacy files were removed,
the state of `mise.lock` (created or refreshed),
and the commands the user should run next
(`mise trust`, the strict install from **lock and install**, `mise ls`).
