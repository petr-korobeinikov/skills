---
name: fd
description: >-
  Prefers `fd` (https://github.com/sharkdp/fd) over `find` for filesystem
  searches in shell snippets. `fd` is faster on large trees, has saner
  defaults (respects `.gitignore`, smart case, regex by default), and a
  more ergonomic CLI. Falls back to `find` when `fd` is not available
  on the host.
when_to_use: >-
  Use whenever about to search the filesystem from a shell — `Bash`
  tool calls, scripts, snippets in `SKILL.md` / `README.md`, Makefile
  recipes, CI configs, install instructions. Triggers: «найди файлы»,
  «найди в папке», «поиск файлов», «рекурсивный поиск», «список файлов
  по расширению», «удали все .X», «find a file», «search the
  filesystem», «list files matching», «recursive search», «find -name»,
  «find -type», «find -exec», `find .`, `find /`.
---

# fd

Prefer `fd` over `find` for filesystem searches.
`fd` is faster on large trees,
has saner defaults
(respects `.gitignore`, smart case, regex by default),
and a more ergonomic CLI.

Authoritative documentation:
https://github.com/sharkdp/fd
(README + `fd --help`).
Verify any non-trivial form against the manual before relying on it.

## Why

- Parallel directory traversal —
  noticeably faster than `find` on large repos.
- Sensible defaults out of the box:
  hidden entries skipped,
  `.gitignore` / `.ignore` / `.fdignore` honored when inside a git
  repo,
  smart case
  (case-insensitive unless the pattern has an uppercase letter),
  patterns are regex by default.
- Pattern matches against the filename by default,
  not the full path —
  `fd foo` does what `find . -iname '*foo*'` does,
  without the wildcards-and-quotes ritual.
- Parallel `-x` execution,
  null-separated output (`-0`),
  and a single set of options that compose without
  `-print`/`-prune` gymnastics.

## Rule

```sh
fd PATTERN [PATH]
```

The first positional argument is the pattern;
the second is the search root
(defaults to the current directory).
A path-shaped first argument (containing `/`) is **not** a path —
use `fd . PATH` or `fd --full-path PATTERN`.

## Direct substitutions

| Don't                                           | Do                              |
| ----------------------------------------------- | ------------------------------- |
| `find . -name 'foo*'`                           | `fd '^foo'`                     |
| `find . -iname '*.md'`                          | `fd -e md`                      |
| `find . -type f -name foo`                      | `fd -t f foo`                   |
| `find . -type d -name node_modules`             | `fd -t d node_modules`          |
| `find . -type l`                                | `fd -t l`                       |
| `find . -name '*.txt' -delete`                  | `fd -e txt -X rm`               |
| `find . -name '*.txt' -exec rm {} \;`           | `fd -e txt -x rm`               |
| `find . -name node_modules -prune`              | `fd -E node_modules`            |
| `find . -size +100M`                            | `fd -S +100mi`                  |
| `find . -mtime -7`                              | `fd --changed-within 7d`        |
| `find . -mtime +7`                              | `fd --changed-before 7d`        |
| `find . -name '*.go' -print0 \| xargs -0 wc -l` | `fd -e go -0 \| xargs -0 wc -l` |
| `find /etc -name passwd`                        | `fd passwd /etc`                |
| `find / -name foo`                              | `fd -u foo /`                   |

For an exact `find -size` match,
use the binary suffixes
(`ki`, `mi`, `gi`) —
`-S +100m` is `100 * 10^6` bytes,
`-S +100mi` is `100 * 2^20` bytes.

For `find /` searches that should also descend into hidden and
`.gitignore`-d directories,
add `-u` (alias for `--no-ignore --hidden`).

## Idioms

Match by extension (one or many):

```sh
fd -e md
fd -e h -e cpp
```

Glob mode for shell-style patterns
(`-g` so `**` and `*` mean the usual thing):

```sh
fd -g '**/*.test.ts'
fd -g 'libc.so' /usr
```

Match against the full path,
not just the filename:

```sh
fd --full-path 'src/.*\.proto$'
```

Include hidden entries
(`.git`, `.vscode`, etc.)
but keep `.gitignore` rules:

```sh
fd -H pre-commit
```

Disregard `.gitignore` rules but keep hidden entries hidden:

```sh
fd -I node_modules
```

Search everywhere — both hidden and ignored:

```sh
fd -u PATTERN
```

Exclude paths
(glob syntax, overrides every ignore rule, repeatable):

```sh
fd -E .git -E '*.bak' PATTERN
```

Limit depth:

```sh
fd -d 2 -t d
```

Combine multiple required patterns
(all must match the filename, regex by default):

```sh
fd 'foo' --and 'bar' --and 'baz'
```

Execute per result
(parallel, `{}` substitution, no `\;` terminator):

```sh
fd -e jpg -x convert {} {.}.png
```

Execute once with all results as arguments
(batch tools like `wc`, `clang-format`, `rg`):

```sh
fd -e rs -X wc -l
```

Pipe null-separated to other tools:

```sh
fd -e go -0 | xargs -0 wc -l
```

## Footguns

- **First positional argument is the pattern, not the path.**
  `fd /etc/passwd` interprets `/etc/passwd` as a regex with `/` in it
  and errors out
  with the suggestion to use `fd . /etc/passwd`
  or `--full-path`.
- **Default search excludes hidden and `.gitignore`d entries.**
  When looking for something inside `node_modules` or `.git`,
  add `-H -I` (or `-u`)
  or the result list will be empty for the wrong reason.
- **`--exec` parallelizes by default.**
  Output order is non-deterministic.
  Pass `-j 1` (or `--threads 1`) when serial execution is required.
- **Patterns are regex by default.**
  A bare `.` matches any character;
  for a literal `.foo` anchor it (`'\.foo$'`)
  or pass `-F`/`--fixed-strings`.
- **Size suffixes are decimal SI by default.**
  `-S +100m` is `100 MB` (`10^6` bytes per `m`);
  use `mi`/`ki`/`gi` for binary units when matching `find -size`
  semantics exactly.

## Fallback to `find`

`fd` is not always available —
minimal CI images,
locked-down servers,
container base images without package extras.
Probe with `command -v` and fall back to `find`:

```sh
if command -v fd >/dev/null; then
  fd -e log -t f -X rm
else
  find . -type f -name '*.log' -exec rm {} +
fi
```

For a one-shot snippet in a controlled local environment,
use `fd` directly.
Switch to `find` unconditionally only when the target host is known
not to have `fd`
(e.g. a minimal Alpine container without `fd` in the image).

Do **not** install `fd` automatically just to satisfy this skill —
the fallback is the correct response when it is missing.

## Installing `fd`

- macOS: `brew install fd`
- Debian/Ubuntu: `apt install fd-find` —
  the binary is named `fdfind`
  to avoid a name conflict
  (`alias fd=fdfind` is the standard workaround).
- Fedora: `dnf install fd-find`
- Arch: `pacman -S fd`
- Alpine: `apk add fd`
- Prebuilt binaries and other platforms:
  https://github.com/sharkdp/fd/releases

When mise manages the host's tools,
the registry shortname `fd` resolves to `aqua:sharkdp/fd` —
install with `mise use -g fd@<version>`
(pinned, per the project's `mise` skill).
Do **not** use the `ubi:` backend —
the `mise` skill disables it.

## Scope

Applies anywhere a shell snippet searches the filesystem:

- `Bash` tool invocations.
- Snippets in `SKILL.md` / `README.md` / docs.
- Scripts under `scripts/`.
- Makefile recipes.
- CI configs.
- Install instructions.

If you encounter a `find` invocation while editing a file,
replace it with the equivalent `fd` form
(plus the `find` fallback when the snippet must run on a host that
may not have `fd`)
as part of the same edit.
