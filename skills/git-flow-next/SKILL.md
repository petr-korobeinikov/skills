---
name: git-flow-next
description: >-
  Installs git-flow-next (the actively maintained Tower / gittower fork
  documented at https://git-flow.sh/) via mise's aqua backend, initializes
  the classic Gitflow preset, configures squash as the upstream merge
  strategy for feature branches, and runs the feature / release / hotfix
  flow with `git flow …` commands.
when_to_use: >-
  Use whenever Gitflow tooling is involved: starting / finishing
  feature, release, or hotfix branches; setting up `git flow` in a new
  repo; migrating off the abandoned nvie git-flow or gitflow-avh forks.
  Triggers: «настрой git-flow», «поставь git-flow», «начни фичу»,
  «заверши фичу со сквошем», «git flow init», «set up gitflow»,
  «start a feature branch», «finish feature with squash»,
  «release branch», «hotfix branch». Also triggers when the repo
  already carries `gitflow.*` git config keys or branches under
  `feature/`, `release/`, `hotfix/` prefixes.
---

# git-flow-next

git-flow-next (https://git-flow.sh/, repo `gittower/git-flow-next`)
is the Gitflow CLI this skill applies to —
no other git-flow forks.

## Do not confuse with the abandoned forks

Both legacy tools ship a `git-flow` binary too,
which makes mix-ups easy.

- `nvie/gitflow` —
  the original 2010-era shell script,
  unmaintained for over a decade.
- `petervanderdoes/gitflow-avh` —
  the AVH fork,
  no longer actively developed.

If `command -v git-flow` resolves outside mise's shim directory,
or `brew list` shows `git-flow` / `git-flow-avh`,
follow the **migrate from old git-flow / gitflow-avh** procedure
below before installing git-flow-next.
The binary name (`git-flow`) is the same;
only one can win on `$PATH`.

## Hard rules

1. **Only git-flow-next is allowed.**
   No `nvie/gitflow`, no `gitflow-avh`, no shell-script forks.
2. **Install through mise's aqua backend**, per the `mise` skill —
   pinned in `mise.toml` as `aqua:gittower/git-flow-next`,
   recorded in `mise.lock`.
   The explicit `aqua:` prefix is required here,
   overriding the `mise` skill's preference for bare names:
   the bare name `git-flow` is ambiguous in mise's registry
   (legacy `nvie/gitflow` and `gitflow-avh` claim it too),
   so naming the aqua package directly is the only way to be sure
   git-flow-next wins.
   No `brew install git-flow-next`,
   no manual binary downloads,
   no `go install`.
3. **Squash is the upstream merge strategy for feature branches** —
   enforced once, in exactly one place per repo:
   - **Local-finish workflow** —
     set `gitflow.branch.feature.upstreamstrategy = squash`
     so `git flow feature finish` collapses the branch into a single
     commit on `develop`.
   - **PR-review workflow** —
     leave that key unset and configure the hosting platform's
     PR-merge button to **squash** (not merge / rebase),
     so the policy is enforced server-side.

   Pick one path per repo;
   mixing both double-merges or fails.
   Releases and hotfixes keep their default `merge` strategy
   so the merge commit and the version tag stay paired.
4. **Use the `classic` preset.**
   `main` and `develop` are the long-lived branches;
   topic prefixes are `feature/`, `release/`, `hotfix/`, `bugfix/`.
   Other presets (`github`, `gitlab`) are out of scope for this skill —
   if a project explicitly requires one,
   fall back to git-flow-next's upstream docs
   and do not apply the rest of this skill.

## Verifying memory against docs

git-flow-next is young and changes flag names, config keys, and
preset behavior between releases.
If anything below — command flags, config keys, preset contents,
binary names, version availability — looks unfamiliar or stale,
verify at https://git-flow.sh/docs before writing config or running
commands.
Do not guess.

## Procedure: install via mise

This skill specifies *what* to install;
the `mise` skill governs *how* —
creating `mise.toml` with the required `[settings]` block,
`mise trust` / `mise install` / `mise ls`,
the lockfile,
the commit.
If the project isn't on mise yet,
run the `mise` skill's setup procedure first;
do not duplicate its steps here.

git-flow-next-specific steps:

1. Pick a pinned version
   (releases:
   https://github.com/gittower/git-flow-next/releases).
2. Add the entry under `[tools]` in `mise.toml`
   via the explicit `aqua:` backend —
   the `pkgs/gittower/git-flow-next` aqua package ships a `git-flow`
   binary,
   and Hard rule 2 explains why the bare name is unsafe here:

   ```toml
   [tools]
   "aqua:gittower/git-flow-next" = "1.1.0"
   ```

3. After running the install per the `mise` skill,
   verify git-flow-next specifically:

   ```sh
   git flow version
   ```

   It should print the pinned version.
   If it prints something else,
   another `git-flow` binary is shadowing the mise-managed one on
   `$PATH` —
   locate it with `command -v git-flow` and remove it
   (the typical culprit is a Homebrew install of the old
   `git-flow` or `git-flow-avh`).

## Procedure: initialize a repository

Run from the repo root.
The `classic` preset configures Gitflow branches and prefixes:
`main` ← `develop` ← `feature/`,
plus `release/`, `hotfix/`, `bugfix/`.
`--defaults` skips the interactive prompts.

```sh
git flow init --preset=classic --defaults
```

If the production branch isn't `main`,
or any branch / prefix needs a different name,
drop `--defaults` and pass the documented overrides instead
(`--main=master`, `--develop=…`, `--feature=feat/`, etc.).

Then, in the local-finish workflow (Hard rule 3),
enable squash on feature finish:

```sh
git config gitflow.branch.feature.upstreamstrategy squash
```

In the PR-review workflow,
skip this command and configure the platform's PR-merge button
to squash instead.

Verify:

```sh
git config --get-regexp '^gitflow\.'
```

The output should list the preset's branch and prefix entries.
For the local-finish workflow it must also include
`gitflow.branch.feature.upstreamstrategy squash`;
for the PR-review workflow that key must be absent.

## Procedure: feature branch lifecycle

The procedure below describes the **local-finish** workflow
(Hard rule 3):
the squash-merge happens on the developer's machine via
`git flow feature finish`.
For the **PR-review** workflow,
jump to the alternative section below.

Start:

```sh
git flow feature start <name>
```

This creates `feature/<name>` from `develop` and checks it out.
Work on it normally —
commit, push for backup or sharing if needed.

Publish (optional, for shared work):

```sh
git flow feature publish <name>
```

Finish:

```sh
git flow feature finish <name>
```

With `upstreamstrategy=squash` set,
this:

1. switches to `develop`,
2. squash-merges `feature/<name>` into `develop` as a single commit,
3. deletes the local `feature/<name>` branch.

Push `develop` afterwards.
Argument-less shorthand works on the current branch:

```sh
git flow finish
```

### PR-review alternative

In the PR-review workflow (Hard rule 3),
the squash is produced by the hosting platform,
not by `git flow feature finish`.
After the PR is merged:

- the remote feature branch is already gone
  (the platform deletes it on merge, in the typical settings);
- delete the local copy with `git branch -d feature/<name>`
  (or `git flow feature delete <name>`);
- do not run `git flow feature finish` —
  it would double-merge or fail.

## Procedure: release branches

Releases keep the default merge strategy
so the merge commit and the tag stay paired.

```sh
git flow release start 1.4.0
# bump version files, update changelog, commit
git flow release finish 1.4.0 --tag --message "release 1.4.0"
```

`release finish` merges into `main`,
tags the merge,
merges `main` back into `develop`,
and deletes the release branch.

## Procedure: hotfix branches

```sh
git flow hotfix start 1.4.1
# fix, commit
git flow hotfix finish 1.4.1 --tag --message "hotfix 1.4.1"
```

`hotfix finish` mirrors `release finish`:
merge to `main`,
tag,
merge `main` back into `develop`,
delete the hotfix branch.

## Procedure: bugfix branches

Bugfix branches target `develop` like features
but are reserved for non-feature defects;
production fixes go through `hotfix`.
The lifecycle mirrors features:

```sh
git flow bugfix start <name>
# fix, commit
git flow bugfix finish <name>
```

`bugfix finish` uses the default `merge` upstream strategy.
If the repo wants bugfixes squashed like features
(local-finish workflow per Hard rule 3),
set the analogous key:

```sh
git config gitflow.branch.bugfix.upstreamstrategy squash
```

In the PR-review workflow,
treat bugfix branches the same as features —
the platform produces the squash,
no local `git flow bugfix finish`.

## Procedure: migrate from old git-flow / gitflow-avh

Trigger when `command -v git-flow` resolves to a Homebrew or manual
install of `nvie/gitflow` or `petervanderdoes/gitflow-avh`,
or when `brew list` / `apt list --installed` shows `git-flow` or
`gitflow-avh`.

1. Confirm the existing setup is a legacy fork:
   `command -v git-flow`,
   then `git-flow version` —
   the legacy AVH fork prints `AVH Edition`,
   nvie's prints a version like `0.4.x` with no Go/Tower attribution.
2. Read the legacy Gitflow config —
   only to extract the **branch and prefix names**,
   not to reapply verbatim.
   The legacy forks and git-flow-next use different key schemas
   (legacy: `gitflow.branch.master`, `gitflow.prefix.feature`;
   git-flow-next: `gitflow.branch.<type>.parent`,
   `gitflow.branch.<type>.prefix`),
   so the captured keys can't be replayed —
   they're a source for `git flow init` flag values:

   ```sh
   git config --get-regexp '^gitflow\.'
   ```

   Note the values for `gitflow.branch.master`,
   `gitflow.branch.develop`,
   `gitflow.prefix.feature`,
   `gitflow.prefix.release`,
   `gitflow.prefix.hotfix`,
   and `gitflow.prefix.bugfix`.
3. Remove the old binary:
   - Homebrew: `brew uninstall git-flow` or `brew uninstall git-flow-avh`.
   - Manual install: delete the binary from its `$PATH` location.
4. Install git-flow-next via the procedure above.
5. Re-run `git flow init`,
   passing the names captured in step 2 as overrides
   (omit any flag whose value already matches the classic-preset default):

   ```sh
   git flow init --preset=classic \
     --main=<captured master/main> \
     --develop=<captured develop> \
     --feature=<captured feature prefix> \
     --release=<captured release prefix> \
     --hotfix=<captured hotfix prefix> \
     --bugfix=<captured bugfix prefix>
   ```

   If every captured value matches the classic defaults,
   `git flow init --preset=classic --defaults` is enough.
6. Re-apply the squash setting per the workflow (Hard rule 3):
   - local-finish:
     `git config gitflow.branch.feature.upstreamstrategy squash`;
   - PR-review:
     skip the config and configure the platform's PR-merge button
     to squash.
7. Verify `git flow version` reports the git-flow-next version
   from `mise.toml`,
   and `command -v git-flow` resolves into `~/.local/share/mise/`
   (or wherever mise materializes shims).

## Reporting

After changes,
state which `mise.toml` entry was added or updated
(tool, backend, version),
which `gitflow.*` keys were set,
which legacy binaries were removed,
and the next commands the user should run
(`git flow version`,
`git flow feature start <name>`,
or whatever fits the current task).
