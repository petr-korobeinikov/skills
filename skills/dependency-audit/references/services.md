# Service reference — vulnerability databases, Socket, Snyk

Exact syntax and URLs for Stages 1–3 of the `dependency-audit` skill.
Most read-only stages have a no-install web option,
so a missing CLI rarely blocks the audit.
The exception is Socket:
`socket.dev` serves browsers only (403 to `curl` and other agents),
so an automated run needs the Socket CLI (with a token)
or escalates that stage to a human.

## Ecosystem names differ per service (footgun)

The same ecosystem is spelled differently by each tool.
Get this wrong and the lookup silently returns nothing.

| Ecosystem | OSV (osv-scanner / OSV.dev) | GitHub Advisories | Socket web slug | Socket CLI / PURL | Snyk web   |
|-----------|-----------------------------|-------------------|-----------------|-------------------|------------|
| npm       | `npm`                       | `npm`             | `npm`           | `npm`             | `npm`      |
| Python    | `PyPI`                      | `pip`             | `pypi`          | `pypi`            | `pip`      |
| Go        | `Go`                        | `go`              | `go`            | `golang`          | `golang`   |
| Rust      | `crates.io`                 | `rust`            | `cargo`         | `cargo`           | unsupported|
| Maven/JVM | `Maven`                     | `maven`           | `maven`         | `maven`           | `maven`    |
| .NET      | `NuGet`                     | `nuget`           | `nuget`         | `nuget`           | `nuget`    |
| Ruby      | `RubyGems`                  | `rubygems`        | `gem`           | `gem`             | —          |
| PHP       | `Packagist`                 | `composer`        | `composer`      | `composer`        | `composer` |
| Dart      | `Pub`                       | `pub`             | —               | —                 | —          |
| Elixir    | `Hex`                       | `erlang`          | —               | —                 | —          |

- OSV strings are **case-sensitive** (`PyPI`, `Go`, `crates.io`).
- **Go is the worst offender:**
  Socket's *website* slug is `go`
  but its *CLI/PURL* token is `golang`.
- Maven/JVM packages are addressed as `<groupId>:<artifactId>`.

## Stage 0 — identity & metadata (no install)

Confirm a package is what you think it is before scanning it —
none of this installs anything.

- **deps.dev** (Google) — versions, licenses, advisories, dependency
  graph, OpenSSF Scorecard.
  Web `https://deps.dev/<system>/<name>`;
  API `https://api.deps.dev/v3/systems/<system>/packages/<name>`.
  `<system>` is deps.dev's own token —
  `npm` `pypi` `go` `maven` `cargo` `nuget`
  (case-insensitive, but **not** the OSV strings above:
  it wants `cargo` / `go`, not `crates.io` / `Go`).
  It does **not** cover Dart, Elixir, or other smaller ecosystems —
  there, use the native registry (`pub.dev`, `hex.pm`, …).
