# Per-ecosystem commands

Operational reference for Stage 1 (known-vuln auditors)
and Stage 4 (sandbox install + transitive tree)
of the `dependency-audit` skill.

Drop each ecosystem's **image** and **inner command**
into the hardened base invocation from `SKILL.md`:

```sh
docker run --rm \
  --network bridge \          # recon needs the registry; Pass B (SKILL.md) cuts egress
  --cap-drop ALL \
  --security-opt no-new-privileges \
  --pids-limit 512 \
  --memory 1g \
  <image> sh -c '<inner command>'
```

The commands below were exercised in exactly this sandbox.
Example output is trimmed to the shape, not the full tree.

| Ecosystem            | Sandbox image                  | Transitive tree (recon)         | Native known-vuln auditor      |
|----------------------|--------------------------------|---------------------------------|--------------------------------|
| npm / yarn / pnpm    | `node:lts-alpine`              | `npm ls --all`                  | `npm audit`                    |
| Go                   | `golang:alpine`                | `go mod graph`                  | `govulncheck ./...`            |
| Python               | `python:3-slim`                | `pip install --dry-run --report`| `pip-audit`                    |
| Rust                 | `rust:slim`                    | `cargo tree`                    | `cargo audit`                  |
| JVM (Kotlin / Java)  | `maven:3-eclipse-temurin-21`   | `mvn dependency:tree`           | OWASP dependency-check         |
| Ruby                 | `ruby:slim`                    | `bundle list` / `gem dependency`| `bundler-audit`                |
| PHP                  | `composer:latest`              | `composer show --tree`          | `composer audit`               |
| .NET                 | `mcr.microsoft.com/dotnet/sdk` | `dotnet list package --include-transitive` | `dotnet list package --vulnerable --include-transitive` |

The first five are validated below.
The last three are the standard documented forms —
verify locally before relying on the exact flags.

## npm / yarn / pnpm

**Recon (no install scripts):**
`--ignore-scripts` skips every pre/post-install hook,
so nothing the package ships executes during resolution.

```sh
node:lts-alpine sh -c \
  'cd "$(mktemp -d)" && npm init -y >/dev/null \
     && npm install --ignore-scripts --no-audit --no-fund <pkg> \
     && npm ls --all'
```

```
tmp@1.0.0 /tmp/tmp.XXXX
`-- is-odd@3.0.1
  `-- is-number@6.0.0
```

`npm ls --all --json` for a machine-readable tree.

**Known vulns:** `npm audit` (reads `package-lock.json`;
add `--omit=dev` for prod-only,
`--audit-level=high` to gate,
`--json` for output).
It is **always online** —
it POSTs your dependency tree to the registry advisory endpoint;
there is no offline mode (`--no-audit` disables it).

## Go

Go runs **no install scripts** —
fetching a module never executes its code —
so resolution is inherently safer than npm/pip.
`go.mod` is the manifest; the build graph is the tree.

```sh
golang:alpine sh -c \
  'cd "$(mktemp -d)" && go mod init sandbox >/dev/null \
     && go get <module>@latest \
     && go mod graph'
```

```
sandbox github.com/spf13/cobra@v1.10.2
github.com/spf13/cobra@v1.10.2 github.com/spf13/pflag@v1.0.9
github.com/spf13/cobra@v1.10.2 github.com/inconshreveable/mousetrap@v1.1.0
github.com/spf13/cobra@v1.10.2 github.com/cpuguy83/go-md2man/v2@v2.0.6
...
```

**Known vulns:** `govulncheck ./...`
(`go install golang.org/x/vuln/cmd/govulncheck@latest`).
It is **reachability-aware** —
it reports only vulnerabilities your code transitively *calls*,
not every vulnerable module in `go.mod`,
so its findings are higher-signal than a plain lockfile scan.

## Python

pip resolves an sdist by running its `setup.py` —
there is no clean `--ignore-scripts`.
`--dry-run --report` resolves the full set without *installing* it,
and runs no code **when a wheel is available** (the common case);
an sdist-only dependency still executes `setup.py` to produce metadata,
so there the container isolation — not the flag — is your protection:

```sh
python:3-slim sh -c \
  'pip install --dry-run --report - --root-user-action=ignore <pkg>'
```

```
resolved: Flask==3.1.3, Jinja2==3.1.6, MarkupSafe==3.0.3,
          Werkzeug==3.1.8, blinker==1.9.0, click==8.4.2,
          itsdangerous==2.2.0
```

(The report is JSON on stdout; the names live under `install[].metadata`.)
To actually install and walk the tree —
fine inside the sandbox, where any `setup.py` execution is contained:

```sh
python:3-slim sh -c \
  'pip install --quiet --root-user-action=ignore <pkg> pipdeptree \
     && pipdeptree -p <pkg>'
```

