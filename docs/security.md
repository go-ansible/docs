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

## ⛔ Untrusted data reaching the template engine

The most serious defect this audit found, and the one with the widest
blast radius.

A module's result is **data a managed host chose**. This port re-rendered
it as a Jinja template — and `lookup('pipe', ...)` runs on the **control
node**. So a host returning

```
{{ lookup('pipe','touch FILE') }}
```

from any command created that file **on the controller**: the machine
holding the vault password, the fleet's SSH keys and the cloud
credentials. One compromised target, and the control node runs its code.

Measured against ansible-core 2.21.4, a command whose output is the text
`{{ 7*7 }}`:

| | `r.stdout` |
|---|---|
| real | `"{{ 7*7 }}"` — literal, and still a string |
| this port, before `playbook` v0.135.0 | `49` — evaluated, and no longer a string |

Real refuses because it wraps such data in `AnsibleUnsafeText` and its
templar will not template those. The fix is the same brake, applied where
the second evaluation actually happens: `resolved()`, which walks the
merged variables and re-renders strings until they settle — this port's
own emulation of lazy variables, and the only place a value is rendered
twice.

### Three routes, and a proof of concept that kept firing

The payload was a `touch` into a temporary directory, run after each
attempted fix. It kept firing until all three were closed, which is the
argument for having one rather than reasoning about coverage:

1. **The `Facts` and `Registered` variable layers** are never re-rendered.
   That covers the direct case — `register:`, `set_fact:`, gathered facts.
2. **`hostvars` and `groups` are untrusted too.** They are derived
   *structures*, never a place an author writes a template, and `hostvars`
   carries every host's registered results — so leaving it resolvable put
   all of that untrusted data back under a *trusted* key.
3. **One call site had been missed.** `snapshotVars`, which publishes a
   host's variables at the end of every task, called the unbraked form —
   and it was added by the `hostvars` work a few hours earlier. A security
   brake is only as good as its least-covered caller.

### The brake keys on provenance, not on appearance

It asks whether a template **references an untrusted name**. The first
attempt asked whether *the result still looks like a template*, which is
wrong for an ordinary chain: `c: "{{ b }}/c"` renders to something that
is still a template simply because `b` has not been resolved yet, and that
version stopped resolving it. Provenance is the distinction real draws
too.

### What this does not cover

A playbook's **own** text is trusted, as it is in real: if an author
writes `{{ lookup('pipe', ...) }}`, it runs. The boundary is where data
from a host, a file or an API becomes a variable — not what the author
wrote.

### A second source: a lookup's result

The brake above keys on *untrusted names*, and that is not enough on its
own:

```yaml
vars:
  from_file: "{{ lookup('file', 'data.txt') }}"
```

names nothing untrusted, yet pulls a file's contents into a **trusted**
variable — and the next resolution pass rendered *that*. With `data.txt`
holding `{{ lookup('pipe','touch FILE') }}`, measured against
ansible-core 2.21.4:

| | result | file |
|---|---|---|
| real | the text, literal | none |
| this port, before `playbook` v0.136.0 | empty — the lookup **ran** | **created** |

A lookup returns a file's contents, a command's output, an API's answer:
**data by definition**. Real marks a lookup's *result* unsafe, and this
is the same rule. `query()` and `q()` carry it too — the same machinery
under other names, and covering only `lookup` would leave two spellings
of the same hole open.

An **ordinary lookup still resolves**, with a test to say so: refusing to
re-render a result must not stop the lookup from running, or "the payload
did not fire" would also be satisfied by breaking lookups entirely.

It was found by asking where *else* untrusted data enters rather than
waiting for it — the first fix covered data arriving through a module
result, this is the same data arriving through a function call.

### Provenance is per key, not per layer — and getting that wrong broke something

The first version inferred provenance from the **layer** a variable sits
in, and that is an approximation which is wrong in **both** directions.
`include_vars` shares the `Facts` layer with `set_fact`, and real
*templates* an included vars file — it is a file the author named. So
marking the layer broke an ordinary pattern:

```
include_vars of  greeting: "Hello {{ who }}"
real:   Hello world
ours:   Hello {{ who }}      ← playbook v0.135.0 and v0.136.0
```

Fixed in `playbook` v0.137.0, which reads a per-key mark recorded beside
the variables (`vars` v0.3.0). **A security brake that breaks a common,
legitimate pattern does not survive contact with a real playbook**, so
this is worth as much attention as the hole itself.

`vars_files` was checked at the same time and needed no change: real
evaluates a payload there too, because that file is named by the author
as well. We match.

### Storing and marking happen in one statement

Marking "the three obvious entry points" by hand left **five** other
stores into the registered layer unmarked — including the main
`register:` path — and the proof of concept fired again. The two actions
now happen in a single call, and a test refuses a raw write into those
layers.

Two details of that test earned their place:

- its exceptions **declare themselves in the code** with a
  `provenance: trusted` comment, rather than being recognised by a nearby
  function name — which is how its own first version missed
  `include_vars`, whose function is long enough that the name was out of
  window;
