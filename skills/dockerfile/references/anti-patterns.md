# Dockerfile anti-patterns → fixes

The bad-practice catalog for the `dockerfile` skill:
each entry is the bad form, the fix, why it matters,
and the linter rule that flags it —
`DL####` / `SC####` from **hadolint**,
`AVD-DS-####` from **Trivy** config scanning,
`DF###` from **droast**.
Run the linters (`SKILL.md` §6);
this is how to read their output and,
when composing, what to avoid up front.

## High-value hadolint rules

| Rule | Sev | Meaning |
|------|-----|---------|
| DL3006 | warn | Tag the base image explicitly (bare `FROM image`) |
| DL3007 | warn | Don't use `:latest` — pin a release tag |
| DL3008 | warn | Pin versions in `apt-get install` (`pkg=version`) |
| DL3009 | info | Delete apt lists after install (`rm -rf /var/lib/apt/lists/*`) |
| DL3015 | info | Use `--no-install-recommends` |
| DL3013 / DL3016 / DL3018 / DL3028 | warn | Pin versions in pip / npm / apk / gem |
| DL3042 | warn | `pip install --no-cache-dir` |
| DL3019 | info | `apk add --no-cache` |
| DL3020 | error | Use `COPY`, not `ADD`, for files/dirs |
| DL3003 | warn | Use `WORKDIR`, not `RUN cd` |
| DL3025 | warn | Use JSON (exec) form for `CMD`/`ENTRYPOINT` |
| DL4006 | warn | Set `SHELL … -o pipefail` before a piped `RUN` |
| DL3059 | info | Consecutive `RUN`s — consider consolidating |
| DL3002 | warn | Last `USER` must not be root |
| DL3004 | error | Don't use `sudo` |
| DL3066 | info | `USER` id should be numeric (host may not resolve names) |
| DL3026 | error | `FROM` only from an allowed registry (`trustedRegistries`) |
| DL4000 | error | `MAINTAINER` is deprecated → `LABEL` |
| DL4003 / DL4004 | warn/err | Multiple `CMD` / multiple `ENTRYPOINT` |

Config: `.hadolint.yaml`
(`failure-threshold`, `ignored: [DL3008]`,
`trustedRegistries: [docker.io]`, per-rule `override:`).
Inline: `# hadolint ignore=DL3008,SC2086`.
Every ignore should name why.

## The catalog

### 1. Unpinned / `:latest` base — DL3006, DL3007, AVD-DS-0001

```dockerfile
# bad
FROM node:latest
# good — pin the tag; pin the digest for full integrity
FROM node:22.11-bookworm-slim@sha256:a8560b36e8b8...df3ef88c
```

Mutable tags make builds non-reproducible
and silently pull breaking or vulnerable images;
a digest freezes the exact bytes.
Trade-off: pinning opts out of automatic security updates —
pair it with digest-refresh automation (Scout, Renovate, Dependabot).

### 2. `apt-get update` in a separate RUN from install — AVD-DS-0017

```dockerfile
# bad — the update layer caches forever, later pairing a stale index with install
RUN apt-get update
RUN apt-get install -y curl
# good — one RUN, so a changed install list re-runs update too ("cache busting")
RUN apt-get update && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*
```

This is the single most common Dockerfile bug.
Docker keys `RUN` on the command string only,
so a standalone `apt-get update` layer is reused indefinitely.

### 3. Not cleaning package caches — DL3009 (apt), DL3019 (apk)

```dockerfile
# good (Debian): clean in the SAME run — a later rm only adds a layer, the bloat stays
RUN apt-get update && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*
# good (Alpine)
RUN apk add --no-cache curl
```

Layers are additive:
bytes written in one layer and deleted in a later one still ship.
(BuildKit cache mounts sidestep this —
content under `--mount=type=cache` is never committed; see `SKILL.md` §2.)

### 4. `ADD` where `COPY` suffices — DL3020

```dockerfile
# bad
ADD . /app
# good
COPY . /app
```

`COPY` is predictable local-only copy.
Reserve `ADD` for its extra features —
remote fetch with `ADD --checksum=sha256:… <url>` (pin it),
Git repos, or local tar auto-extraction.
(Modern Docker prefers `ADD --checksum` over `wget && tar`
for remote artifacts, reversing older advice —
but never `ADD` a bare URL unpinned.)

### 5. `COPY . .` before installing deps — busts the dependency layer

```dockerfile
# bad — any source edit re-runs the expensive install
COPY . .
RUN npm install
# good — manifest first (SKILL.md §2)
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
```

### 6. No `.dockerignore` / huge context

```gitignore
.git
node_modules
.venv
target
dist
*.log
.env
```

The whole context is sent to the builder —
an un-ignored `.env`/`.git` leaks into the image
and every churn-y file busts `COPY . .`.