- Registry-native metadata, no install:
  - npm: `npm view <pkg>`
    (maintainers, `time`, `dist-tags`, repository, license)
  - Python: `https://pypi.org/pypi/<pkg>/json`
  - Go: `go list -m -versions <module>`; `https://pkg.go.dev/<module>`
  - Rust: `https://deps.dev/cargo/<crate>`
    (crates.io's own API rejects plain `curl`)
  - Maven / JVM: `https://deps.dev/maven/<group>:<artifact>`

What you're checking:
the real repository and homepage,
download / usage counts,
release cadence and last-publish date,
the maintainer list and any recent ownership change,
whether the version is yanked or deprecated,
and a license compatible with your project.

## Stage 1 — known vulnerabilities (CVE / OSV)

### osv-scanner (cross-ecosystem, off the lockfile)

Install: `brew install osv-scanner`, or
`go install github.com/google/osv-scanner/v2/cmd/osv-scanner@latest`
(note the **`/v2/`** in the path).

```sh
osv-scanner scan source -r <dir>            # whole project, recursive
osv-scanner scan source -L <lockfile>       # one lockfile (repeatable)
osv-scanner scan image <image>:<tag>        # a built container image
osv-scanner scan source -L <sbom.spdx.json> # an SBOM (detected by filename)
osv-scanner scan source --format json -L <lockfile>
```

Supported lockfiles include
`package-lock.json` / `yarn.lock` / `pnpm-lock.yaml` / `bun.lock`,
`go.mod`,
`requirements.txt` / `poetry.lock` / `Pipfile.lock` / `uv.lock`,
`Cargo.lock`,
`pom.xml` (transitive resolution on by default) / `gradle.lockfile`,
`composer.lock`, `Gemfile.lock`, `packages.lock.json`,
and — beyond the tabled ecosystems —
`pubspec.lock` (Dart), `mix.lock` (Elixir),
`conan.lock` (C/C++), `renv.lock` (R).
This list is **not** exhaustive:
it is the cross-ecosystem backstop for the
"when the ecosystem isn't listed" path in `SKILL.md` —
if a package manager writes a lockfile,
osv-scanner most likely reads it.

This is a **v2** tool — the v1 CLI was restructured:

- `osv-scanner --lockfile=<p>` → `osv-scanner scan source -L <p>`
  (the old bare forms still work, aliased to `scan source`).
- `osv-scanner --docker <img>` → `osv-scanner scan image <img>`.
- `--json` was removed → use `--format json`.

### OSV.dev — query one package, no install

```sh
curl -X POST -d \
  '{"package":{"name":"jinja2","ecosystem":"PyPI"},"version":"2.4.1"}' \
  https://api.osv.dev/v1/query
```

Ecosystem is **case-sensitive** and drawn from a fixed list
(the table above, plus the OSV schema —
`PyPI` `npm` `Go` `crates.io` `Maven` `NuGet` `RubyGems` `Packagist`
`Pub` `Hex` `GitHub Actions`, Linux distros, …).
A wrong string returns **HTTP 400 `Invalid ecosystem`**,
and a *plausible-but-wrong* one
(`Dart` / `Elixir` instead of `Pub` / `Hex`) fails the same way —
**never read a 400, or an empty hit from a bad slug, as
"no vulnerabilities".** Fix the slug, then trust the result.
Browser: `https://osv.dev/list?q=<name>&ecosystem=<ECOSYSTEM>`,
single record `https://osv.dev/vulnerability/<ID>`.

### GitHub Advisory Database — browse, no install

`https://github.com/advisories`, filtered via the `query` param
(`:` URL-encodes to `%3A`):

- ecosystem: `?query=ecosystem%3Anpm`
- **package: `affects:<name>`** → `?query=affects%3Alodash`
  (the qualifier is `affects:`, **not** `package:`)
- severity: `?query=severity%3Ahigh`
- type: `?query=type%3Amalware` (`reviewed` / `unreviewed` / `malware`)

### Native auditors

Per-ecosystem (`npm audit`, `pip-audit`, `govulncheck`,
`cargo audit`, OWASP dependency-check):
see [`ecosystems.md`](ecosystems.md).

## Stage 2 — Socket (supply-chain behaviour)

CVE databases know reported vulnerabilities;
Socket knows what the package **does**.

### CLI

Package `socket` (legacy alias `@socketsecurity/cli`).
Run `npx socket@latest <cmd>` or `npm install -g socket`.

```sh
# Deep score — reflects the package AND its transitive deps.
# "When you want to know whether to trust a package, run this."
socket package score <ecosystem> <name[@version]>   # --json / --markdown
socket package score npm express
socket package score pypi requests@2.31.0
socket package score pkg:golang/github.com/foo/bar@v1.2.3   # PURL form

# Shallow — the package itself only, no transitive.
socket package shallow <ecosystem> <name...>

# Whole-manifest scan (Go, Gradle, JS, Kotlin, Python, Scala).
socket scan create [TARGET...]      # --reach adds reachability analysis

# Install-time gating — runs the install through Socket and blocks
# risky packages before they land (npm, pip, pnpm, yarn, cargo, go, …).
socket npm install <pkg>
```

**Auth:** `package score` / `shallow` / `scan create` require a token —
`socket login` (interactive) or `SOCKET_CLI_API_TOKEN` (CI).
A deep score costs 100 quota units per call;
the token is created in the Socket dashboard.
Use the CLI for programmatic vetting —
`socket.dev` returns 403 to non-browser requests, so don't scrape it.

### Web (free, public, no login)

```
https://socket.dev/<ecosystem>/package/<name>
https://socket.dev/<ecosystem>/package/<name>/overview/<version>
```

Slugs are the PURL types in the table above
(remember: website `go`, but CLI `golang`; Rust is `cargo`).

### What Socket flags (≈90 alert types)

Categories: **Supply Chain Risk, Quality, Maintenance, Vulnerability,
License.** The behavioural alerts a CVE DB can't give you:

- **Execution:** `installScripts` ("most npm malware hides here"),
  `binScriptConfusion`, `shellScriptOverride`.
- **Capabilities:** `networkAccess`, `filesystemAccess`, `shellAccess`,
  `envVars`, `usesEval`, `dynamicRequire`, `hasNativeCode`.
- **Obfuscation:** `obfuscatedFile`, `minifiedFile`, `highEntropyStrings`,
  `bidi` (Trojan Source), `homoglyphs`, `invisibleChars`.
- **Identity:** `newAuthor`, `unstableOwnership`, `missingAuthor`,
  `compromisedSSHKey`, `suspiciousStarActivity`.
- **Typosquat / malware:** `didYouMean`, `gptDidYouMean`, `malware`,
  `manifestConfusion`, `troll` (protestware).
- **Telemetry & integrity:** `telemetry`, `gitDependency` /
  `httpDependency` (non-immutable installs), `chronoAnomaly`.
- **Quality:** `deprecated`, `unmaintained`, `trivialPackage`,
  `unpopularPackage`.
- It also carries CVEs (`criticalCVE`, `cve`)
  and license alerts (`copyleftLicense`, `nonpermissiveLicense`).

Treat behavioural alerts that don't match the package's stated purpose
(a string helper with `networkAccess` + `installScripts`)
as **blocking until explained**.

## Stage 3 — Snyk (known vulns + health, second opinion)

### CLI (needs a free account)

Install `npm install -g snyk`; authenticate with `snyk auth`
(browser OAuth) or `SNYK_TOKEN` (CI).

```sh
snyk test <pkg>            # latest — NPM ONLY
snyk test <pkg>@<version>  # specific — NPM ONLY (e.g. snyk test ionic@1.6.5)
```

For **every other ecosystem there is no single-package form** —
create a throwaway manifest and run `snyk test` in that directory:

- Python: a `requirements.txt` pinning the candidate → `snyk test`
  (non-standard filename needs `--file=… --package-manager=pip`).
- Go: a `go.mod` requiring it → `snyk test` (no build needed).
- Maven: a `pom.xml` → `mvn install` → `snyk test`.
- **Rust / Cargo is not supported by `snyk test`** —
  use `snyk sbom test --file=<sbom>` or the web page below.

Reports known vulns by severity, with CVSS / CVE / CWE,
the vulnerable dependency paths (`--show-vulnerable-paths`),
fix / upgrade paths, and license issues.
`--json`, `--severity-threshold=<low|medium|high|critical>`, `--fail-on`.
Exit codes: `0` clean, `1` vulns found, `2` error, `3` no supported project.

### Web (free, public, no login)

```
https://security.snyk.io/package/<ecosystem>/<name>
```

e.g. `/npm/express`, `/pip/flask`,
`/golang/github.com%2Fgin-gonic%2Fgin` (slashes URL-encoded),
`/maven/org.apache.logging.log4j:log4j-core`.
Shows the **Package Health Score**
(security / popularity / maintenance / community)
alongside known vulnerabilities.
The old `snyk.io/advisor/...` URLs now redirect here.

Snyk is **known CVEs + licenses + a health score** —
not behavioural supply-chain analysis.
That's Socket's job (Stage 2);
the two are complementary, not redundant.