- it **refuses to run** if the helper is used fewer than five times,
  since "no raw stores" would otherwise be satisfied by an engine that
  stores nothing.

### Where the hole was, and where it was not

The vulnerable mechanism was *repeated variable resolution* — and only
that. The other places a value meets an evaluator render **once** and do
not re-render what they substituted. Checked with the same payload
against the **vulnerable** build, which is the only way the answer means
anything: `when:`, `loop:`, a module argument, a task's `vars:`,
`changed_when:` and `assert:` were all clean *before* the fix as well as
after.

That is a boundary, not a reassurance: it says the fix had one mechanism
to cover, and that the single-pass paths are safe by construction rather
than by luck.

## A fact a target injects is inert

A fact *value* can contain a newline, and the probe's output is parsed
line by line — so a managed host can make the parser see an extra
`key=value`:

```go
parseKV("distribution='Debian\nconnection=ssh'")
  -> map[connection:ssh distribution:Debian]
```

`/etc/os-release` is the reachable route: the probe **sources** it and
emits `$ID`, `$VERSION_ID`, `$ID_LIKE` and `$VERSION_CODENAME`.

It reaches nothing, because `assemble` **names** every fact it builds and
never ranges over the parsed map — an allow-list. The stakes if it did
not: `assemble` returns *unprefixed* keys and the engine adds `ansible_`,
so an injected `connection` would land as `ansible_connection`, which
since `playbook` v0.128.0 takes effect **per task** and decides where
later tasks run.

That safety is one refactor away from being false, so it is pinned by a
test (`facts` v0.18.1) that fails if the parsed map is ever copied
wholesale. The env-variable route is guarded the same way, by cutting the
stream before the first `ENV ` line — a guard that was already there.

## A fact can now redirect where later tasks run

As of `playbook` v0.128.0 a task's own variables decide which connection
it gets — which is real Ansible's behaviour, and is what makes
`set_fact: {ansible_connection: ...}` work. It also means a value that
sets `ansible_connection`, `ansible_host`, `ansible_user` or `ansible_port`
can send **subsequent tasks to a different endpoint**.

This is parity, not a hole we opened: real does the same, measured — after
`set_fact: {ansible_connection: ssh}` the next task really does go over
SSH and reports UNREACHABLE. It is on this page because the consequence is
worth stating plainly rather than leaving for someone to find.

What it means in practice:

- **A module result is trusted.** A module that returns `ansible_facts`
  with a connection variable in it changes where later tasks run. Real
  Ansible has the same property, and it is why `gather_facts` against a
  host you do not control is a trust decision, not a read-only one.
- **A custom fact file is namespaced, and that matters here.**
  `ansible_local` entries land under `ansible_local.<name>`, keyed by the
  `.fact` filename, so a target's own file cannot reach a top-level
  `ansible_connection`.
- **The connection variables are a known, closed set.** They are listed
  in one place and a test reads the functions that build a connection and
  fails if one consults a variable the list does not name — which is how
  thirteen `ansible_winrm_*` variables were found missing from it. A
  reviewer asking "what can redirect a task?" has one list to read.

## Resource exhaustion: a malformed expression leaked a goroutine per evaluation

`gonja`'s `tokens.Lex` runs the lexer in a goroutine feeding an **unbuffered**
channel. The expression evaluator drove that stream with `ParseExpressionNode`,
which stops at the closing `}}` — and on a parse error returns immediately, with
tokens still unsent. The lexer goroutine is then blocked on a send nobody will
ever receive, holding the lexer, its input string and a token alive with it.

`Eval` and `EvalBool` are the `when:` path: once per task, per host. A single
malformed condition therefore leaked a goroutine on **every** evaluation —
unbounded growth in goroutines and memory for as long as the run lasts, out of
one typo in a playbook.

### Measured — and the first measurement said there was nothing

200 evaluations of an expression that fails to parse:

| expression | where it fails | leaked goroutines |
|---|---|---|
| `a + 1 > 0` | — | +0 |
| `a +` | at the very end | +0 |
| `a + + + b \| nosuchfilter \| another` | mid-stream | **+200** |

The first probe used `a +` and reported zero, which is true and useless: a
failure at the *end* of the stream leaves nothing unsent. Only a failure
**mid-stream** strands the lexer. A test case has to fail in the right place,
and "no leak found" was the wrong conclusion from the right number.

The fix is `tokens.LexAll`, which lexes synchronously into a slice and starts no
goroutine at all; `NewStream` accepts either a channel or a slice, so it is a
drop-in. Removing a goroutine handoff per token also made that hot path about
**2.05×** faster — ~9728 → ~4748 ns/op, 169 → 149 allocs/op.

### The test carries its own positive control

A leak probe that has gone blind reports zero just as loudly as a fixed engine
does. `TestLeakProbeCanSeeALeak` abandons lexer streams on purpose and **fails**
if it cannot count them, so a green leak test means something. Verified by
neutering: restoring `tokens.Lex` makes the test fail with `+200 goroutines`.

It was found in a `ppc64le` CI timeout dump — a lexer goroutine parked on
`chan send` for nine minutes, beside a test that had timed out for an unrelated
reason.

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
