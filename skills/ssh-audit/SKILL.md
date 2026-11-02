---
name: ssh-audit
description: >-
  Audit the crypto posture of an SSH server or client with jtesta/ssh-audit —
  key-exchange, host-key, cipher and MAC algorithms, plus non-crypto checks
  (the DHEat DoS, CVE-2002-20001, and Terrapin, CVE-2023-48795). Installs
  cleanly through the mise skill (pypi, isolated venv, zero dependencies,
  nothing on the target — the scan is remote). Reads the fail/warn/rec output,
  then turns findings into concrete sshd_config directives via the tool's
  built-in hardening guides — without prescribing how you deploy them.
when_to_use: >-
  Use when auditing or hardening SSH crypto, or interpreting a scan result.
  Triggers: «проверь ssh», «аудит ssh», «безопасен ли ssh», «захардь sshd»,
  «слабые алгоритмы ssh», «слабые шифры ssh», «какие KexAlgorithms оставить»,
  «Terrapin», «DHEat», «ssh-audit»; «audit ssh», «harden sshd», «is my ssh
  server secure», «weak ssh ciphers/kex/macs», «which KexAlgorithms should I
  keep», «check for Terrapin or DHEat», «ssh crypto posture». Also when a scan
  report needs interpreting or a finding turning into an sshd_config fix.
---

# ssh-audit

