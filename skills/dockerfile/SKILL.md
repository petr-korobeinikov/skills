---
name: dockerfile
description: >-
  Compose, optimize, and fix Dockerfiles for fast, cache-friendly builds and
  the smallest secure final image — multi-stage, deps-before-source layer
  caching, and a final base driven toward scratch / distroless / slim. Keeps
  builds portable when BuildKit is unavailable and secrets out of layers,
  runs non-root, pins bases, and gates on a linter.
when_to_use: >-
  Use when writing, reviewing, optimizing, or debugging a Dockerfile or
  Containerfile, or when a container build is slow or an image is too large.
  Triggers: «напиши dockerfile», «составь dockerfile», «оптимизируй
  dockerfile», «исправь dockerfile», «почему образ такой большой», «уменьши
  образ», «ускорь сборку docker», «сборка долгая», «кэш слоёв», «многостадийная
  сборка», «FROM scratch», «distroless», «плохие практики в dockerfile»,
  «залинтить dockerfile»; «write/compose a dockerfile», «optimize dockerfile»,
  «fix this dockerfile», «shrink the image», «speed up the docker build»,
  «layer caching», «multi-stage build», «FROM scratch», «distroless»,
  «dockerfile best practices», «lint dockerfile», «hadolint», «droast»,
  «cache go.mod / package.json / requirements.txt / pom.xml / Cargo.toml layer».
  Also on the presence of a `Dockerfile`, `Containerfile`, or `*.dockerfile`
  in the repo.
---

# dockerfile

A good Dockerfile is judged on three axes at once,
and this skill optimizes all three without trading one for another:

- **Fast rebuilds** —
  order layers so a source edit never re-runs dependency installation.
  Copy the dependency manifest and resolve it *before* the source.
- **Small, secure final image** —
  a multi-stage build keeps the compiler, SDK, and dev dependencies
  in a build stage;
  the final stage carries only the artifact and its runtime.
  Aim the final `FROM` at `scratch`,
  and step down the ladder only as far as the runtime forces you.
- **Correct and safe** —
  non-root, exec-form entrypoint,
  no secret ever written into a layer,
  pinned bases, and a linter as the gate.

Two levers do most of the work: **cache** (§2) and **size** (§3).
Everything else is hygiene the linter enforces (§6).

Per-language recipes —
a complete, current multi-stage Dockerfile
for Go, Rust, C/C++, Node, Python, Java, .NET, Ruby, PHP, Elixir —
live in [`references/languages.md`](references/languages.md).
The bad-practice catalog,
mapped to the linter rules that flag each,
lives in [`references/anti-patterns.md`](references/anti-patterns.md).

## When in doubt, ask the operator

Do not silently rewrite a working build into one you can't verify.
If the base image,
the runtime's real dependencies,
whether a binary is truly static,
or whether the target builder has BuildKit
is unclear — **ask, or verify, before changing it.**
A Dockerfile that builds smaller but no longer runs,
or leaks a token into `docker history`,
is worse than the slow one it replaced.
Prefer a change you can build and run over a clever one you can't.

## 1. First, know your builder — buildx may be absent

Several of the fastest techniques below are **BuildKit-only** and
**hard-fail on the legacy builder** —
so the very first question is *what will actually build this file.*
Don't assume `docker buildx`.

- BuildKit is the default for plain `docker build`
  only since **Docker Engine 23.0** (Feb 2023) on Linux;
  older engines fall back to the legacy builder.
- **buildx is a separate package** (`docker-buildx-plugin`).
  A daemon can be BuildKit-capable yet have no buildx plugin,
  and CI / Podman / Kaniko / Windows-container hosts
  may have no BuildKit at all.

What **hard-fails** on the legacy builder (each errors, not degrades):
`RUN --mount=type=cache|secret|ssh|bind`, `COPY --link`, `COPY --chmod`,
heredocs (`RUN <<EOF`), and `docker build --secret`.
The `# syntax=docker/dockerfile:1` line is **silently ignored** there —
it does *not* make a cache-mount Dockerfile portable.

