# Security

This port executes shell commands on remote targets, so a value that reaches a
command string is a value that reaches a shell. Two kinds of defect follow from
that, and this page states what is done about each, what was found when it was
audited, and what remains unsolved and why.

- **Injection** — a playbook-supplied value changing the *structure* of the
  command rather than being an argument to it.
- **Exposure** — a credential ending up somewhere another local user can read
  it, most often a process's own `argv`, which `ps` prints to anyone.

Two more came out of auditing the rest of the stack rather than the module
library, and they are on this page because they have the same consequence:

- **Disclosure** — a secret the playbook author explicitly asked to hide
  (`no_log`) reaching the screen anyway, through an output path nobody checked.
- **Partial application** — a run that was going to be refused doing part of its
  work first. Not a secret at all; it changes machines, which is worse.

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

## Disclosure: `no_log` has to cover every output path

`no_log: true` is the author saying *this task's result is a secret*. The
default callback honours it in the printed result, the verbose dump and every
item label — and **did not** honour it in the diff.

```yaml
- copy: {content: "{{ db_password }}", dest: /etc/app.conf}
  no_log: true
```

Under `--diff` — the one flag a reader adds when they want to see what
changed — that printed the password in full. Fixed in
[`playbook` v0.123.0](https://github.com/go-ansible/playbook/releases/tag/v0.123.0).

Real's mechanism is worth stating, because it explains why real's own
`v2_on_file_diff` has no `no_log` branch to copy: `as_result_dict()` **replaces**
a `no_log` result with the keys in its `PRESERVE` set — `_ansible_no_log`,
`attempts`, `changed`, `deprecations`, `exception`, `retries`, `warnings` — plus
`censored`. `diff` is not among them, so there is nothing left to print. A
struct-based port has to do by checking what real does by construction, which is
exactly the kind of difference that leaves one path uncovered.

**Why the existing test did not catch it:** it covered ok, changed, skipped and
failed — four output paths, and `--diff` is a fifth. Under the neuter it stays
**green** while the new test fails, which is the proof it could never have seen
that path. The same shape as the `keyring` gap above. Also measured, so the next
reader need not wonder: real does **not** reveal a `no_log` result at `-vvv`.

## A secret belongs in a file, not a variable

This port honoured `ANSIBLE_VAULT_PASSWORD`, carrying the vault password
**itself**. Real Ansible has no such variable — only
`ANSIBLE_VAULT_PASSWORD_FILE`, which names a *path* — and that asymmetry is the
whole point: an environment variable is inherited by every child process, which
here means every local command a module runs. A file has an owner and a mode.

It is **refused** now, with an error naming its replacement, rather than ignored:
ignoring it would send a caller who relied on it to the password *prompt*, which
in a CI job with no terminal reads EOF and reports something unrelated. The error
does not repeat the secret.

Real's own variable, and `[defaults] vault_password_file` in `ansible.cfg`, were
**not supported at all** before this — so someone doing the correct thing got no
password. Both work as of
[`cli` v0.127.0](https://github.com/go-ansible/cli/releases/tag/v0.127.0).

## Partial application: a refused run that ran anyway

Not a secret, and the most consequential finding on this page. Real refuses a
playbook naming a module it cannot resolve and **touches no host**. This port ran
every task up to the bad one:

| | exit | first task's `touch` file |
|---|---|---|
| real | 4 | **not created** |
| this port, before | 2 | **created** |

So a **typo in task 5 applied tasks 1 through 4 for real**, then failed. Fixed in
[`playbook` v0.124.0](https://github.com/go-ansible/playbook/releases/tag/v0.124.0),
which refuses before the first play: a plain task, inside `block`/`rescue`/
`always`, in a handler, a bad collection prefix, and a role's tasks. A file pulled
in at run time by `include_tasks` is still only checked where it runs — real
refuses those too, and that gap is named rather than papered over. A templated
module name is left alone, because real accepts it.

The first probe for this reported the opposite, and how is worth keeping: it
looked for the first task's output in real's stdout and **found it**, because
real's error message quotes the offending source lines — including the
neighbouring one. A file witness is what settled it.

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

A detector per class, run over the whole stack, and **each one shown capable of
reporting a violation before its zero was believed** — the statement-aware
credential-prefix check was given the pre-fix `keyring` line and fired on it;
the `ps` test's control uses the shape known to be visible; the `no_log` test's
control is the same task *without* `no_log`, and it is fatal, because the secret
being absent is equally satisfied by a diff that never rendered. A probe that
reports "nothing found" is worth nothing until it has found something.

Three of the findings on this page were measured **against the real binary
running beside this one**, which is the only way some of them could have been
seen at all: that real strips `diff` from a `no_log` result, that real ignores
`ANSIBLE_VAULT_PASSWORD` and reads `ANSIBLE_VAULT_PASSWORD_FILE`, and that real
refuses an unresolvable module before touching a host. None of the three is
documented behaviour anyone would think to look up.

And two of the probes were **wrong the first time**, in the same way: they read
something other than what they meant to. One matched a canary in the shell that
had *written the probe file*; one matched a source line real's error message
*quoted back*. Both were settled by a witness that an output stream cannot
fabricate — a runtime-generated value, and a file on disk.
