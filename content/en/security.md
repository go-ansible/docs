---
title: "Security"
weight: 110
---

# Security {#security}

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

## Injection: quoting is the control, not care {#injection-quoting-is-the-control-not-care}

Every value interpolated into a command string goes through this package's
`shellQuote` — single quotes with `'\''` for an embedded quote, which no shell
metacharacter escapes. There are **1 715** such call sites across the module
library. The rule is mechanical on purpose: *being careful* does not scale to
608 files, and a missed site is indistinguishable from a deliberate one until
somebody looks.

### What the audit found {#what-the-audit-found}

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

## Exposure: where a credential may and may not go {#exposure-where-a-credential-may-and-may-not-go}

A module that needs a secret at the target end puts it in the **environment**,
through [`transport`](https://github.com/go-remoteexec/transport)'s
`ExecWithEnv` — which is what real Ansible does, via `run_command`'s
`environ_update`. It is an optional interface (`EnvExecer`) rather than a
`Connection` method because SSH genuinely cannot set arbitrary variables:
`sshd`'s `AcceptEnv` decides, and it defaults to almost nothing. Where the
environment is unavailable the call reports which path it took rather than
failing silently.

### The boundary, measured {#the-boundary-measured}

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

### What the audit found {#what-the-audit-found_1}

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

## Disclosure: `no_log` has to cover every output path {#disclosure-no_log-has-to-cover-every-output-path}

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

## A secret belongs in a file, not a variable {#a-secret-belongs-in-a-file-not-a-variable}

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

## Partial application: a refused run that ran anyway {#partial-application-a-refused-run-that-ran-anyway}

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

## ⛔ Untrusted data reaching the template engine {#untrusted-data-reaching-the-template-engine}

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

### Three routes, and a proof of concept that kept firing {#three-routes-and-a-proof-of-concept-that-kept-firing}

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

### The brake keys on provenance, not on appearance {#the-brake-keys-on-provenance-not-on-appearance}

It asks whether a template **references an untrusted name**. The first
attempt asked whether *the result still looks like a template*, which is
wrong for an ordinary chain: `c: "{{ b }}/c"` renders to something that
is still a template simply because `b` has not been resolved yet, and that
version stopped resolving it. Provenance is the distinction real draws
too.

### What this does not cover {#what-this-does-not-cover}

A playbook's **own** text is trusted, as it is in real: if an author
writes `{{ lookup('pipe', ...) }}`, it runs. The boundary is where data
from a host, a file or an API becomes a variable — not what the author
wrote.

### A second source: a lookup's result {#a-second-source-a-lookups-result}

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

### Provenance is per key, not per layer — and getting that wrong broke something {#provenance-is-per-key-not-per-layer-and-getting-that-wrong-broke-something}

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

### Storing and marking happen in one statement {#storing-and-marking-happen-in-one-statement}

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

### Where the hole was, and where it was not {#where-the-hole-was-and-where-it-was-not}

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

## A fact a target injects is inert {#a-fact-a-target-injects-is-inert}

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

## A fact can now redirect where later tasks run {#a-fact-can-now-redirect-where-later-tasks-run}

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

## A leak I reported that was not one — and the gap underneath it {#a-leak-i-reported-that-was-not-one-and-the-gap-underneath-it}

This section said real `community.general.telegram` leaked its bot token,
because its failure message embeds the endpoint:

```python
msg=f"Failed to send message, return status = {info['status']}\n"
    f"url = {url}\napi_args = {api_args}"   # url = .../bot<token>/<method>
```

**That was wrong.** `AnsibleModule._return_formatted` — which `exit_json` and
`fail_json` both run — calls `remove_values(kwargs, self.no_log_values)`, and
`_remove_values_conditions` does *substring* replacement:

```python
for omit_me in no_log_strings:
    native_str_value = native_str_value.replace(omit_me, '*' * 8)
```

So the token inside that url prints as `********`. Real has a scrubber, and the
claim was published here without checking for one. The modules page's own
description of `_return_formatted` — *"runs on EVERY result, from `exit_json`
and `fail_json` alike"* — was already in this documentation.

### The finding is real, and it is about this port {#the-finding-is-real-and-it-is-about-this-port}

⛔ **There is no equivalent of `remove_values` here.** Task-level `no_log: true`
is honoured, and suppresses a whole result — that mechanism and the bug once
found in it are described under *Disclosure* above. What does not exist is
real's *automatic* scrubbing: nothing removes the **value** of a `no_log`
argument from a message that happens to contain it.

The practical consequence is an inversion of the usual rule. Reproducing real's
behaviour is this project's default, and here it would be unsafe: real can
afford to put a credential in a message because a scrubber stands behind it, and
this port cannot. So `telegram` reports the status and `api_args` and omits the
url — the same code either way, for the opposite reason.

`bigpanda` is the second instance, and it is a plainer one: real *returns* the
token, as a field of its own deployment object, and gets away with it for the
same reason. Here the field is omitted. Both divergences are in the module's
doc comment next to real's line that justifies it, and both are held by a test
rather than by the comment.

It also means the discipline has to hold at the point a message is *composed*,
not at the point it is printed. Every module that reaches an HTTP service
interpolates something; a credential must not be part of it.

### Where a credential can hide in these modules {#where-a-credential-can-hide-in-these-modules}

These services have no official CLI, so this port posts to the same endpoints
through `curl` **on the target** — where real's own module runs and calls
`fetch_url` from. A credential then has five places to sit, and all five are
places a naive implementation puts on the command line:

| | example |
|---|---|
| a form field | `pushover`'s `app_token` and `user_key` |
| the **URL path** | `grove`'s `channel_token`, `telegram`'s and `discord`'s tokens, `rocketchat`'s token — which is *two* path segments and refuses to be anything else |
| the **query string** | `nexmo`'s `api_key` *and* `api_secret`, `google_chat`'s `key` and `token` |
| a **header** | `cisco_webex`'s and `bigpanda`'s bearer tokens |
| the **basic-auth user** | `campfire`'s token, sent as `<token>:X` — `curl -u` would put it in `argv` |

`curlDo` sends the url, the query string, the headers, the body and any
basic-auth pair on **stdin**, through curl's own `-K -` config — the mechanism
`infinityCurl` already used for HTTP Basic credentials, and the one the
`Connection` interface documents for reaching `sudo` without a password in
`argv`.

The URL row is the one worth dwelling on. `-u` and `-H` are the obvious
mistakes; a webhook url looks like an address rather than a secret, and for
four of these modules it *is* the secret. A review that only looks for the
flags that take credentials will pass all four.

`assertSecretsOffArgv` asserts it per module, and carries its own control: each
secret must appear on **stdin**. Without that half, the check passes just as
well on a module that never sent the credential at all.

## `vault`: the crypto was right, the property was untested {#vault-the-crypto-was-right-the-property-was-untested}

The audit of `vault` found nothing wrong with the cryptography:

- `crypto/rand` for the salt, not `math/rand`
- PBKDF2-SHA256, **10000** rounds, 80 bytes split 32/32/16 — real Ansible's
  parameters, which is also why the interop tests pass in both directions
- `subtle.ConstantTimeCompare` for the HMAC
- the HMAC verified **before** decrypting: encrypt-then-MAC
- PKCS7 unpadding that validates the length *and every pad byte*, and which is
  **unreachable without a valid HMAC** — so it is not a padding oracle

What no test asserted was the property all of that exists to provide.

### AES-CTR is malleable, and that is the whole point of the HMAC {#aes-ctr-is-malleable-and-that-is-the-whole-point-of-the-hmac}

Ciphertext is plaintext XOR a keystream, so flipping a ciphertext bit flips
exactly the corresponding plaintext bit. Nothing about the cipher stops someone
with write access to an encrypted file — a repository, a backup, a clone in
flight — from making a chosen, surgical edit to a secret **they cannot read**.

The existing tests covered a **wrong password** and malformed **shapes**. A
*correct* password against a *modified* ciphertext was untested. Those are
different properties: "the wrong key is rejected" is not "the data is
authentic".

### The control re-authenticates the tamper {#the-control-re-authenticates-the-tamper}

A tampered-ciphertext test that passes because the flip was a no-op proves
nothing, so the control does what an attacker cannot — recomputes a valid HMAC
over the tampered ciphertext — and shows the edit landing:

```
permit_root_login: 0   becomes   permit_root_login: 1
```

one flipped bit, because `'0'` is `0x30` and `'1'` is `0x31`. It fails loudly if
the flip turns out to be ineffective.

Four tampering shapes are now refused with `ErrHMACMismatch`: a flipped
ciphertext bit, a flipped salt bit, a flipped HMAC bit, and a truncated
ciphertext.

Verified by neutering — and the neuter was written to keep **compiling**, so
that a build failure could not be mistaken for evidence. Discarding the
comparison's *result* makes the flipped-ciphertext case decrypt cleanly:

```
tampering was not refused: err=<nil>, plaintext="permit_root_login: 1\n"
```

All four shapes fail under the neuter while the untampered round trip keeps
passing, so they are not passing by refusing everything.

### The judge cannot be allowed to skip {#the-judge-cannot-be-allowed-to-skip}

The interop tests measure this package against the real `ansible-vault`, and a
skipped judge reads exactly like a passing one. The CI lane installs
`ansible-core==2.21.4` and sets `ANSIBLE_REQUIRE`, which turns the "not
installed" skip into a **failure**. A comment claiming CI did not install
Python had gone stale against that and was corrected.

## The file we examined and the file we ran were not the same file {#the-file-we-examined-and-the-file-we-ran-were-not-the-same-file}

`Load` stats the inventory path and reads its mode to decide the file is a
dynamic inventory script — real Ansible's own detection, any executable bit
set. It then handed that same string to `exec.Command`, which does a **`$PATH`
lookup** for any name with no separator in it.

So `-i inv.sh` stat'd `./inv.sh`, concluded from *that* file's permissions that
it was a script, and then executed a different `inv.sh` found somewhere on
`$PATH`. A check is worth nothing when it is applied to a different object than
the one the decision acts on.

### Measured, with an impostor of the same basename earlier on `$PATH` {#measured-with-an-impostor-of-the-same-basename-earlier-on-path}

| | `Load("inv.sh")` runs |
|---|---|
| before | `FROM-SOMEWHERE-ON-PATH` |
| after | **`FROM-THE-FILE-WE-NAMED`** |

Real **ansible-core 2.21.4**, given that identical setup, runs the file it was
pointed at, and names it absolutely in its own diagnostics:

```
Origin: <inventory script output from '.../work/inv.sh'>
```

so this was a parity divergence as well as a security defect. Fixed in
`go-ansible/inventory v0.19.3` with `filepath.Abs` in `ParseScript`, at the
**exported** entry point, so a caller that bypasses `Load` is covered too.

### A command name and a file are different things {#a-command-name-and-a-file-are-different-things}

The stack's only other `exec.Command` on a variable is `ansible-vault`'s
`$EDITOR`, and that one is **correct as it stands**: a command *name* is
supposed to be resolved on `$PATH`, which is how `EDITOR=vim` works at all.
Only a path we have already stat'd must not be. The audit question is therefore
not "does this do a `$PATH` lookup?" but "did something already decide this
string names a particular file?"

### The test carries a control, because the trap can disarm itself {#the-test-carries-a-control-because-the-trap-can-disarm-itself}

`TestTheImpostorIsReachableOnPath` resolves the bare name itself and **fails if
the impostor is not reachable**. Without it, the real test would pass whenever
the setup silently failed to arm — a green that means nothing. Verified by
neutering: removing `filepath.Abs` fails only the one test, with
`FROM-SOMEWHERE-ON-PATH`, while the control and the explicit-`./` case stay
green.

## Resource exhaustion: a malformed expression leaked a goroutine per evaluation {#resource-exhaustion-a-malformed-expression-leaked-a-goroutine-per-evaluation}

`gonja`'s `tokens.Lex` runs the lexer in a goroutine feeding an **unbuffered**
channel. The expression evaluator drove that stream with `ParseExpressionNode`,
which stops at the closing `}}` — and on a parse error returns immediately, with
tokens still unsent. The lexer goroutine is then blocked on a send nobody will
ever receive, holding the lexer, its input string and a token alive with it.

`Eval` and `EvalBool` are the `when:` path: once per task, per host. A single
malformed condition therefore leaked a goroutine on **every** evaluation —
unbounded growth in goroutines and memory for as long as the run lasts, out of
one typo in a playbook.

### Measured — and the first measurement said there was nothing {#measured-and-the-first-measurement-said-there-was-nothing}

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

### The test carries its own positive control {#the-test-carries-its-own-positive-control}

A leak probe that has gone blind reports zero just as loudly as a fixed engine
does. `TestLeakProbeCanSeeALeak` abandons lexer streams on purpose and **fails**
if it cannot count them, so a green leak test means something. Verified by
neutering: restoring `tokens.Lex` makes the test fail with `+200 goroutines`.

It was found in a `ppc64le` CI timeout dump — a lexer goroutine parked on
`chan send` for nine minutes, beside a test that had timed out for an unrelated
reason.

## What remains, named rather than smoothed over {#what-remains-named-rather-than-smoothed-over}

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

## Reporting {#reporting}

Open an issue on the repository the defect is in. There is no embargo process
and no private channel; this is a parity port maintained in the open, and a
proof-of-concept in an issue is more useful than a description.

## How these were found {#how-these-were-found}

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
