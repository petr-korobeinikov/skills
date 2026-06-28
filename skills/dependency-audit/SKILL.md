---
name: dependency-audit
description: >-
  A hard pre-adoption gate for every third-party dependency, in any
  language. Before a new package — or a major version bump — is added in
  js, go, python, kotlin, rust, ruby, java, anything: resolve its exact
  identity, scan it for known CVEs, review its supply-chain behaviour on
  socket.dev and snyk, then install it inside a disposable,
  network-isolated Docker container with no host mounts to inspect its
  full transitive dependency tree and what it runs at install time.
  Nothing enters the project until it passes.
when_to_use: >-
  Use whenever a new third-party dependency is about to be introduced, or
  bumped across a major version, in ANY ecosystem — before `npm install`,
  `yarn add`, `pnpm add`, `go get`, `pip install`, `poetry add`,
  `uv add`, `cargo add`, `gem install`, `composer require`,
  `dotnet add package`, or editing `package.json` / `go.mod` /
  `requirements.txt` / `pyproject.toml` / `build.gradle(.kts)` /
  `pom.xml` / `Cargo.toml` to add an entry. Triggers: «добавь
  зависимость», «поставь библиотеку», «поставь пакет», «нужен пакет»,
  «давай возьмём библиотеку», «можно ли использовать этот пакет»,
  «безопасна ли эта библиотека», «проверь зависимость перед установкой»,
  «add a dependency», «install this package», «pull in a library»,
  «should we use this library», «is this package safe»,
  «vet this dependency», «audit a dependency before adding»,
  «npm install X», «go get X», «pip install X», «cargo add X».
---

# dependency-audit

Adding a third-party dependency is a supply-chain decision,
not a convenience.
Every package you pull in runs its code on your machine and in CI,
ships you its transitive dependencies,
and inherits your trust.
So nothing enters the project —
in **any** language,
direct or transitive —
until it has passed the audit below.

The reflex to just run
`npm install` / `go get` / `pip install` / `cargo add`,
or to hand-edit a manifest,
is **blocked** until the package clears the pipeline.
There is no "just this once":
a quick local experiment with an unvetted package
runs its install scripts on your host all the same.
If you must try something, try it in the sandbox (Stage 4) first.

## When in doubt, ask the operator

If anything is ambiguous,
a tool is missing,
or a signal sits on the line between sloppy and malicious —
stop and ask the operator instead of adopting on a guess.
A bad dependency is expensive to remove
and may have already executed code by the time you notice.
One question is cheaper than a compromised lockfile.
This applies to every step below.

## When this runs (the gate)

Run the procedure whenever a **new** third-party dependency
is about to be introduced,
or an existing one bumped **across a major version**
(a major bump is a fresh transitive surface).
Concretely, before any of:

- `npm install <pkg>` / `yarn add` / `pnpm add` / `bun add`
- `go get <module>`
- `pip install <pkg>` / `poetry add` / `uv add` /
  adding a line to `requirements.txt` or `pyproject.toml`
- `cargo add <crate>`
- editing `build.gradle` / `build.gradle.kts` / `pom.xml`
  to add `implementation(...)` / `<dependency>`
- `gem install` / `bundle add`, `composer require`,
  `dotnet add package`, and every other ecosystem's equivalent

It applies to **prod, dev, test, and build-tool / plugin** dependencies
alike —
a dev dependency's `postinstall` script
runs on your machine and in CI just the same as a prod one's.

It does **not** apply to the language standard library,
or to first-party / internal packages from your own trusted registry.
Re-pinning to a **patch** of an already-audited dependency
doesn't need the full pipeline;
a major bump does.

## The audit pipeline

Five stages.
Stages 0–3 are read-only intelligence
and can run in parallel once identity is fixed;
Stage 4 is the only one that executes the package's code,
and it always runs in isolation.
A blocking finding at any stage stops adoption
until it's explained or the package is rejected.

Per-ecosystem commands live in
[`references/ecosystems.md`](references/ecosystems.md);
service syntax and URLs (osv-scanner, OSV.dev, GitHub Advisories,
Socket, Snyk) live in
[`references/services.md`](references/services.md).

### Stage 0 — Identity & hygiene

Before any scan, pin down exactly **what** you're auditing —
supply-chain attacks live in the gap between
"the package I meant" and "the package I typed":