What works on **every** builder (this is the portable foundation):
multi-stage builds (since Engine 17.05, 2017),
deps-before-source layer ordering,
`FROM scratch`, `.dockerignore`, `COPY --chown`, cache cleanup,
`USER`, exec-form `ENTRYPOINT`/`CMD`, `HEALTHCHECK`.

**The design rule this imposes:**
put the whole caching + size win on the portable foundation,
and treat BuildKit cache/secret mounts as an *accelerator* layered on top —
never as the thing the build depends on to work.
Deps-before-source layer ordering (§2) already caches the common case;
cache mounts only speed the "dependencies changed" path.

Detect what's available before reaching for a mount:

```sh
docker buildx version                          # present + version, or non-zero if absent
docker version --format '{{.Server.Version}}'  # >= 23.0 => BuildKit default on Linux
docker buildx ls                               # active builder + driver
```

If BuildKit is present but off: `DOCKER_BUILDKIT=1 docker build …`.
If buildx is genuinely unavailable and can't be installed,
either enable BuildKit via the daemon,
or emit a **BuildKit-free variant**
that relies only on layer-order caching
(no `--mount`, no `--link`, no `--chmod`, no heredocs) —
and say so explicitly.
There is **no in-Dockerfile conditional** that toggles cache mounts on and off,
so a single file cannot both use `RUN --mount=type=cache` and build on legacy.

> **Non-Docker builders:**
> Podman / Buildah *do* support `RUN --mount=type=cache` (Buildah 1.24+,
> shipped in Podman 4.x) and `--secret`, but not the `# syntax=` external frontend.
> Kaniko is archived and supports none of the mounts —
> prefer rootless BuildKit (`buildctl`) for daemonless CI.

## 2. Lever one — cache: order layers cheap→expensive, deps before source

This is the single most important optimization,
and it needs no BuildKit.

**How the cache decides.**
Each instruction is one layer with a content-addressed key,
and the key folds in every upstream key —
so **invalidation cascades**:
once a layer changes, every layer after it re-runs.
Two rules follow from how keys are computed:

- `COPY` / `ADD` (and `RUN --mount=type=bind`) key on a **checksum of the
  copied files' contents**, not their mtime.
  Change a file's bytes → the layer busts;
  touch its mtime alone → it doesn't.
- `RUN` keys on the **command string only** —
  not on what the command fetches.
  `RUN apt-get install curl` is *not* re-run next week
  just because upstream `curl` changed;
  force it with a changed earlier layer or `--no-cache`.

**The canonical shape — manifest first, source last:**

```dockerfile
# syntax=docker/dockerfile:1
FROM node:22-slim
WORKDIR /app
COPY package.json package-lock.json ./   # 1. manifest + lockfile only
RUN npm ci                               # 2. cached unless the manifest changes
COPY . .                                 # 3. editing source never busts step 2
RUN npm run build
```

Order instructions **least-frequently-changed first**.
The dependency install is expensive and stable;
source churns constantly —
so the install must sit above the source copy,
keyed only on the manifest.
Every ecosystem has the same two files to copy first:

| Ecosystem      | Copy first (manifest + lock)        | Then install                         |
|----------------|-------------------------------------|--------------------------------------|
| Go             | `go.mod go.sum`                     | `go mod download`                    |
| Node (npm)     | `package.json package-lock.json`    | `npm ci`                             |
| Node (pnpm)    | `pnpm-lock.yaml`                    | `pnpm fetch` → `pnpm install --offline` |
| Python (pip)   | `requirements.txt`                  | `pip install -r requirements.txt`    |
| Python (uv)    | `pyproject.toml uv.lock`            | `uv sync --locked --no-install-project` |
| Rust           | `Cargo.toml Cargo.lock`             | `cargo chef cook` (see references)   |
| Java (Maven)   | `pom.xml` (+ `mvnw .mvn`)           | `mvn dependency:go-offline`          |
| .NET           | `*.csproj`                          | `dotnet restore`                     |
| Ruby           | `Gemfile Gemfile.lock`              | `bundle install`                     |
| PHP            | `composer.json composer.lock`       | `composer install --no-scripts`      |
| Elixir         | `mix.exs mix.lock`                  | `mix deps.get`                       |