### 7. Running as root — DL3002, AVD-DS-0002

```dockerfile
# good — create an unprivileged user and switch, numeric UID
RUN groupadd --gid 10001 app \
 && useradd --uid 10001 --gid 10001 app
USER 10001:10001
```

Container root is host-root-adjacent;
a breakout starts at max privilege.
Numeric UID is required for Kubernetes `runAsNonRoot`
(the kubelet validates from image metadata and can't resolve a username
→ `CreateContainerConfigError`).

### 8. Build toolchain in the final image — use multi-stage

```dockerfile
# good — SDK stays in the build stage; final carries only the artifact
FROM golang:1.25 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /app .

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /app /app
USER 65532:65532
ENTRYPOINT ["/app"]
```

### 9. Shell-form CMD/ENTRYPOINT breaks signals — DL3025

```dockerfile
# bad — runs under /bin/sh -c; the app isn't PID 1 and never sees SIGTERM
ENTRYPOINT node server.js
# good
ENTRYPOINT ["node", "server.js"]
```

`docker stop` then waits out the grace period and `SIGKILL`s
(no graceful shutdown).
Add `tini`/`dumb-init` — or `docker run --init` —
where the process must reap children.

### 10. `RUN cd` instead of `WORKDIR` — DL3003

```dockerfile
# bad — each RUN is a fresh shell; the cd doesn't persist
RUN cd /app && npm ci
# good
WORKDIR /app
RUN npm ci
```

### 11. Missing `--no-install-recommends` — DL3015

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*
```

Recommended-but-unneeded packages enlarge the image and attack surface.

### 12. Separate `chmod`/`chown` layer doubles a tree's size

```dockerfile
# bad — chown -R rewrites every file into a NEW layer; the original copy also persists
COPY node_modules /app/node_modules
RUN chown -R app:app /app/node_modules
# good — set ownership in the copy layer
COPY --chown=app:app node_modules /app/node_modules
```

(`COPY --chmod` is BuildKit-only —
omit it on the legacy builder; see `SKILL.md` §1.)

### 13. Piped RUN without `pipefail` — DL4006

```dockerfile
# sh -c only checks the last command's exit code, so a failed download is masked
SHELL ["/bin/bash", "-o", "pipefail", "-c"]
RUN wget -O - https://example.com/x | tar xz
```

On Alpine/busybox: `SHELL ["/bin/ash", "-eo", "pipefail", "-c"]`.

### 14. Unpinned OS/language package versions — DL3008/DL3013/DL3016/DL3018

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends curl=7.88.* \
    && rm -rf /var/lib/apt/lists/*
```

Pin `pkg=version` for reproducibility.
Caveat: pinned distro versions eventually age out of moving mirrors and 404 —
pair with regular rebuilds on a fresh base.

### 15. `sudo` in a container — DL3004

Do privileged setup while still root, then `USER` down;
use `gosu`/`su-exec` to de-escalate at runtime.
`sudo` has unpredictable TTY/signal behavior and is a privesc vector.

### 16. `apt-get upgrade` / `dist-upgrade`

Don't blanket-upgrade the base OS —
it defeats base pinning and reproducibility.
Install only what you need
and rebuild on a fresh pinned base (`docker build --pull`).

### 17. Secrets in `ARG` / `ENV`

They persist in `docker history`, image config, and `max`-mode provenance.
Use `RUN --mount=type=secret` (`SKILL.md` §4).
Audit with `docker history --no-trunc <image>`.

### 18. `MAINTAINER` — DL4000

```dockerfile
# bad
MAINTAINER Jane Doe <jane@example.com>
# good
LABEL org.opencontainers.image.authors="jane@example.com"
```

### 19. Missing `HEALTHCHECK` where it matters — DL3057, AVD-DS-0026

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=5s --retries=3 \
  CMD curl -fsS http://localhost:8080/health || exit 1
```

Without it, Docker knows the process is alive, not that it's serving.
On a no-shell (distroless) image,
use an exec-form probe binary instead of `curl`.

## Dockerfile-time vs runtime hardening

Set what the image can carry;
know what it can't.
The Dockerfile should **still set `USER`**
even though the strongest controls are runtime —
whoever runs the image may omit them,
so `USER` is the defense-in-depth baseline.

| In the Dockerfile | Runtime only (`docker run` / Compose / k8s `securityContext`) |
|-------------------|---------------------------------------------------------------|
| `USER 10001:10001` (numeric) | `--cap-drop ALL` then add only what's needed |
| `COPY --chown` ownership | `--security-opt no-new-privileges` |
| write only to tmpfs/volumes | `--read-only` (+ `--tmpfs /tmp`) |
| drop SUID/SGID bits | seccomp / AppArmor; rootless daemon |
| `HEALTHCHECK` | `-m 256m --pids-limit 512`; never `--privileged` |
