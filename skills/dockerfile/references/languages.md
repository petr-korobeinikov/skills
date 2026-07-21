# Per-language multi-stage Dockerfiles

Complete, current (2025–2026) best-practice recipes for the `dockerfile` skill.
Each is multi-stage,
caches dependencies **manifest-before-source**,
and drives the final image as far down the base-image ladder (`SKILL.md` §3)
as the runtime allows.
Copy the flagship for an ecosystem, then adjust versions.

Pins here (`golang:1.25`, `node:22`, `python:3.13`, `-debian12`, …)
are illustrative —
**bump to a current release and pin by digest** (`FROM img:tag@sha256:…`)
in a real pipeline.

## Conventions (read once, applied throughout)

- **`# syntax=docker/dockerfile:1`** is the first line of every file.
  It selects the current BuildKit frontend
  and is what makes `RUN --mount=…` legal.
  On a non-BuildKit builder it is inert —
  drop the `--mount` lines and rely on layer ordering alone (`SKILL.md` §1).
- **Cache mounts (`--mount=type=cache`) are a BuildKit-only accelerator**,
  never committed to the image, and empty on ephemeral CI runners.
  The `COPY manifest → install → COPY source` ordering
  is the portable part that does the real caching.
- **Non-root, numeric `USER` in the runtime stage.**
  A *named* `USER` makes Kubernetes `runAsNonRoot` fail —
  the kubelet reads the uid from image metadata and can't resolve a name.
  Which uid: the base image's built-in user where one exists
  (`node` → `1000`, `www-data` → `82`, .NET chiseled → `1654`),
  the distroless `nonroot` uid `65532` on `scratch` / distroless,
  or a created user `10001` otherwise.
- **`COPY` runs as root; hand ownership over with `COPY --chown`.**
  The `--chown` may use the *name* (it resolves against the image's passwd
  at build time) — only the `USER` directive itself must be numeric.
- **Exec-form `ENTRYPOINT`/`CMD`** —
  mandatory on scratch/distroless (no shell).
- Ship a **`.dockerignore`**
  (`.git`, build output, `node_modules`/`.venv`/`target`/`bin`+`obj`/`_build`,
  secrets)
  so `COPY . .` neither leaks files nor busts the cache.

---

# Part 1 — Compiled languages → `scratch`

A statically-linked binary needs nothing but itself,
so the final stage can be the empty `scratch` image.
See `SKILL.md` §3 for what a real service must still `COPY` in
(CA certs, tzdata, passwd)
and the `distroless/static` alternative that bundles all of it.

## Go — the flagship scratch case

```dockerfile
# syntax=docker/dockerfile:1

FROM --platform=$BUILDPLATFORM golang:1.25 AS build
# BuildKit fills TARGETOS/TARGETARCH from --platform; building on $BUILDPLATFORM
# runs the cross-compiler natively instead of under slow QEMU emulation.
ARG TARGETOS TARGETARCH
WORKDIR /src

# Dependency layer: manifest only, resolved before any source is copied.
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
COPY --from=build /usr/share/zoneinfo /usr/share/zoneinfo
COPY --from=build /out/app /app
USER 65532:65532
ENTRYPOINT ["/app"]
```

Non-obvious lines:

- **`CGO_ENABLED=0`** disables cgo,
  forcing the pure-Go `net`/`os/user` implementations
  and a **fully static binary** —
  the one flag that makes `scratch` viable.
  (Output goes to `/out`, *not* under a cache mount,
  so it survives into the layer the final stage copies.)
- **`-trimpath`** strips build-machine paths out of the binary
  (reproducible, no path leakage in panics).
  **`-ldflags="-s -w"`** drops the symbol table and DWARF
  (~20–30% smaller; Go panic traces still work).
- **`USER 65532:65532`** numeric — needs no `/etc/passwd`.
  For a *named* user or `$HOME`,
  create a passwd entry in the builder and COPY it,
  or use `distroless/static:nonroot`.
- tzdata alternative:
  `import _ "time/tzdata"` embeds the zone database in the binary (~450 KB),
  dropping the `zoneinfo` COPY.