Full, per-line recipes for each are in
[`references/languages.md`](references/languages.md).

**Cache mounts — the BuildKit accelerator (optional, §1 applies).**
`RUN --mount=type=cache,target=…` persists a package-manager / compiler cache
*across builds*.
It is **never committed to the image layer** —
so it speeds rebuilds without bloating the image.
It helps the *dependencies-changed* path (re-download only what changed);
the plain layer ordering above already handles the *unchanged* path.

```dockerfile
# syntax=docker/dockerfile:1
RUN --mount=type=cache,target=/root/.npm      npm ci
RUN --mount=type=cache,target=/root/.cache/pip pip install -r requirements.txt
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    go build -o /bin/app ./cmd/app
```

Two caveats to state whenever you add one:

- On **ephemeral CI** (a fresh runner per job)
  the mount starts **empty** every run —
  `--cache-to` / `--cache-from` do **not** capture it.
  It pays off on a persistent builder, or via a cache-dance action.
- **apt** needs the cache kept and locked,
  because the default hook deletes `.deb`s
  and parallel builds corrupt the lists:

  ```dockerfile
  RUN rm -f /etc/apt/apt.conf.d/docker-clean; \
      echo 'Binary::apt::APT::Keep-Downloaded-Packages "true";' > /etc/apt/apt.conf.d/keep-cache
  RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
      --mount=type=cache,target=/var/lib/apt,sharing=locked \
      apt-get update && apt-get install -y --no-install-recommends gcc
  ```