- The exact package name, registry / namespace,
  and the specific version you intend to adopt.
  Watch for typosquats,
  dependency-confusion
  (a public package shadowing an internal name),
  and look-alike scoped names.
- Confirm it's the real project:
  repository link, homepage, download counts, release cadence,
  last publish date, maintainer(s),
  and that the version isn't yanked or deprecated.
- License — compatible with this project?
  A vuln-free package under the wrong license is still a non-starter.

Red flags here alone can end the audit:
a brand-new package with no history impersonating a popular one,
a sudden maintainer change,
a version published hours ago with a spike of unusual activity.

Fetch these facts without installing anything:
**deps.dev** (`https://deps.dev/<ecosystem>/<name>`)
covers versions, licenses, advisories, and maintainer data
across npm / PyPI / Go / Maven / Cargo / NuGet,
and each registry has its own metadata command
(`npm view`, the PyPI JSON API, `go list -m -versions`, …) —
see [`references/services.md`](references/services.md).

### Stage 1 — Known vulnerabilities (CVE / OSV)

Check whether the package **and its transitive deps**
carry known, published vulnerabilities.
The candidate isn't in your project yet,
so there is no host lockfile to scan —
and you must not create one by resolving on the host.
Two paths respect that:

- **Query by name + version** — no lockfile, no install:
  OSV.dev's `/v1/query`,
  the GitHub Advisory Database,
  or the ecosystem's single-package check
  (e.g. `snyk test <pkg>@<ver>`, npm-only).
- **Scan the full resolved tree inside the Stage 4 sandbox,**
  where the lockfile actually exists and never leaves the container:
  run `osv-scanner` against it,
  or the ecosystem's native auditor —
  `npm audit`, `pip-audit`, `govulncheck` (Go, reachability-aware),
  `cargo audit`, OWASP dependency-check (JVM) —
  in the same disposable container that resolved it.

Whichever query you run,
confirm the tool **accepts** the ecosystem slug:
OSV rejects a bad one with HTTP 400 `Invalid ecosystem`,
and a plausible-but-wrong slug returns an empty hit —
neither is evidence of safety, only of a mis-aimed query.

A known, unpatched, reachable vulnerability with no fixed version
is a blocking finding.

### Stage 2 — Supply-chain behaviour (Socket)