**Distroless alternative** —
beats scratch when you'd otherwise hand-copy certs/tz/passwd;
it bundles them, correct and maintained, at ~2 MB:

```dockerfile
FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/app /app
USER 65532:65532
ENTRYPOINT ["/app"]
```

**If the binary uses cgo / glibc**
(`CGO_ENABLED=1`, sqlite-via-cgo, a C dep)
it is dynamically linked → not scratch.
Use `gcr.io/distroless/base-debian12` (glibc + libssl)
or `…/base-nossl-debian12`.

## Rust — near-scratch via static musl + cargo-chef

Cargo has no clean "download dependencies only" step,
so a naive build recompiles every crate on any source edit.
[`cargo-chef`](https://github.com/LukeMathWalker/cargo-chef)
distills the dependency set into `recipe.json`
and compiles *only* dependencies,
in a layer reused until the dependency set changes.

```dockerfile
# syntax=docker/dockerfile:1
# One shared base => identical Rust version in every stage (or the cook cache
# is silently invalidated).
FROM lukemathwalker/cargo-chef:latest-rust-1 AS chef
RUN apt-get update && apt-get install -y --no-install-recommends musl-tools \
    && rm -rf /var/lib/apt/lists/* \
    && rustup target add x86_64-unknown-linux-musl
WORKDIR /app

FROM chef AS planner
COPY . .
RUN cargo chef prepare --recipe-path recipe.json

FROM chef AS builder
COPY --from=planner /app/recipe.json recipe.json
RUN --mount=type=cache,target=/usr/local/cargo/registry \
    --mount=type=cache,target=/app/target \
    cargo chef cook --release --target x86_64-unknown-linux-musl --recipe-path recipe.json
COPY . .
# Copy the binary OUT of the cache-mounted target/ in the SAME RUN — cache
# mounts are NOT part of the image FS, so a later COPY --from would find nothing.
RUN --mount=type=cache,target=/usr/local/cargo/registry \
    --mount=type=cache,target=/app/target \
    cargo build --release --target x86_64-unknown-linux-musl --bin app \
    && cp target/x86_64-unknown-linux-musl/release/app /app/app

FROM scratch
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /app/app /app
USER 65532:65532
ENTRYPOINT ["/app"]
```

- **`x86_64-unknown-linux-musl`** links statically by default → scratch-ready.
  (A `-gnu` target needs `RUSTFLAGS="-C target-feature=+crt-static"`,
  which is fragile — prefer musl.)
  `musl-tools` provides the `musl-gcc` linker.
- **The cache-mount cp-out trap**
  is the #1 reason a cargo-chef scratch build ships an empty binary —
  copy to a normal path inside the build `RUN`.
- **Prefer rustls over OpenSSL** for static builds
  (`default-features = false`, `features = ["rustls-tls"]`);
  with `webpki-roots` the trust store is compiled in
  and you can drop the `ca-certificates.crt` COPY.
- Strip via `Cargo.toml`:
  `[profile.release] strip = true`
  (+ `opt-level="z"`, `lto=true`, `codegen-units=1` for size).
- **glibc / C++ deps** →
  build on `rust:1` (gnu) and ship on `gcr.io/distroless/cc-debian12`.

## C / C++ — static link → scratch, or distroless

```dockerfile
# syntax=docker/dockerfile:1
# glibc -static breaks DNS/getpwnam on scratch — safe only for a non-networking
# binary; use musl for networking (see the glibc-static trap below).
FROM gcc:14 AS build
WORKDIR /src
COPY . .
RUN g++ -O2 -static -static-libgcc -static-libstdc++ -o /out/app main.cpp

FROM scratch
COPY --from=build /out/app /app
USER 65532:65532
ENTRYPOINT ["/app"]
```

**The glibc-static trap**
(the C/C++ analogue of Go's netgo story):
`-static` against **glibc** still `dlopen`s NSS shared libs at runtime,
so `getaddrinfo` (DNS) and `getpwnam` silently break on scratch.
Fixes, in order:
build against **musl** (Alpine toolchain / `musl-gcc`, or `zig cc`)
for genuinely self-contained networking binaries → scratch;
for non-networking CLIs glibc `-static` → scratch is fine;
otherwise link dynamically and ship on
`gcr.io/distroless/base-debian12` (C) / `…/cc-debian12` (C++).

## Zig and other statically-compiled languages

The recipe is identical for every static-binary language:
compile a static (ideally musl) binary → `COPY --from` into `scratch` →
add `ca-certificates.crt` if it does TLS →
numeric `USER` → exec-form entrypoint.
Zig makes musl-static cross-compilation a first-class flag
(`zig build-exe -O ReleaseSmall -target x86_64-linux-musl`)
and doubles as a clean C/C++ musl cross-compiler (`zig cc`).
It ships **no official Docker image** —
supply the toolchain from a community image or the verified tarball at
<https://ziglang.org/download/>, then apply the pattern above.

The same shape applies to Crystal (`--static`),
Nim (`--passL:-static`),
Haskell/OCaml (static + musl),
Swift (`--static-stdlib`),
and D.

---

# Part 2 — Interpreted / VM languages → distroless / slim

These ship a **dynamically-linked interpreter/VM
plus its standard library on disk**,
so they cannot reach `scratch`
(the first `exec` needs `ld.so` + libc).
The floor is a libc-bearing base:
distroless (smallest attack surface),
slim (the pragmatic default),
or chiseled (.NET).
See the base table at the end.

## Node.js

**`npm ci`, not `npm install`.**
`npm ci` requires a lockfile,
wipes `node_modules` for a deterministic tree,
errors on lock/manifest drift,
and never mutates the lockfile — correct for an image.
`npm install` reconciles and can bump versions.
Two current gotchas:
**Corepack is unbundled from Node 25+**
(`npm i -g corepack` first on `node:25`),
and **never set `NODE_ENV=production` in a build stage**
or npm drops the devDependencies your build needs —
set it only in the runtime stage.

```dockerfile
# syntax=docker/dockerfile:1
ARG NODE_VERSION=22

FROM node:${NODE_VERSION}-slim AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm npm ci

FROM node:${NODE_VERSION}-slim AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM node:${NODE_VERSION}-slim AS prod-deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm npm ci --omit=dev

FROM node:${NODE_VERSION}-slim AS runtime
ENV NODE_ENV=production
RUN apt-get update && apt-get install -y --no-install-recommends dumb-init \
    && rm -rf /var/lib/apt/lists/*
WORKDIR /app
COPY --from=prod-deps --chown=node:node /app/node_modules ./node_modules
COPY --from=build     --chown=node:node /app/dist         ./dist
COPY --chown=node:node package.json ./
USER 1000:1000
EXPOSE 3000
ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "dist/index.js"]
```

- `/root/.npm` is npm's cache
  (build RUNs are root; `USER 1000:1000` applies at runtime only).
  A separate `prod-deps` stage produces a dev-free `node_modules`;
  the one-stage alternative is
  `npm ci && npm run build && npm prune --omit=dev`.
- `dumb-init` (or `tini`, or `docker run --init`) gives a real PID 1 —
  Node was not designed as PID 1
  and otherwise ignores SIGTERM and leaks zombies.
  Never `CMD ["npm","start"]` in prod
  (npm forks Node and swallows signals).
  On Alpine: `apk add --no-cache dumb-init`.

**pnpm** —
corepack enable, then cache the content-addressable store;
`pnpm fetch` reads **only the lockfile**,
so its layer survives even `package.json` edits (great in monorepos):

```dockerfile
RUN corepack enable
COPY pnpm-lock.yaml ./
RUN --mount=type=cache,id=pnpm,target=/pnpm/store pnpm fetch
COPY . .
RUN --mount=type=cache,id=pnpm,target=/pnpm/store pnpm install --frozen-lockfile --offline
```

**yarn** —
classic: `yarn install --frozen-lockfile`
(cache `/usr/local/share/.cache/yarn`).
Berry (v2–4): `yarn install --immutable`,
COPY `.yarn/releases`,
and `yarn workspaces focus --production` for prod-only.

**Runtime bases.**
`node:22-slim` (Debian, built-in `node` uid 1000) is the default;
`node:22-alpine` is smaller but musl (native-addon/DNS risk);
`gcr.io/distroless/nodejs22-debian12:nonroot` is smallest —
**its entrypoint is already `node`**,
so `CMD ["server.js"]` runs `node server.js`
(there's no shell or dumb-init inside —
use `docker run --init` or a COPY'd static tini).

**Bun / Deno.**
`oven/bun` + `bun install --frozen-lockfile`.
Deno's `denoland/deno` runs deno as entrypoint;
`deno compile` bundles a runtime + code into one
**dynamically-linked** executable →
ship on `gcr.io/distroless/cc-debian12`, not scratch.

## Python

There is no static Python —
CPython is dynamically linked and needs its whole stdlib on disk,
so the floor is **distroless**
(`gcr.io/distroless/python3-debian12`, no pip/shell/apt)
or, more forgivingly, `python:3.13-slim`.

**uv — the official, fastest pattern**
(bind-mount the two manifests so the deps layer keys only on them;
install the project separately):

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.13-slim AS builder
COPY --from=ghcr.io/astral-sh/uv:0.11 /uv /uvx /bin/
ENV UV_COMPILE_BYTECODE=1 UV_LINK_MODE=copy UV_PYTHON_DOWNLOADS=0
WORKDIR /app
RUN --mount=type=cache,target=/root/.cache/uv \
    --mount=type=bind,source=uv.lock,target=uv.lock \
    --mount=type=bind,source=pyproject.toml,target=pyproject.toml \
    uv sync --locked --no-install-project --no-dev --no-editable
COPY . /app
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --locked --no-dev --no-editable

FROM python:3.13-slim AS runtime
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1 PATH="/app/.venv/bin:$PATH"
RUN groupadd --gid 10001 appuser \
 && useradd --uid 10001 --gid 10001 --create-home appuser
WORKDIR /app
COPY --from=builder --chown=appuser:appuser /app/.venv /app/.venv
USER 10001:10001
CMD ["/app/.venv/bin/myapp"]
```

- **`--locked`** fails on a stale lockfile
  (use `--frozen` only for the workspace `--no-install-workspace` case).
  `UV_LINK_MODE=copy` because the cache mount and the venv
  are on different filesystems (hardlink would warn);
  `UV_COMPILE_BYTECODE=1` precompiles `.pyc`;
  `UV_PYTHON_DOWNLOADS=0` uses the base image's Python.

**pip** —
same layer order;
**with a cache mount, omit `--no-cache-dir`**
(you *want* the cache, and it lives on the mount, not the layer);
**without** a cache mount, use `pip install --no-cache-dir`:

```dockerfile
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip pip install -r requirements.txt
```

**poetry** —
`POETRY_VIRTUALENVS_IN_PROJECT=true` so the venv is a copyable `./.venv`;
`poetry install --no-root --only main --no-directory` before source,
then `poetry install --only main` after.

**Gotchas.**
A venv is **not relocatable across interpreter paths**:
the official images put Python at `/usr/local/bin/python`
but **distroless at `/usr/bin/python`** —
a venv built in `python:3.13-slim` breaks on distroless
unless you `ln -s /usr/local/bin/python /usr/bin/python` before creating it,
and the Python minor version + libc must match exactly
(a 3.12 venv won't run on 3.13; glibc wheels won't load on musl).
**Prefer `-slim` over `-alpine`**:
PyPI wheels are glibc `manylinux`,
so musl compiles from source (much slower).
This coupling is why `python:3.13-slim`
is often the safer final base than distroless.

## Java / JVM

Cache the dependency download
(`/root/.m2` for Maven, `/root/.gradle` for Gradle),
then use **Spring Boot layered jars**
so dependency layers cache apart from app code,
or **jlink** to build a minimal JRE.

```dockerfile
# syntax=docker/dockerfile:1
FROM eclipse-temurin:21-jdk-jammy AS builder
WORKDIR /build
COPY --chmod=0755 mvnw mvnw
COPY .mvn/ .mvn/
COPY pom.xml pom.xml
RUN --mount=type=cache,target=/root/.m2 ./mvnw -B dependency:go-offline -DskipTests
COPY src/ src/
RUN --mount=type=cache,target=/root/.m2 ./mvnw -B clean package -DskipTests \
    && cp target/*.jar app.jar
# Spring Boot 3.3+ tools jarmode; --layers writes the four layer dirs.
RUN java -Djarmode=tools -jar app.jar extract --layers --destination extracted

FROM eclipse-temurin:21-jre-jammy AS final
WORKDIR /application
RUN groupadd --gid 10001 appuser \
 && useradd --uid 10001 --gid 10001 --shell /usr/sbin/nologin --no-create-home appuser
USER 10001:10001
# One COPY per layer, least-changing first, so a code change re-pushes only the
# tiny application/ layer.
COPY --from=builder /build/extracted/dependencies/ ./
COPY --from=builder /build/extracted/spring-boot-loader/ ./
COPY --from=builder /build/extracted/snapshot-dependencies/ ./
COPY --from=builder /build/extracted/application/ ./
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "application.jar"]
```

- `dependency:go-offline` is best-effort
  (Maven MDEP-82 misses some artifacts) —
  the `/root/.m2` cache mount is the reliable mechanism.
  Gradle equivalent:
  cache `/root/.gradle`, `./gradlew --no-daemon bootJar`.
- **jlink** shrinks the runtime itself:
  `jdeps --print-module-deps` →
  `jlink --add-modules "$modules,jdk.crypto.ec" --strip-debug --no-man-pages
  --no-header-files --compress zip-6 --output /javaruntime`
  (on JDK ≤ 20 use `--compress=2`; `zip-6` is JDK 21+).
  Ship the custom JRE on `gcr.io/distroless/java-base-debian12`
  (~40–70 MB vs a ~300 MB JDK).
  The JVM can't go to scratch — `libjvm.so` is dynamically linked.
- Dockerfile-free options:
  `mvn spring-boot:build-image` (buildpacks)
  or Google Jib (`mvn compile jib:build`, daemonless).

## .NET — the one interpreted-family case that nears scratch

Copy `*.csproj` first and `dotnet restore`,
then source and `dotnet publish`.
**The NuGet cache mount must appear on both the restore and publish `RUN`s**
(or combine them),
or `--no-restore` fails "package not found."
Chiseled Ubuntu images (`-noble-chiseled`) are distroless-like
and **non-root by default** (user `app`, `$APP_UID` 1654);
default HTTP port is 8080.

```dockerfile
# syntax=docker/dockerfile:1
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY ["MyApp.csproj", "./"]
RUN --mount=type=cache,id=nuget,target=/root/.nuget/packages dotnet restore "MyApp.csproj"
COPY . .
RUN --mount=type=cache,id=nuget,target=/root/.nuget/packages \
    dotnet publish "MyApp.csproj" -c Release -o /app --no-restore

FROM mcr.microsoft.com/dotnet/aspnet:10.0-noble-chiseled
WORKDIR /app
EXPOSE 8080
COPY --from=build /app .
USER $APP_UID
ENTRYPOINT ["./MyApp"]
```

- **NativeAOT** is the nearest-to-scratch case:
  `-p:PublishAot=true` emits a self-contained native binary
  that lands on `runtime-deps:*-chiseled` (~15–30 MB total).
  On .NET 8/9 install `clang zlib1g-dev` in the SDK stage
  and deploy on `runtime-deps:9.0-noble-chiseled-aot`;
  **.NET 10 dropped the `chiseled-aot` tag** —
  use `sdk:10.0-noble-aot` (clang bundled)
  + plain `10.0-noble-chiseled`.
  AOT forbids `Reflection.Emit` / dynamic assembly loading.
- Version status (2026-07):
  **.NET 8 LTS** (to 2026-11),
  **.NET 9 EOL** (2026-05),
  **.NET 10 LTS** (to 2028-11).
  Chiseled defaults to non-root, but the example sets `USER $APP_UID` (=1654)
  explicitly — the always-set-`USER` baseline;
  a **non**-chiseled base makes it mandatory (those default to **root**).
  Dockerfile-free: `dotnet publish /t:PublishContainer`.

## Ruby, PHP, Elixir

The framework generators
(`rails new`, `mix phx.gen.release --docker`)
already emit best-practice Dockerfiles that cache via layer ordering.
The essentials:

**Ruby / Rails** —
`Gemfile Gemfile.lock` → `bundle install` before source;
`BUNDLE_DEPLOYMENT=1`, `BUNDLE_WITHOUT="development:test"`;
split build-only `libpq-dev` (headers) from runtime `libpq5` (shared lib);
precompile assets with `SECRET_KEY_BASE_DUMMY=1`
so no real secret enters a layer;
final base `ruby:3.4-slim`, non-root `USER 10001:10001`.

**PHP** —
share one `base` stage (php-fpm-alpine + extensions via
`install-php-extensions`) between build and final
so Composer sees the same extensions at runtime.
`composer.json composer.lock` →
`composer install --no-dev --no-scripts --no-autoloader` before source,
then `composer dump-autoload --no-dev --optimize --classmap-authoritative`
over full source.
`USER 82:82` (Alpine's `www-data`), `EXPOSE 9000`.

**Elixir** —
`mix deps.get --only prod` before source,
`MIX_ENV=prod`,
then `mix release` (bundles ERTS, so the runner needs no Erlang/Elixir).
Not scratch:
the BEAM dynamically links openssl, libncurses, libstdc++ —
run `mix release` output on `debian:bookworm-slim`
with `libstdc++6 openssl libncurses6 ca-certificates` + a UTF-8 locale,
non-root `USER 10001:10001`.
Phoenix recommends Debian over Alpine (musl DNS / NIF issues).

---

# Smallest realistic final base, per ecosystem

| Ecosystem | Smallest realistic base | Reaches scratch? | What sets the floor |
|-----------|-------------------------|:----------------:|---------------------|
| **Go / Rust / C / Zig** | `scratch` (or `distroless/static:nonroot`) | **Yes** | static binary needs nothing |
| **.NET NativeAOT** | `runtime-deps:*-chiseled(-aot)` | **Nearest** | native binary, only libc/zlib remain |
| **.NET (framework-dep)** | `aspnet:*-noble-chiseled` | No | needs the .NET shared runtime + libc |
| **Java / JVM** | jlink JRE on `distroless/java-base` | No | `libjvm.so` + JRE modules, dynamically linked |
| **Node.js** | `distroless/nodejs22:nonroot` | No | V8 + libuv dynamically linked; built-in modules |
| **Python** | `distroless/python3` (else `python:3.x-slim`) | No | CPython dynamic + whole stdlib on disk |
| **Ruby** | `ruby:3.x-slim` | No | MRI dynamic (libyaml/openssl/libffi) |
| **PHP** | `php:8.x-fpm-alpine` / `-fpm` | No | php-fpm + extensions dynamic |
| **Elixir** | `mix release` on `debian:*-slim` | No | ERTS bundled but VM links openssl/ncurses/libstdc++ |

Ordering everywhere:
**`scratch` → `distroless/static` (no libc, static binaries only) →
`distroless/base|cc|<lang>` or `chiseled` (smallest realistic) →
`slim` (pragmatic default) →
`alpine` (smallest OS, musl caveats) → full image.**

---

# Debugging a scratch / distroless image (no shell)

You don't add a shell to production —
you attach one temporarily:

- **`:debug` distroless tags** bundle a busybox shell —
  swap the tag, `docker exec` in, swap back.
  Zero code change.
- **`docker debug <container>`** (Docker Desktop)
  side-loads a toolbox without modifying the image;
  **`cdebug exec -it <container>`** (open-source)
  does the same on plain Docker.
- **Kubernetes:**
  `kubectl debug -it <pod> --image=busybox --target=<container>`
  attaches an ephemeral container sharing the target's namespaces
  (stable since k8s 1.25) — no pod restart.
