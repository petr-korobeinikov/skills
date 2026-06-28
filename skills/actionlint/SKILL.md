---
name: actionlint
description: >-
  Validates GitHub Actions workflow files with actionlint before they are
  committed, so obviously broken workflows never land in the repository.
  Installs actionlint through the mise skill when the project lacks it.
when_to_use: >-
  Use as a pre-commit precondition whenever staged changes include
  `.github/workflows/*.yml` or `*.yaml`, alongside vcs-identity-check and
  vcs-commit-msg; also when the user asks to lint, validate, or check a
  GitHub Actions workflow. Triggers: «проверь workflow»,
  «проверь воркфлоу», «завалидируй github actions», «линтани экшены»,
  «actionlint», «lint workflow», «validate github actions»,
  «check the workflow before commit».
---

# actionlint

Keep obviously broken GitHub Actions workflows out of the repository.
Before a commit that touches workflow files,
run [actionlint](https://github.com/rhysd/actionlint) over the staged ones;
any non-zero exit blocks the commit until it's fixed.

This pairs with vcs-identity-check and vcs-commit-msg
as one more pre-commit precondition —
the three are independent, so their order doesn't matter.

## When in doubt, ask the operator

If anything is ambiguous,
a prerequisite is missing,
or you're unsure how to proceed —
stop and ask the operator instead of guessing.
One question is cheaper than a wrong commit,
a misfired fix,
or tokens spent working around a gap you could resolve by asking.
This applies to every step below.

## When this runs (the gate)

Run the procedure when the staged changes include a workflow file —
`.github/workflows/*.yml` or `.github/workflows/*.yaml`.
If no staged file matches, this skill does not apply;
stay silent.

**Remote is not checked.**
Workflow files under `.github/workflows/` exist only for GitHub Actions
(GitHub.com and GitHub Enterprise alike),
so their presence is already a sufficient signal —
checking that a remote points at `github.com` would only add false negatives
on Enterprise (custom domain) and on local repos with no remote yet.

**Scope is workflow files only.**
actionlint does not lint `action.yml` / `action.yaml`
(composite-action metadata),
so those are out of scope here.

## Procedure

Run every step below from the repository root —
`cd "$(git rev-parse --show-toplevel)"` —
since the paths `git` prints are relative to it,
and actionlint must resolve them from the same directory.

1. **Collect the staged workflow files**
   (Added / Copied / Modified / Renamed — deletions don't need linting):

   ```sh
   git diff --cached --name-only --diff-filter=ACMR -- .github/workflows \
     | grep -E '\.ya?ml$'
   ```

   Empty result → the gate doesn't match;
   stop.
2. **Ensure actionlint is available:** `command -v actionlint`.
   If it's missing, install it via the mise skill
   (see **Installing actionlint** below) —
   never `brew` / `go install` / curl-pipe.
   If actionlint can't be installed that way
   (no mise, or the project doesn't use mise),
   tell the operator and stop — don't reach for another installer.
3. **Confirm staged and working-tree versions agree.**
   actionlint reads from the working tree, not the index,
   so a staged workflow carrying extra unstaged edits
   would be linted in a different version than the one committed:

   ```sh
   git diff --name-only -- <staged-files…>
   ```

   Any output → the versions diverge;
   stop and ask the operator to re-stage or set the edits aside,
   rather than linting content that won't be committed.
4. **Lint the staged files**, passing them as arguments
   so unrelated errors in untouched legacy workflows don't block this commit:

   ```sh
   actionlint <staged-files…>
   ```

   Run plain actionlint —
   it auto-uses shellcheck / pyflakes only when they're on `PATH`,
   nothing to toggle for the gate.
   With step 3 clean, this checks exactly what will be committed.
5. **Act on the exit code** (`$?`):
   - `0` — clean;
     the workflow part of the commit is good to go.
   - `1` — actionlint found problems in the workflow,
     including invalid YAML and a wrong workflow structure
     (these carry the `[syntax-check]` tag) —
     this is the "obviously broken" case.
     Show the output and keep the commit blocked.
     Don't rewrite the workflow blindly:
     if the fix isn't obvious, ask the operator.
     After a fix, re-stage it (`git add`) and re-run from step 3;
     the commit clears only once exit is `0`.
   - `2` — invalid command-line argument:
     a mistake in how actionlint was invoked, not a workflow problem.
     Fix the invocation (step 4) and re-run.
   - `3` — actionlint could not read a file (`could not read …`):
     usually the paths didn't resolve —
     confirm the command ran from the repository root, then re-run.
6. **Clear the commit only on exit `0`.**
   This skill is the gate, not the commit itself —
   on `0` the workflow check has passed
   and the normal commit flow (vcs-identity-check / vcs-commit-msg) continues.
   Any non-zero exit keeps the commit blocked;
   don't reach for `--no-verify` or otherwise bypass it.

## Installing actionlint when it's missing

Install with mise.
If the mise skill is available, let it own the details
(version pinning, `mise.lock`, the backend policy) —
that's the canonical path here.

Without that skill but with the `mise` binary,
add actionlint yourself under `[tools]` in `mise.toml`:

```toml
[tools]
actionlint = "<version>"
```

Resolve `<version>` from `mise latest actionlint`
(all versions: `mise ls-remote actionlint`) — pin it fully, no `latest`.
The raw name `actionlint` resolves through aqua to `aqua:rhysd/actionlint`.
Either way: never `brew`, `go install`, or a curl-pipe script.

This skill does **not** install shellcheck or pyflakes.
actionlint picks them up automatically when they're on `PATH`
(extending its checks into `run:` shell and Python),
and their absence is not an error.
To turn the integrations off deterministically,
pass empty values: `actionlint -shellcheck= -pyflakes=`.

## Existing actionlint config

If the repo already has `.github/actionlint.yaml` (or `.yml`),
actionlint uses it automatically
(self-hosted runner labels, config-variables, per-path ignores).
Respect it:
don't overwrite it, and don't run `-init-config` over it.
The config is optional —
actionlint runs fine without one.

## Optional: enforce with a pre-commit hook

The procedure above protects commits the agent makes.
To also catch hand-typed commits, offer a pre-commit hook
(install only with the user's agreement — propose, don't impose).

actionlint ships an official pre-commit-framework integration.
In `.pre-commit-config.yaml`:

```yaml
- repo: https://github.com/rhysd/actionlint
  rev: <vX.Y.Z>
  hooks:
    - id: actionlint-system
```

Set `rev` to the matching release tag —
`v` plus the version from `mise latest actionlint`
(so version `X.Y.Z` → tag `vX.Y.Z`),
also listed at https://github.com/rhysd/actionlint/releases.

Use the `actionlint-system` hook id:
it runs the binary already on `PATH` (the mise-managed one)
instead of fetching its own via the Go toolchain (`actionlint`)
or Docker (`actionlint-docker`).
Install the `pre-commit` runner itself through the mise skill
(it's a Python CLI → the `pipx` backend).

## Not in scope

- SHA-pinning `uses:` references — that's pinact / Dependabot, not actionlint.
- CI integration (handle separately).

## Reporting

After the check, state which workflow files were linted,
the exit code, the problems found (if any),
and whether the commit is cleared or blocked.