[ssh-audit](https://github.com/jtesta/ssh-audit)
reads what an SSH endpoint actually negotiates —
its key-exchange, host-key, cipher and MAC algorithms —
and grades each as info / warn / fail,
alongside non-crypto checks (the DHEat DoS and Terrapin).
It only *reports*.
This skill covers running it, reading it,
and turning a finding into a concrete `sshd_config` directive —
but it stays out of *how* you deploy that directive.
Whether the fix lands via a config-management role, a hand edit,
or anything else
is the host project's decision, not this skill's.

## When in doubt, ask the operator

If a scan result is ambiguous,
a tool is missing,
or a fix would trade away compatibility you can't confirm is safe —
stop and ask the operator instead of guessing.
Locking yourself out of a host you can only reach over SSH is expensive;
one question is cheaper.
This applies to every step below.

## Authorization

A scan is active traffic against a live service —
run it only against hosts you own or are explicitly cleared to test.
A default scan already includes a DHEat rate-test —
a burst of dozens of connections
(pass `--skip-rate-test` to suppress it).
And `--dheat` is a *real*, sustained DoS attack against the target
(`--conn-rate-test` floods connections to measure DHEat susceptibility) —
never point either at a host you don't control.

## Model

The scan runs from your machine against a remote endpoint —
nothing is installed on the target.
An SSH server exposes one global crypto config, not a per-interface one,
so reaching it by public IP or over a VPN / overlay address
hits the same `sshd` and yields the same verdict.

## Install

Check for it first — `command -v ssh-audit`.
If it's missing, install it with the mise skill;
`pypi:ssh-audit` is a mise tool spec (mise's pypi backend), not a shell command.
With the mise binary but no mise skill, add and pin it yourself —
uv first, because without it the pypi backend demands pipx:

```sh
# Resolve each version via `mise latest uv`, `mise latest python`, `mise latest pypi:ssh-audit`
mise use -g "uv@<uv-version>"
mise use -g "pypi:ssh-audit[python=<python-version>]@<version>"
```

ssh-audit is pure-stdlib Python,
so its venv holds that one wheel with zero dependencies,
on a private Python that stays off your `PATH`,
and nothing lands in system site-packages.
Never `pip install` it into site-packages, and never `brew` / curl-pipe it.
If there's no mise at all, say so and stop — don't reach for another installer.

## Running a scan

Server (the common case):

```sh
ssh-audit <host>            # default port 22
ssh-audit <host> -p 2222    # non-standard port
```

The default scan runs the DHEat rate-test — a burst of dozens of connections.
On a host you depend on SSH access to, or one running fail2ban / sshguard,
that burst can get your **source IP banned** (a self-lockout):
scan such hosts with `--skip-rate-test`, or allowlist your IP first.
Keep the default (with the rate-test) for the *first* scan when it's safe —
it's what produces the DHEat verdict; only re-scans routinely skip it.

Client audit — ssh-audit listens (default port 2222)
and grades the next SSH client that connects to it:

```sh
ssh-audit -c                       # listens on :2222
ssh -p 2222 user@<this-machine>    # from the client you want graded
```

The client connection won't complete —
ssh-audit only needs the handshake to read what the client offers.
(`-c` opens a transient listener on `:2222` on this machine.)

Flags worth knowing:

- `-t N` — connection/read timeout, default 5s.
  Raise it when an outbound firewall makes the first connection slow.
- `-n` — no colours; use it whenever you pipe or capture output.
- `--skip-rate-test` — skip the connection-flood DHEat probe
  (faster, and see **Re-scanning** below).
- `-l fail` / `-l warn` — raise the minimum level shown.
  Don't set `-l fail` for a real audit: it hides the very warnings
  this skill wants you to judge. (Lower-case `-l`; the upper-case `-L`
  lists policies — a different flag.)
- `--lookup <alg,…>` — explain specific algorithm names without scanning.
- `-p N`, `--socks5 host:port` — port and SOCKS proxy
  (scanning through `--socks5` implies `--skip-rate-test`, so no DHEat verdict).

Capture the full report to a file —
`tee` it to a `mktemp -d` path —
rather than `… | tail`, which throws away sections you'll need.
(If the tee and mktemp-d skills are present, follow them.)

## Reading the report

A trimmed scan looks like this — banner, graded lines, then a `rec` block:

```
(gen) banner: SSH-2.0-OpenSSH_9.7p1
(kex) sntrup761x25519-sha512@openssh.com  -- [info] hybrid post-quantum kex
(kex) ecdh-sha2-nistp256                  -- [fail] using weak elliptic curves
(enc) aes256-ctr                          -- [info]
(mac) hmac-sha1                           -- [fail] using broken SHA-1 hash
(rec) -ecdh-sha2-nistp256                 -- kex algorithm to remove
(rec) -hmac-sha1                          -- mac algorithm to remove
```

The body is grouped —
`(kex)` key exchange,
`(key)` host-key,
`(enc)` ciphers,
`(mac)` MACs —
each line tagged `[info]`, `[warn]`, or `[fail]`,
followed by a `(rec)` block of suggested add/remove changes
and `(nfo)` notes for the non-crypto findings.

Exit code is the CI gate: `0` is clean, non-zero means the run needs a look.
ssh-audit documents `1` as a connection error, `2` as warnings-only,
and `3` as at least one failure —
so wire *non-zero* as the gate rather than a specific number,
but don't read every non-zero as a crypto finding:
`1` means the scan itself didn't run
(an unreachable host, or the rate-limit stall on a re-scan),
so confirm the run actually connected before chasing a "crypto failure".

Treat `[fail]` lines as must-fix —
broken or backdoor-suspect primitives:
NIST-curve kex, ECDSA host keys, SHA-1 MACs, `ssh-rsa`, CBC ciphers
(some grade `[warn]` in some versions — trust the tag on the line, not this list).
Read warnings with judgement — they are often *intentional*.
A server hardened to the vendor guide still warns on non-PQ algorithms
(`curve25519-sha256`, the classical DH groups)
because the guide keeps them as fallback for older clients,
while the post-quantum kex (`sntrup761x25519-…`) is offered first
and used when both peers support it.
Chasing exit `0` means removing those fallbacks —
a real compatibility trade-off, not a free win (see below).

For the non-crypto notes:
`kex-strict-…-v00@openssh.com` in the kex list
means the target is protected against Terrapin —
but the note reminds you the *client* must support it too.
A DHEat note means connection throttling is weak (see fixes).

## From findings to fixes

Do **not** hand-write algorithm lists.
ssh-audit ships canonical, vendor-matched `sshd_config` snippets:

```sh
ssh-audit --list-hardening-guides            # the exact platform names
ssh-audit --get-hardening-guide "<platform>" # one of those names, verbatim
```

`--get-hardening-guide` runs **offline** — no scan, no network —
and prints ready `KexAlgorithms` / `Ciphers` / `MACs` / `HostKeyAlgorithms` /
`PubkeyAcceptedAlgorithms` / `RequiredRSASize` lines,
plus host-key and moduli steps.
Match the platform to the target's OS + OpenSSH.
The banner shows the OpenSSH version;
if it doesn't name the distro (many show only `OpenSSH_9.7`),
take the OS from the host you administer and pick the closest guide.
Prior guide revisions: `--get-hardening-guide "<platform> (version N)"`
(`N` is the guide's revision number, not an OpenSSH version).

Reading the guide correctly:

- It is **compatibility-preserving, not PQ-only.**
  It clears failures but deliberately keeps some non-PQ algorithms,
  so a guide-hardened server still scans as warnings (exit `2`), by design.
  Going PQ-only (leaving only `sntrup761x25519-…`) reaches exit `0`
  but locks out any client without post-quantum kex —
  decide that trade-off deliberately, don't chase the green number.

- **Weak host key** (e.g. an ECDSA NIST key):
  restricting `HostKeyAlgorithms` to the good types
  stops the server *offering* it — ssh-audit then stops flagging it —
  without touching the key file.
  Keep at least one strong type you actually hold a key for
  (an `ssh_host_ed25519_key` / `ssh_host_rsa_key` on disk) —
  restrict to a type with no key file and sshd has nothing to offer.
  `sshd -t` does *not* catch this, so verify by hand:
  `sudo sshd -T | grep -i hostkey` must list a type whose key file exists.
  Prefer this to regenerating host keys:
  regeneration changes the host fingerprint,
  which breaks every client's `known_hosts`
  and trips host-key-changed warnings.

- **GEX (group exchange) / moduli:**
  if you keep `diffie-hellman-group-exchange-sha256`,
  `/etc/ssh/moduli` on the target must contain only strong primes
  (`--get-hardening-guide` emits this step too).
  Editing that file is destructive — back up, guard against an empty result,
  and run it as **root in one shell** (a plain `sudo awk … > file`
  opens the redirect as your user and fails):

  ```sh
  cp /etc/ssh/moduli /etc/ssh/moduli.bak
  awk '$5 >= 3071' /etc/ssh/moduli > /etc/ssh/moduli.strong
  [ -s /etc/ssh/moduli.strong ] && mv /etc/ssh/moduli.strong /etc/ssh/moduli
  ```

  Use the guide's threshold verbatim — don't recompute it:
  the size field is the bit length **minus one**
  (that's why `3071` keeps 3072-bit primes),
  so a naive `>= 4096` would mis-filter and could empty the file.
  And OpenSSH has a hard-coded 2048-bit GEX fallback
  that config alone can't close,
  so dropping GEX entirely is the cleaner fix when you don't need it.

- **DHEat** (CVE-2002-20001):
  throttle new connections at the firewall —
  `ufw limit` on the SSH port, or an iptables `recent`-module rule —
  rather than `PerSourceMaxStartups 1`,
  which ssh-audit itself warns breaks clients on connection bursts
  and false-positives its own rate test.
  `ufw limit` caps ~6 connections / 30 s *per source IP*, so clients behind
  one shared NAT / CI egress share that budget — size the rule for them.

Some of these fixes are an `sshd_config` directive,
others a firewall rule or a moduli edit —
this skill gives you the fix and the reason.
Note the shift: the *scan* is remote and read-only,
but applying and verifying a fix needs an administrative (usually root)
shell on the target — if you were only cleared to scan it,
stop at the findings and hand them off.
*How* you roll it out —
config-management, a manual edit, a drop-in file —
is the host project's call; this skill doesn't prescribe the mechanism.

## Verify before you apply (don't lock yourself out)

Restricting SSH crypto on a host you reach only over SSH is a lockout risk.
Close it before deploying, not after —
and mind *where* each command runs:
`sshd -t` / `sshd -T` and the moduli edit run **on the target, as root**
(SSH in first — they read the target's own OpenSSH build and host keys,
not your laptop's; `sshd` lives in `/usr/sbin`, and a non-root run fails
in a way that looks like a rejected directive but is a permission error),
while `ssh -Q` runs **on the client** you connect from.

- **On the target (root) — dry-run the exact directives, writing nothing.**
  Each guide line `Directive value` becomes one `-o 'Directive=value'`:

  ```sh
  sudo sshd -t -o 'KexAlgorithms=curve25519-sha256,…' \
               -o 'Ciphers=…' -o 'MACs=…' -o 'PubkeyAcceptedAlgorithms=…'
  ```

  A non-zero exit names a directive that build rejects —
  e.g. `gss-*` algorithms need a GSSAPI-enabled build.
  A pass on your laptop's sshd proves nothing about a Linux server,
  and this can't see whether the client set still intersects — check that next.

- **On every client that must keep access — confirm it can still negotiate.**
  `ssh -Q` reports only the *local* OpenSSH binary
  (not its `ssh_config` restrictions, and nothing about other clients):

  ```sh
  ssh -Q kex; ssh -Q cipher; ssh -Q mac; ssh -Q key
  ssh -Q PubkeyAcceptedAlgorithms   # older clients: ssh -Q key / ssh -Q sig
  ```

  Check the intersection with each new server allow-list
  is non-empty on *every* axis — for *every* client population, not just your
  laptop: other admins' keys (an RSA-only holder), automation / CI behind a
  shared egress IP, PuTTY, mobile apps, non-OpenSSH libraries.
  Your own session staying up won't reveal that you locked them out.
  The silent-lockout axis is `PubkeyAcceptedAlgorithms` —
  each auth key's type must be in it:
  an ed25519 key needs `ssh-ed25519`, an RSA key `rsa-sha2-*`,
  and a *certificate* needs the matching `…-cert-v01@openssh.com`.

- **Apply with `reload`, not `restart`.**
  `systemctl reload ssh` (the unit is `ssh` on Debian/Ubuntu, `sshd` elsewhere)
  sends SIGHUP — sshd re-reads the config and rotates listeners
  **without ever signalling existing connections**, so your session is safe;
  on many distros it also runs `sshd -t` first, so a broken config fails the
  reload and the old good listener stays up.
  A full `restart` usually spares established sessions on modern systemd hosts,
  but not always (minimal / container images can drop them) —
  don't lean on it as your net.
  Either way: hold a second session open, keep the serial / web console ready,
  and test a *fresh* connection (not a reused multiplexed master) first.

- **On the target — know what's actually in effect.**
  `sshd -T` dumps the merged config from the files —
  it confirms your drop-in isn't shadowed
  (drop-ins in `sshd_config.d/*.conf` are first-match-wins in lexical order),
  but it is *not* proof the daemon reloaded; the live check is reload + re-scan.
  On socket-activated hosts (`systemctl is-active ssh.socket`)
  each connection spawns a fresh sshd, so `reload` can be a no-op —
  there the fresh-connection test / re-scan *is* the confirmation.

**Order of operations on a remote host:**

1. baseline — `sudo sshd -T` on the target;
2. write the change (drop-in / firewall rule), backing up anything you edit;
3. validate on the target: `sudo sshd -t` (the written file), and
   `sudo sshd -t -o …` to pre-check a set before writing — syntax + build;
4. `ssh -Q …` per client population — intersection non-empty on every axis;
5. open a second session; confirm console access;
6. `sudo systemctl reload ssh` (or `sshd` — the unit name from the note above);
7. test a *fresh* connection from each client population — if any fails, revert
   **in your still-open session** and reload again, then diagnose;
8. re-scan (with `--skip-rate-test` if you added a throttle in step 2).

## Re-scanning after hardening

Re-scan to confirm failures are gone.
One catch:
once you've rate-limited the SSH port for DHEat,
ssh-audit's own connection flood trips that limit,
so a full re-scan may stall or show connection errors.
Use `--skip-rate-test` on re-scans —
the crypto verdict is unaffected.
The trade-off: skipping it means you *don't* re-verify the DHEat throttle
you just added, so confirm that rule independently —
check the firewall rule and its hit counters,
or a rapid-connect probe from a throwaway source you can afford to get banned.

## Policy gate (CI)

Beyond the pass/fail exit code, ssh-audit can gate against a policy:

```sh
ssh-audit -L                        # list built-in policies
ssh-audit -M baseline.txt <host>    # snapshot this host as the baseline
ssh-audit -P baseline.txt <host>    # later: assert a host still matches
ssh-audit -P "Built-In Policy Name" <host>
```

`-j` / `-b` give JSON / batch output for a CI job to parse.
A failing policy check exits non-zero — wire that as the gate.

## Gotchas

- **Outbound firewall (Little Snitch, LuLu, …)** cuts a *fresh* CLI
  the first time it connects —
  the tool's private Python is a new binary, so allow it,
  and raise `-t` if the prompt pushes the first connection past the timeout.
  Offline commands (`--get-hardening-guide`, `--list-hardening-guides`,
  `--lookup`) need no network and won't prompt.
- **Piped output loses colour meaning** — pass `-n` and read the level tags.
- **Overlay vs public IP** hit the same `sshd` (config is global),
  so scan from whichever path reaches the SSH port.

## Reporting

After a scan, summarise:
the target and its banner (OS + OpenSSH version),
the failures (must-fix) and which fixes clear them,
the warnings that are *intentional* (kept for compatibility)
versus any that are actionable,
the recommended directives and the guide they came from,
and the exit code.