**Without cache mounts, the classic apt one-liner still applies** —
and `update` + `install` must share one `RUN`
(splitting them pairs a stale cached `update` with a fresh `install`;
this is the most common Dockerfile bug):

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates \
      curl \
    && rm -rf /var/lib/apt/lists/*
```

## 3. Lever two — size: multi-stage, aim the final FROM at scratch

A build toolchain is 10–100× the artifact it produces.
Multi-stage keeps it in a build stage;
the final stage copies only the artifact and its runtime.
Multi-stage works on **every** builder — it is not a BuildKit feature.

**The base-image ladder. Start at `scratch`; step down only when forced.**

| Rung | Final base | You get | Step down when |
|------|------------|---------|----------------|
| 0 | **`scratch`** | nothing but what you `COPY` | you'd hand-assemble certs/tz/passwd, or can't produce a static binary |
| 1 | **`distroless/static:nonroot`** (~2 MB) | CA certs, tzdata, passwd (uid 65532), nsswitch, `/tmp` | the binary needs **libc** (cgo / glibc-dynamic) |
| 2 | **`distroless/base:nonroot`** (`base-nossl` if no OpenSSL) | + glibc (+ libssl) | you need **libstdc++/libgcc** (C++, glibc-dynamic Rust) |
| 3 | **`distroless/cc:nonroot`** | + libstdc++, libgcc | you need a shell / package manager as a first-class feature |
| 4 | **`alpine`** | musl libc, busybox, `apk` | musl breaks you (glibc-only binaries, Python wheels, DNS quirks) |
| 5 | **`debian:trixie-slim`** | glibc, `apt`, full shell | broadest compatibility, many system deps |

**Which languages actually reach `scratch`:**

- **Static-compiled → yes.**
  Go (`CGO_ENABLED=0`), Rust (musl static target),
  C/C++ (musl-static), Zig —
  a single statically-linked binary needs nothing else.
  **.NET NativeAOT** is the nearest non-static case
  (lands on `runtime-deps` chiseled).
- **Interpreted / VM → no.**
  Node, Python, Java, Ruby, PHP, Elixir ship a
  **dynamically-linked interpreter/VM plus its standard library on disk**;
  `scratch` has no `ld.so` or libc, so the first `exec` fails.
  Their floor is **distroless / slim / chiseled** —
  see the per-ecosystem table in
  [`references/languages.md`](references/languages.md).

**What a real service needs on `scratch`**
(supply each with `COPY --from=build`,
or drop to `distroless/static` which bundles all of it):

| Missing on scratch | Breaks | Fix |
|--------------------|--------|-----|
| CA certificates | outbound TLS (`x509: certificate signed by unknown authority`) | `COPY --from=build /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/` |
| tzdata | `LoadLocation`, non-UTC times | `COPY … /usr/share/zoneinfo`; Go: `import _ "time/tzdata"` |
| `/etc/passwd`, `/etc/group` | named `USER`, `$HOME`, user lookups | use **numeric** `USER 65532:65532`, or COPY a minimal passwd |
| `/tmp` | apps that write temp files | create in builder + `COPY --chmod=1777`, or mount a tmpfs at runtime |

> `/etc/resolv.conf`, `/etc/hosts`, `/etc/hostname` are **injected by the
> runtime** into every container including scratch — DNS config is not an
> image concern. (The old Go/nsswitch "DNS breaks on scratch" gotcha is a
> pre-Go-1.16 relic.)

**Beware the Alpine false economy.**
`alpine`'s musl libc is not a drop-in for glibc:
Python `manylinux` wheels and most native Node addons are glibc —
on musl they compile from source (much slower, larger build stage),
and musl's DNS resolver has caused intermittent failures in clusters.
For Python / native-heavy stacks,
`debian-slim` (glibc) is usually smaller *in practice* and far less surprising;
Chainguard/Wolfi give glibc at near-Alpine size.
Never `COPY` a musl-linked binary into a glibc image or vice versa.

The canonical illustration —
Go to `scratch`, deps cached, non-root:

```dockerfile
# syntax=docker/dockerfile:1
FROM --platform=$BUILDPLATFORM golang:1.25 AS build
ARG TARGETOS TARGETARCH
WORKDIR /src
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download
COPY . .
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 GOOS=$TARGETOS GOARCH=$TARGETARCH \
    go build -trimpath -ldflags="-s -w" -o /out/app ./cmd/app

FROM scratch
COPY --from=build /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=build /out/app /app
USER 65532:65532
ENTRYPOINT ["/app"]
```

**The same recipe, BuildKit-free** (portable to the legacy builder, §1) —
drop the `--mount` cache mounts and the automatic
`--platform` / `TARGETOS` / `TARGETARCH` args (the BuildKit-only parts);
the `# syntax` line is inert on legacy, so dropping it is just tidiness.
Deps-before-source layer ordering still caches the common case;
what you give up is the cache-mount speedup on the *dependencies-changed* path
and the `--platform` cross-arch plumbing (a legacy build targets the host arch):

```dockerfile
FROM golang:1.25 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o /out/app ./cmd/app

FROM scratch
COPY --from=build /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=build /out/app /app
USER 65532:65532
ENTRYPOINT ["/app"]
```

## 4. Secrets and supply chain — never bake a credential into a layer

**`ARG` and `ENV` are not for secrets.**
Both persist in the image:
`ENV` is in the config (`docker inspect`),
`ARG` values are in `docker history`
**and in `max`-mode provenance attestations**.
A build-arg token is recoverable by anyone with the image.
Audit with `docker history --no-trunc <image>`.

**The correct way (BuildKit) — mount the secret for one `RUN`:**

```dockerfile
# syntax=docker/dockerfile:1
# The secret mount exposes .npmrc (with the token) at /root/.npmrc for THIS RUN
# only; BuildKit tears the tmpfs down afterward, so it never enters a layer.
# (Contrast `npm config set …_authToken`, which WRITES the token into a real
#  /root/.npmrc that the layer then commits and `docker history` exposes.)
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm ci
```
```sh
docker build --secret id=npmrc,src=$HOME/.npmrc .
```

For private-repo dependencies, forward the agent, not the key:
`RUN --mount=type=ssh git clone …` with `docker build --ssh default`.

**If BuildKit is unavailable there is no safe build-time equivalent** —
`ARG` leaks.
The honest options: enable BuildKit;
pass the secret at **runtime** instead of build time;
or do the authenticated work in an intermediate stage
and `COPY --from` only the non-secret artifact forward.

**Supply chain:**
pin the base by immutable digest (`FROM img:tag@sha256:…`) —
a tag is mutable, `:latest` is the anti-pattern.
Pin the frontend
(`# syntax=docker/dockerfile:1` — floats patch/minor, no breaking changes)
on the first line.
Emit SBOM / provenance attestations in CI
(`docker buildx build --sbom=true --provenance=true`);
note `mode=max` provenance embeds `ARG` values,
one more reason secrets never go through `ARG`.

## 5. Runtime hygiene the image must carry

These are builder-agnostic and belong in almost every Dockerfile;
the linter (§6) flags their absence.
Detail + the bad→good form for each is in
[`references/anti-patterns.md`](references/anti-patterns.md).

- **Exec-form `ENTRYPOINT`/`CMD`** (JSON array), never shell form —
  shell form runs under `/bin/sh -c`, which is PID 1 and swallows `SIGTERM`,
  so `docker stop` waits out the grace period then `SIGKILL`s.
  Pattern: `ENTRYPOINT ["app"]` + `CMD ["--default-arg"]`.
- **PID 1 / signals** —
  if the app doesn't reap children or handle SIGTERM,
  add an init: `ENTRYPOINT ["tini","--"]` / `dumb-init`,
  or `docker run --init` (a runtime flag, useful for distroless).
- **Non-root, numeric `USER`** — e.g. `USER 65532:65532`.
  Numeric is required for Kubernetes `runAsNonRoot`,
  which validates from image metadata and cannot resolve a username.
  (The per-base uid choice — the tiers — is in
  [`references/languages.md`](references/languages.md).)
- **`WORKDIR`, not `RUN cd`** —
  a `cd` doesn't persist across `RUN` layers.
- **`.dockerignore`** —
  exclude `.git`, `node_modules`, build output, `.env`;
  it shrinks the context and stops churn-y files from busting `COPY . .`.
- **`HEALTHCHECK`** where a liveness signal matters
  (exec-form probe on distroless).
- **`COPY --chown`** to set ownership in the copy layer,
  not a later `chown -R` (which doubles a large tree's size).
  `--chmod` is BuildKit-only (§1).

## 6. Lint the file, then scan the image

Static linting catches Dockerfile structure;
scanning catches CVEs in the result.
They are complementary — run both.
Install these CLIs through a version-pinned tool manager such as **`mise`**
(pin each and record it in the lockfile),
rather than unpinned global installs.

### hadolint — the primary gate

[hadolint](https://github.com/hadolint/hadolint) parses the Dockerfile AST
**and runs ShellCheck on `RUN` scripts**,
so one pass gives Dockerfile rules (`DL####`) and shell rules (`SC####`).

```sh
mise use "aqua:hadolint/hadolint@<version>"    # or: docker run --rm -i hadolint/hadolint < Dockerfile
hadolint Dockerfile
hadolint --failure-threshold warning Dockerfile   # gate: exit non-zero at/above warning
```

Read the findings against the catalog in
[`references/anti-patterns.md`](references/anti-patterns.md),
which maps each high-value rule
(DL3006/DL3007 pin the base, DL3008/DL3018 pin OS packages,
DL3009 clean apt lists, DL3025 exec-form CMD, DL3059 layer sprawl,
DL4006 pipefail, DL3002 non-root, DL3020 COPY-not-ADD, …)
to its fix.
Configure via `.hadolint.yaml`
(`failure-threshold`, `ignored`, `trustedRegistries` —
note the key is `trustedRegistries`, not `allowedRegistries`);
silence a line inline with `# hadolint ignore=DL3008`.
Don't blanket-ignore — each ignore should name why.

### droast — optional second, opinionated pass

[dockerfile-roast](https://github.com/immanuwell/dockerfile-roast)
(binary `droast`) is a **standalone** Rust linter
(its own `DF###` rule set, not a hadolint wrapper) — a useful independent net.
As a Rust CLI it installs through mise's `cargo` backend (pinned),
rather than an unpinned `curl | sh` bootstrap:

```sh
mise use "cargo:dockerfile-roast@<version>"    # binary: droast
droast Dockerfile
droast --preset production --min-severity warning Dockerfile
droast --format sarif Dockerfile               # or: json | github | compact (CI)
```

Zero-install alternative:
`docker run --rm -v "$PWD/Dockerfile":/Dockerfile ghcr.io/immanuwell/droast /Dockerfile`.
Presets: `minimal | security | performance | production | strict`;
exit `0` clean (or with `--no-fail`), `1` on findings;
`droast init --from-hadolint .hadolint.yaml` migrates an existing config.
Treat hadolint as the mature primary gate
and droast as a complementary opinion —
reconcile, don't blindly obey both.

### Scan the built image

- **dive** (`aqua:wagoodman/dive`) —
  per-layer wasted space and an efficiency score;
  `CI=true dive <image>` gates on `.dive-ci` thresholds.
- **Trivy** (`aqua:aquasecurity/trivy`) —
  CVEs *and* Dockerfile misconfig
  (`trivy config .`, `trivy image --severity HIGH,CRITICAL --exit-code 1`).
- **docker scout** (`docker scout cves`, `… recommendations`) —
  CVEs + base-image update advice; a Docker CLI plugin, **not** in aqua.
- **grype** (`aqua:anchore/grype`) —
  fast CVE scan, consumes Syft SBOMs.

> Exit-code gotcha: `trivy --exit-code N` uses your integer,
> while **docker scout and grype exit `2`** on findings —
> wire *non-zero* as the gate.

## Procedure

**Composing a new Dockerfile**

1. Identify the ecosystem
   and whether the artifact can be a **static binary**
   (Go/Rust/C/Zig → aim `scratch`)
   or needs a runtime (interpreted/VM → distroless/slim).
   Pull the matching recipe from
   [`references/languages.md`](references/languages.md).
2. Check the builder (§1).
   If BuildKit isn't guaranteed,
   keep to the portable foundation and drop cache/secret mounts
   (or ship a BuildKit-free variant).
3. Write it multi-stage:
   build stage installs deps **manifest-first** (§2), then source;
   final stage is the lowest ladder rung the runtime allows (§3).
4. Add hygiene (§5):
   numeric `USER`, exec-form entrypoint, pinned base + `# syntax`,
   and a `.dockerignore`.
5. Route any secret through `--mount=type=secret` (§4),
   never `ARG`/`ENV`.
6. Lint (§6), build, and **run it** —
   confirm it starts and serves, not just that it builds.

**Optimizing / fixing an existing Dockerfile**

1. Run hadolint (and optionally droast) first —
   it names concrete findings.
2. Reorder to manifest-before-source (§2);
   the biggest rebuild-time win.
3. Introduce (or fix) multi-stage
   and lower the final base one ladder rung at a time (§3),
   rebuilding and running after each step.
4. Fold split `apt-get update`/`install`,
   add `--no-install-recommends` + list cleanup,
   and pull any `ARG`/`ENV` secret onto a secret mount.
5. Pin the base (digest) and the frontend;
   add `.dockerignore` if missing.
6. Measure:
   `dive` for wasted space,
   `docker history --no-trunc` for leaks and fat layers,
   and compare final image size before/after.

## Reporting

After composing or fixing, summarize:

- The build strategy
  (stages, final base and which ladder rung, and why not lower),
  and whether it targets BuildKit or the portable foundation.
- What changed and the expected effect —
  rebuild-cache behavior (what now caches),
  final image size before → after.
- Secrets:
  how each is passed (secret mount / runtime),
  and that none is in `ARG`/`ENV`/`history`.
- Lint status:
  hadolint (and droast) findings —
  fixed, or ignored with a named reason —
  and any image-scan (Trivy/Scout) blockers.
- Anything the operator must supply at build or run time
  (build secrets, `--init`, a tmpfs mount, digest to pin).