CVE databases only know about vulnerabilities
someone has already reported.
[Socket](https://socket.dev) catches the other half —
what the package **does**:
install / `postinstall` scripts,
network / filesystem / environment-variable access,
shell access,
obfuscated or minified-only code,
native code,
typosquatting,
and sudden maintainer or capability changes —
the fingerprints of a malicious or compromised package
*before* any CVE is filed.

Treat high-risk alerts as blocking until explained:
an install script that reaches the network,
an obfuscated payload,
a capability that doesn't match the package's stated purpose
(a string helper that opens sockets).

### Stage 3 — Second opinion (Snyk)

[Snyk](https://snyk.io)'s vulnerability database and package-health view
is a strong independent cross-check on Stage 1:
known CVEs with severity, CVSS, and fix / upgrade paths,
license issues,
and a health score
(security / popularity / maintenance / community).
Use `snyk test` (needs a free account)
or the public package page on `security.snyk.io` (no login).

### Stage 4 — Isolated install & transitive inspection

This is where the gate has teeth,
and where the hard requirement lives:
**never resolve or install an unvetted package on your host.**
Do it inside a disposable Docker container
with **no host filesystem mounted**,
so install scripts —
which run arbitrary code —
cannot touch your machine,
your SSH keys,
your environment,
or your repo.

The container does two jobs:

1. **Enumerate the full transitive dependency tree** the package drags in —
   the real blast radius,
   which Stages 0–3 only sampled at the top.
   A two-line package can pull a hundred transitive deps;
   each is its own supply-chain risk.
2. **Observe install-time behaviour safely** —
   first without running install scripts (recon),
   then, only if you need to, with scripts but network-isolated,
   watching what tries to phone home.

The recipe is the next section.

## The Docker sandbox

Hardened base invocation —
swap the image and the install line per ecosystem
([`references/ecosystems.md`](references/ecosystems.md)):

```sh
docker run --rm \
  --network bridge \
  --cap-drop ALL \
  --security-opt no-new-privileges \
  --pids-limit 512 \
  --memory 1g \
  <ecosystem-image> sh -c '<install-and-inspect>'
```

Every flag earns its place:

- **No `-v` / `--mount`.**
  The single most important property:
  the host filesystem is never passed in
  ("без проброса диска").
  The container can do anything it likes to its own ephemeral layer;
  the host stays untouched.
- **`--rm`** — container and writable layer are destroyed on exit.
  Nothing persists.
- **`--network bridge`** — recon (Pass A) must reach the registry
  to resolve the tree, so it stays connected.
  The isolation that matters —
  running install scripts with **no** egress —
  is Pass B below:
  download with the network up,
  then cut it before anything executes.
- **`--cap-drop ALL`** — drop all Linux capabilities;
  installs don't need them.
- **`--security-opt no-new-privileges`** — block setuid escalation.
- **`--pids-limit` / `--memory`** —
  cap a fork-bomb or memory-bomb in a malicious install script.
- Optional extra hardening:
  `--read-only` with a `--tmpfs` for every path the toolchain writes —
  not just `/tmp`, but its cache and work dirs
  (npm's `~/.npm`, pip's `~/.cache`, the install target),
  or redirect those via `npm_config_cache` / `PIP_CACHE_DIR`;
  plus a non-root `--user` where the image supports it.

**Two-pass method:**

**Pass A — recon (no code execution where the ecosystem allows it).**
Resolve and read the tree without running install scripts.
Download needs `--network bridge`;
keep every other flag locked down.

```sh
# npm — --ignore-scripts skips pre/post-install hooks entirely
docker run --rm --network bridge --cap-drop ALL \
  --security-opt no-new-privileges --pids-limit 512 --memory 1g \
  node:lts-alpine sh -c \
  'cd "$(mktemp -d)" && npm init -y >/dev/null \
     && npm install --ignore-scripts --no-audit --no-fund <pkg> \
     && npm ls --all'
```

Read the tree:
how many transitive deps,
anything unexpected,
anything you recognise as risky,
a single maintainer controlling a deep chain.
(Go and Cargo run no code on resolve;
pip has no clean `--ignore-scripts`,
so prefer wheels — see the per-ecosystem notes.)

**Pass B — detonate (only if you must see what install scripts do).**
You can't change `--network` mid-run,
and `--rm` would discard the download —
so run a *detached* container,
fetch the artifacts with scripts still off,
cut the network,
then run the scripts offline:

```sh
cid=$(docker run -d --network bridge --cap-drop ALL \
  --security-opt no-new-privileges --pids-limit 512 --memory 1g \
  node:lts-alpine sleep 600)
docker exec "$cid" sh -c 'cd /tmp && npm init -y >/dev/null \
  && npm install --ignore-scripts --no-audit --no-fund <pkg>'   # fetch only
docker network disconnect bridge "$cid"                         # cut egress
docker exec "$cid" sh -c \
  'wget -T3 -qO- https://registry.npmjs.org >/dev/null 2>&1 \
     && echo REACHABLE || echo "egress blocked"'                # confirm
docker exec "$cid" sh -c 'cd /tmp \
  && npm rebuild --foreground-scripts'                    # run hooks, offline
docker rm -f "$cid"
```

A `postinstall` that errors because it couldn't reach the network,
read an env var,
or find an SSH key
has told you exactly what it wanted —
and it never had your host to reach for.
`npm rebuild` is bare on purpose:
it runs install hooks for the **whole** resolved tree,
transitive deps included —
which is where the malware hides.
`npm rebuild <pkg>` would detonate only the top package and miss them.
Swap the command for the ecosystem's equivalent
(the step that runs the install hooks).

## Decision: adopt or reject

Adopt only when **all** hold:

- No known unpatched, reachable vulnerability (Stages 1, 3).
- No unexplained high-risk Socket behaviour (Stage 2).
- The transitive blast radius is understood and acceptable (Stage 4).
- License is compatible.
- The package is real, maintained, and correctly identified (Stage 0).

Reject or escalate on any of:

- A known exploitable CVE with no fixed version.
- Install scripts that reach the network or read secrets / env
  without a clear, legitimate reason.
- Obfuscated, or minified-source-only, payloads.
- Typosquat / dependency-confusion / impersonation signals.
- A capability mismatch
  (a "left-pad" that opens sockets).
- Unmaintained **and** carrying its own risky transitive chain.
- License incompatibility.
- A transitive blast radius wildly out of proportion to the value
  (a date helper pulling 200 packages).

When it's borderline,
ask the operator,
propose a vetted alternative,
or vendor and pin a reviewed subset.
Don't adopt on a guess.

## After it passes

- Pin the **exact** version —
  never a floating range for something you just vetted.
- Commit the lockfile, with integrity hashes.
- Prefer hash / SHA-pinned references
  where the ecosystem supports them.
- Record what you checked —
  CVE status, Socket verdict, transitive count, sandbox observations.
  One line in the PR saves the next person the re-audit.
- Keep the surface minimal:
  fewer dependencies, and fewer *direct* ones especially.

## Installing the tools

Install `osv-scanner`, the `snyk` and `socket` CLIs,
and the native auditors
(`pip-audit`, `govulncheck`, `cargo audit`, OWASP dependency-check)
through the project's `mise` skill where possible;
otherwise follow [`references/services.md`](references/services.md).

Don't block the audit because one CLI is missing:
OSV.dev, `security.snyk.io`, and `github.com/advisories`
answer over plain HTTP with no install,
and Docker covers Stage 4.
Socket is the exception —
`socket.dev` serves browsers only (403 to agents),
so an automated run needs the Socket CLI (with a token)
or escalates that stage to a human;
a 403 is not "nothing found".
Docker is the one hard prerequisite for Stage 4;
if it isn't available, say so,
and do **not** fake the isolation by installing on the host.

## When the ecosystem isn't listed

The five stages are language-agnostic;
only the tooling changes.
If a package's ecosystem isn't covered in
[`references/ecosystems.md`](references/ecosystems.md) —
a newer language, a niche package manager —
do **not** skip the audit,
and do **not** fall back to installing on the host.
Take the listed ecosystems as the template
and walk the same five stages,
substituting the equivalent tooling:

- **Identity:**
  the ecosystem's native registry
  (`pub.dev`, `hex.pm`, …) when deps.dev doesn't reach it —
  for maintainers, downloads, release cadence, yank status.
- **Known vulns:**
  `osv-scanner` against the ecosystem's lockfile (it is cross-ecosystem),
  the ecosystem's own native auditor if one exists,
  and whatever OSV.dev / GitHub Advisories / Socket / Snyk coverage
  it has.
  **Resolve the exact slug first:**
  services spell ecosystems differently
  (OSV `Pub` / `Hex`, GitHub `pub` / `erlang`),
  and a rejected or wrong slug returns nothing —
  which is *not* a clean result, only an un-run scan.
  The canonical strings are in
  [`references/services.md`](references/services.md).
- **Sandbox (Stage 4):**
  a disposable official base image for that toolchain,
  the **same hardened flags** as the listed ecosystems
  (no host mount, dropped capabilities, bounded pids / memory),
  and the **same two passes** —
  Pass A recon on `--network bridge`,
  Pass B detonation via the detached-container /
  `docker network disconnect` dance —
  with the ecosystem's own commands to install,
  enumerate the full transitive tree,
  and observe install-time behaviour.

The method is invariant whatever the language:
clean container,
no host mount,
resolve,
enumerate the transitive deps,
scan,
observe.
If you genuinely can't find tooling for a stage,
say which stage is uncovered and ask the operator —
don't silently drop it,
and don't let "no tool for this language" become an excuse
to adopt blind.

## Scope

- **Every ecosystem:**
  js / ts, go, python, kotlin / java / jvm, rust, ruby, php, .net,
  swift, dart — no exceptions.
- **Direct and transitive:**
  Stage 4 exists precisely because the risk hides in transitive deps.
- **Every dependency kind:**
  prod, dev, test, build-tool, plugin.
- **New adoptions and major bumps;**
  patch bumps of already-audited deps are exempt.
- **Not** the standard library,
  and not first-party internal packages from a trusted source.

## Reporting

After the audit, summarise:

- The exact package + version audited.
- Stage 0 identity (real package, maintainer, license).
- Stage 1 / 3 known-vuln results (CVEs by severity; fixable?).
- Stage 2 Socket alerts.
- Stage 4 transitive tree size,
  any notable or risky transitive deps,
  and what the sandbox observed.
- The verdict:
  adopt (pinned to `X.Y.Z`) / reject (why) / escalate to the operator.
