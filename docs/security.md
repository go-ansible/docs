# Security

This port executes shell commands on remote targets, so a value that reaches a
command string is a value that reaches a shell. Two kinds of defect follow from
that, and this page states what is done about each, what was found when it was
audited, and what remains unsolved and why.

- **Injection** — a playbook-supplied value changing the *structure* of the
  command rather than being an argument to it.
- **Exposure** — a credential ending up somewhere another local user can read
  it, most often a process's own `argv`, which `ps` prints to anyone.

## Injection: quoting is the control, not care

Every value interpolated into a command string goes through this package's
`shellQuote` — single quotes with `'\''` for an embedded quote, which no shell
metacharacter escapes. There are **1 715** such call sites across the module
library. The rule is mechanical on purpose: *being careful* does not scale to
608 files, and a missed site is indistinguishable from a deliberate one until
somebody looks.

### What the audit found

`async_status`'s `jid` arrives from a playbook argument
(`async_status: {jid: ...}`) and was interpolated **unquoted** — including into
an `rm -rf`. A `jid` of `x; touch /tmp/pwned ;` ran that `touch`. Both
`AsyncCleanup` and `AsyncCheck` were affected. Fixed in
[`modules` v0.80.0](https://github.com/go-ansible/modules/releases/tag/v0.80.0);
a test in that package demonstrates the hole with a harmless payload inside
`t.TempDir()` and fails if it reopens.

`validAsyncJID` — digits and a single dot, the shape this package itself
generates — sits behind the quoting as defence in depth, not instead of it.

Its first version returned an **error** for a malformed `jid`, and that broke
parity: real Ansible accepts `jid: nope` and reports its ordinary not-found
result (`msg: "could not find job"`, `started` and `finished` true). An existing
test caught it. A malformed `jid` is now mapped to exactly that answer — real's
own behaviour, and a value that never reaches a shell. **A security fix that
changes what a caller sees is not finished**, because the next person's workaround
is to turn it off.

## Exposure: where a credential may and may not go

A module that needs a secret at the target end puts it in the **environment**,
through [`transport`](https://github.com/go-remoteexec/transport)'s
`ExecWithEnv` — which is what real Ansible does, via `run_command`'s
`environ_update`. It is an optional interface (`EnvExecer`) rather than a
`Connection` method because SSH genuinely cannot set arbitrary variables:
`sshd`'s `AcceptEnv` decides, and it defaults to almost nothing. Where the
environment is unavailable the call reports which path it took rather than
failing silently.

### The boundary, measured

An environment assignment written as a command prefix is **not** always visible.
Which shape it is decides it, and both were measured rather than reasoned about:

| command | prefix visible in `ps`? |
|---|---|
| `sh -c "VAR=x sleep 4"` | **no** — the shell `exec`s into the command and nothing keeps that `argv` |
| `sh -c "VAR=x sleep 4; true"` | **yes** — a second statement means the shell must stay alive, holding it |

An earlier probe reported the opposite, because its canary was a fixed string
that the shell which had *written the probe file* still carried in its own
`argv`. The canary is generated at run time now, and the test that asserts the
invisible case carries a **mandatory positive control** using the visible shape:
if the control cannot fire, the test fails rather than passing for free.

### What the audit found

24 modules build a `CREDENTIAL=` assignment into a command string. 23 are the
lone-command shape, which is safe. **`keyring` was not**: it built

```
KEYRING_PASSWORD=<secret>; USER_PASSWORD=<secret>; <dispatch>
```

— two statements, so the shell held both passwords for the command's whole
lifetime. Fixed in `modules` v0.80.0 and
[v0.80.1](https://github.com/go-ansible/modules/releases/tag/v0.80.1).

Two things that fix taught, both kept here because they generalise:

- `keyringDispatch` builds **both** platform branches into one shell string, so
  a literal password in the macOS branch was on the command line **even on
  Linux**, where that branch never runs. A dispatcher's unused arm is still
  argv.
- v0.80.0 fixed two of the module's three call sites and its regression test
  passed, because the test drove `state: present` only — and `state: absent` is
  the only state that reaches the third. The test is now a table over every
  state the module accepts, and each case carries a **witness**: the command its
  own state must reach. Without one, a case that returns early passes by finding
  no secret in no commands at all. What surfaced the gap was the detector that
  found the original exposure still naming `keyring.go` after the fix merged.

## What remains, named rather than smoothed over

- **`security`'s own `argv` on macOS.** `keyring` on macOS runs
  `security add-generic-password … -w "$USER_PASSWORD"`, and the value is in
  that short-lived process's `argv` while it runs. There is no stdin form —
  measured: `printf %s pw | security add-generic-password … -w` with no value
  stores nothing — and the Keychain API needs cgo, which this port does not use.
  The Linux branch does not have this: `secret-tool` reads stdin.
- **Modules that shell out to a platform's own CLI** pass credentials through an
  environment variable or a pre-existing authenticated session, never `argv` —
  and where a vendor tool offers only a file (DMTF's `redfishtool` and its `-c
  cfgFile`), the file is used.
- **`ansible.posix.synchronize`** fails loudly instead of approximating: real
  Ansible runs it *from the controller* against the target's own SSH endpoint,
  and this port's `Connection` abstraction deliberately gives a module no way to
  read a target's host, port or credentials.

## Reporting

Open an issue on the repository the defect is in. There is no embargo process
and no private channel; this is a parity port maintained in the open, and a
proof-of-concept in an issue is more useful than a description.

## How these were found

A detector per class, run over the whole library, and **each one shown capable
of reporting a violation before its zero was believed** — the statement-aware
credential-prefix check was given the pre-fix `keyring` line and fired on it,
and the `ps` test's control uses the shape known to be visible. A probe that
reports "nothing found" is worth nothing until it has found something.