**Known vulns:** `pip-audit` (PyPA; Python 3.10+):
`pip-audit -r requirements.txt`,
or `pip-audit .` for a project,
or `pip-audit` for the current environment.
`-s osv` switches the data source to OSV;
`-f json` for output.

## Rust

`cargo add` edits `Cargo.toml` and resolves,
but does **not** build —
crate `build.rs` scripts run on `cargo build`, not on `add` —
so `cargo add` + `cargo tree` is a no-execution recon.

```sh
rust:slim sh -c \
  'cd "$(mktemp -d)" && cargo init -q --name sandbox . \
     && cargo add <crate> && cargo tree'
```

```
sandbox v0.1.0
`-- rand v0.10.1
    |-- chacha20 v0.10.1
    |   |-- cfg-if v1.0.4
    |   `-- rand_core v0.10.1
    |-- getrandom v0.4.3
    ...
```

**Known vulns:** `cargo audit` (RustSec; `cargo install cargo-audit`),
run at the project root against `Cargo.lock`.
It is **offline-capable** (`-n` / `--stale`)
once the advisory DB is cached —
unlike `npm audit` / `pip-audit`, which always hit a remote service.

## JVM (Kotlin / Java — Gradle & Maven)

Kotlin and Java share the Maven/Gradle resolver,
so the transitive tree looks the same whichever build tool the project uses.
Maven is pure-Java and boots reliably in the sandbox;
the `gradle` image's native services can fail to load under CPU emulation
(`libnative-platform.so` on `aarch64`) —
prefer Maven for a throwaway resolve,
or use the project's own Gradle when auditing in place.

```sh
maven:3-eclipse-temurin-21 sh -c \
  'd=$(mktemp -d); cd "$d"; \
   printf "<project><modelVersion>4.0.0</modelVersion>\
<groupId>s</groupId><artifactId>s</artifactId><version>1</version>\
<dependencies><dependency>\
<groupId>GROUP</groupId><artifactId>ARTIFACT</artifactId><version>VER</version>\
</dependency></dependencies></project>\n" > pom.xml; \
   mvn -B dependency:tree'
```

```
\- com.squareup.okhttp3:okhttp:jar:4.12.0:compile
   +- com.squareup.okio:okio:jar:3.6.0:compile
   |  \- com.squareup.okio:okio-jvm:jar:3.6.0:compile
   |     \- org.jetbrains.kotlin:kotlin-stdlib-common:jar:1.9.10:compile
   \- org.jetbrains.kotlin:kotlin-stdlib-jdk8:jar:1.8.21:compile
      ...
```

In a Gradle project, the equivalent tree command is
`gradle <module>:dependencies --configuration runtimeClasspath`.

**Known vulns:** `gradle dependencies` / `mvn dependency:tree`
are **trees, not scanners** —
they carry no CVE data.
To actually vuln-scan the JVM tree, use one of:

- **OWASP dependency-check** —
  `dependency-check.sh --scan <path> --format HTML`.
  It needs an NVD API key (`--nvdApiKey <key>`,
  requested at <https://nvd.nist.gov/developers/request-an-api-key>)
  or updates are extremely slow.
  Gradle plugin: `org.owasp.dependencycheck` → `gradle dependencyCheckAnalyze`.
  Maven: `mvn org.owasp:dependency-check-maven:check`.
- **`osv-scanner`** against `pom.xml` (transitive resolution on by default)
  or a `gradle.lockfile` — see `references/services.md`.

## Other ecosystems

Same shape — disposable image, no host mount, resolve the tree, scan for vulns:

- **Ruby:** `ruby:slim`;
  tree via `bundle list` / `gem dependency <gem>`;
  vulns via `bundler-audit` (`gem install bundler-audit`)
  or `osv-scanner` on `Gemfile.lock`.
- **PHP:** `composer:latest`;
  tree via `composer show --tree`;
  vulns via `composer audit`.
- **.NET:** `mcr.microsoft.com/dotnet/sdk`;
  tree via `dotnet list package --include-transitive`;
  vulns via `dotnet list package --vulnerable --include-transitive`.
- **Dart / Flutter:** `dart:stable`;
  tree via `dart pub get && dart pub deps`;
  vulns via `osv-scanner` on `pubspec.lock` (OSV ecosystem `Pub`).
- **Elixir:** `elixir:slim`;
  tree via `mix deps.get && mix deps.tree`;
  vulns via `mix_audit` or `osv-scanner` on `mix.lock`
  (OSV ecosystem `Hex`; GitHub Advisories labels it `erlang`).

For anything not listed,
`osv-scanner` covers the known-vuln stage off the ecosystem's lockfile
(supported lockfiles in `references/services.md`),
and the sandbox method is identical:
a clean container, no host mount, install, enumerate, observe.
For identity, fall back to the native registry,
and resolve each service's ecosystem slug before trusting a clean result —
see "When the ecosystem isn't listed" in `SKILL.md`.
