---
name: act
description: >-
  Second line of defence after actionlint: runs GitHub Actions workflows
  locally with act (nektos/act) to check they actually work, not just that
  they parse. Proposes the run on a workflow commit; never blocks it.
when_to_use: >-
  After actionlint is clean on staged `.github/workflows/*.yml` or `*.yaml`,
  propose running them through act as a second check; also when the user asks
  to run, execute, or smoke-test a workflow locally. Triggers:
  «прогони workflow», «прогони через act», «запусти workflow локально»,
  «проверь что workflow реально работает», «act», «run workflow locally»,
  «test github actions locally», «nektos act», «does this workflow run».
---

# act

act is the second line of defence for GitHub Actions workflows,
behind the actionlint skill.
actionlint is static — it proves a workflow parses
and its expressions and types are sound.
act is dynamic — it runs the workflow locally in Docker
to see whether it actually works.

Use it as a proposal, not a gate:
once actionlint is clean on the staged workflows,
offer to run them through act and surface real failures —
but never block the commit on act.
Its local run can't cover everything,
and the local environment is fragile.

## When in doubt, ask the operator

If anything is ambiguous —
above all, whether a failure is the workflow's fault
or just the local environment's —
ask the operator instead of guessing.
One question is cheaper than calling a broken workflow fine,
or scaring the operator about a failure that's only a missing local image.
This applies to every step below.

## When this runs

- **After actionlint reports clean** on the staged workflow files —
  don't spend Docker on a workflow the cheap static check already rejected.
- It's a **proposal on every commit that touches a workflow**,
  not a requirement.
  Offer the run; if the operator declines, skip it — the commit is not blocked.

## Preconditions

The run needs a working Docker and a preconfigured image.
Check them first; if any is missing,
tell the operator, skip the run, and let the commit proceed —
never block over a precondition.

1. **Docker reachable** — `docker info` succeeds
   (any provider: Docker Desktop, colima, or podman via `podman info`).
   Missing binary or daemon down → act can't run; skip.
2. **A default runner image is configured.**
   On its first run act interactively asks for an image size,
   and with no TTY it dies with `fatal EOF` — so it must be set up beforehand.
   The config lives in act's `actrc`
   (`~/.config/act/actrc`, `~/.actrc`,
   or `~/Library/Application Support/act/actrc` on macOS);
   a configured one has a `-P` default line.
   If none is set, ask the operator to run `act` once and pick **Medium**
   (~500MB; Large is ~70GB, Micro is node-only) — don't guess an image.
   If you only discover this at run time (the `fatal EOF`), stop and ask then.
3. **Host flags.**
   - Apple Silicon — add up front:
     `uname -m` → `arm64` → `--container-architecture linux/amd64`.
   - Daemon-socket bind-mount failure (seen with colima) —
     add only reactively, after the first run shows it:
     `--container-daemon-socket -`.

## Procedure

1. **Confirm actionlint is clean** on the staged workflows first —
   via the actionlint skill, or a direct `actionlint` run.
   act is the second line, not the first.
2. **Propose the run** to the operator in one line —
   "run the staged workflow(s) through act to check they actually work?".
   Declined → skip and let the commit proceed (act never blocks).
3. **Check the preconditions** above,
   plus that act is installed (`command -v act`).
   If act isn't installed, treat installing it as a separate ask —
   it edits `mise.toml` (see below), which the run proposal doesn't authorize.
   Any precondition missing → tell the operator, skip, let the commit proceed.
4. **Run act on the staged workflow file(s)** —
   one `-W` run per staged workflow file.
   act reads the working tree, not the index,
   so first confirm the file has no unstaged edits
   (`git diff -- <file>` is empty);
   if it has, re-stage (`git add`) before running:

   ```sh
   mise exec -- act -W .github/workflows/<staged-file> [host-flags]
   ```

   - `mise exec -- ` only if act isn't already on `PATH`.
   - `[host-flags]` = whichever precondition-3 flags apply (arch / socket).
   - act picks the event (`push`, or the workflow's only one);
     narrow with `<event>` or `-j <job>` if needed.
   - Don't pre-filter jobs —
     let unrunnable ones surface as env failures (step 5).
   - Secrets: act reads `.secrets` automatically when it exists
     (or pass `--secret-file`); never invent them —
     a missing secret is an environment skip, not a failure.
5. **Read the result — split the two failure kinds,
   because act's exit code doesn't.** Go by the log markers:
   - **Success** — jobs end `✅` / `Job succeeded`:
     the workflow actually ran locally (within what act covers).
   - **Real workflow failure** — a *workflow step* (not "Set up job")
     exits non-zero on its own logic.
     Show it, recommend a fix, but don't block the commit.
   - **Environment / unsupported failure** — it couldn't get off the ground:
     `failed to start container`, `Error response from daemon`,
     `unable to find image` or no image for a macos/windows runner,
     `fatal ... EOF` (image prompt), a missing secret, OIDC, socket, arch.
     Also a `command not found` for a standard tool —
     almost always the Medium image lacking it, not the workflow;
     you can't inspect the image's contents, so default to env,
     and ask the operator if it matters.
     Report these as "not checked locally" with the reason, not as a failure.
   - **Can't tell?** Ask the operator.
6. **Never block the commit on act**, whatever the outcome.
   After the run (or a skip), the commit proceeds.

## Installing act when it's missing

Install with mise — and note that **Docker is a separate, required dependency**;
act can't run without it.
Under `[tools]` in `mise.toml`:

```toml
[tools]
act = "<version>"
```

Resolve `<version>` from `mise latest act`
(all versions: `mise ls-remote act`) — pin it fully, no `latest`.
The raw name `act` resolves through aqua to `aqua:nektos/act`.
Never `brew`, `go install`, or a curl-pipe script.
If act can't be installed that way, or Docker isn't available,
tell the operator and skip the run.

## Honest coverage — act is not your CI

A green act run is a smoke test, not proof the real CI passes.
act does **not** run macos/windows runners, OIDC,
or environments and env-scoped secrets;
it ignores concurrency, timeouts, continue-on-error,
step summaries, and annotations;
caching and services are partial.
Treat act as "does this get off the ground locally",
and the real CI as the source of truth.

## Not in scope

- Replacing real CI (act is a smoke test — see coverage above).
- Supplying or inventing secrets / credentials.
- Forcing runs that can't work locally — report them, don't fight them.

## Reporting

After the run, state which workflows or jobs ran,
the outcome of each
(passed / real failure / not-checked-locally with the reason),
and that the commit is not blocked either way.
