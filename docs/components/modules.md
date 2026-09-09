# modules

[![CI](https://github.com/go-ansible/modules/actions/workflows/ci.yml/badge.svg)](https://github.com/go-ansible/modules/actions/workflows/ci.yml)

`github.com/go-ansible/modules` implements Ansible's module execution model
and the core module library: **566 modules registered** — all 62 of
`ansible.builtin`, all 14 of `ansible.posix`, and 490 curated from
`community.general` (577 real modules in that collection; the remainder is
deliberately excluded, each with a real, checked reason — see the
[org profile](https://github.com/go-ansible) for the full list).

Unlike real Ansible, which copies a Python script to the target and runs it
there, a module here runs its logic on the **control node** and reaches the
target only through a [`go-remoteexec/transport`](https://github.com/go-remoteexec/transport)
`Connection`'s `Exec`/`Put`/`Fetch` primitives. The observable behavior is the
same — the target ends up in the same state — but the difference is
architectural: a module needs no Go toolchain, no interpreter, nothing
installed on the target at all.

## API

```go
type Result struct {
	Changed bool
	Failed  bool
	Msg     string
	Facts   map[string]any // merged into ansible_facts — set_fact, setup, ...
	Extra   map[string]any // module-specific fields — command's rc/stdout/stderr, ...
}

func Ok(msg string) Result
func Changed(msg string) Result
func Fail(msg string) Result
func (r Result) WithExtra(key string, value any) Result

type Func func(ctx context.Context, conn remoteexec.Connection, args map[string]any) (Result, error)

type Registry struct { /* ... */ }

func NewRegistry() *Registry
func Default() *Registry // pre-populated with all 566 built-in modules
func (r *Registry) Register(name string, fn Func)
func (r *Registry) Get(name string) (Func, bool)
func (r *Registry) Names() []string
func (r *Registry) Run(ctx context.Context, name string, conn remoteexec.Connection, args map[string]any) (Result, error)

func NormalizeName(name string) string // strips a known collection prefix
```

`Get` falls back to `NormalizeName(name)`, so a fully-qualified collection
name (`ansible.builtin.copy`, `community.general.ufw`, and the other three
known prefixes) resolves to exactly the same `Func` as its bare form — no
separate registration needed per FQCN.

A module's arguments arrive already Jinja2-rendered by the caller (this
package never templates anything itself). A non-nil `error` from `Run` means
the module could not determine an outcome at all (a transport failure); an
*expected* failure — the `fail` module itself, `assert` on a false condition,
a package manager reporting "not found" — is a `Result{Failed: true}` with a
`nil` error, since it is not the module's own execution that went wrong.

`Run` also finalizes a result on its way out, which is where real Ansible
does the same work (`AnsibleModule.exit_json`) rather than leaving each of
the ~40 modules that produce output to remember it: a trailing newline is
trimmed off `stdout`/`stderr`, and a matching `stdout_lines`/`stderr_lines`
is added. The trim is exactly Python's `rstrip("\r\n")` — every trailing
carriage return and newline goes, trailing spaces and tabs survive, and
leading newlines survive.

## Measured against real Ansible

The module surface has been run side by side with **real ansible-core
2.21.4**: the same playbook through both engines, **twice**, comparing the
registered results *and* `diff -r` of the filesystem trees each produced.
Running it twice is the point — the second pass is what shows whether
idempotence matches, which is where a module port most easily drifts.

Across `file` (directory/touch/absent), `copy`, `template`, `lineinfile`
(both plain and `regexp`), `blockinfile`, `replace`, `stat`, `slurp`,
`find`, `command`, `shell`, `set_fact` and `assert`, the compared values
are **byte-identical to real Ansible on both passes**, and the two produced
trees match exactly, permissions included.

Two defects were found and fixed that way, neither of which any unit test
had caught:

- **`stdout` kept its trailing newline**, and **`stdout_lines`/`stderr_lines`
  did not exist at all** — `{{ result.stdout_lines }}` is everyday playbook
  usage and rendered `null`. Both are handled in `Run`, as above.
- **`assert: that: ["x == 5"]`** — the form every real playbook uses —
  failed with "did not evaluate to a boolean". This package's own
  documentation already stated that the playbook engine would evaluate
  those expressions before calling the module, and nothing did; the
  engine now does, in [`playbook`](playbook.md)'s `runDirective`, for the
  same reason real Ansible makes `assert` an action plugin rather than a
  module — the conditions are Jinja2 expressions over the host's own
  variables, and only the engine holds those.

## Example

```go
package main

import (
	"context"
	"fmt"
	"log"

	"github.com/go-ansible/modules"
	remoteexec "github.com/go-remoteexec/transport"
)

func main() {
	reg := modules.Default()
	conn := remoteexec.NewLocal() // runs on this machine

	result, err := reg.Run(context.Background(), "copy", conn, map[string]any{
		"src":  "/tmp/source.txt",
		"dest": "/tmp/dest.txt",
	})
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(result.Changed, result.Msg)
}
```

## What is not implemented

Every architectural substitution and disclosed gap is documented in the
module's own Go doc comment, not hidden — `ansible-doc <module>` (via the
[`cli`](cli.md) package) prints it. The recurring patterns: a module whose
real Python implementation calls a SaaS/cloud API directly instead shells out
to that platform's own official CLI when one exists (`gh`/`glab`/`scw`/
`aliyun`/`ilorest`/... — never a secret in argv, always an environment
variable or a pre-existing authenticated session); a module real Ansible runs
*from the controller* against the target's own SSH endpoint directly
(`ansible.posix.synchronize`) fails loud rather than silently approximating,
since this port's `Connection` abstraction has no way to expose a target's
host/port/credentials from inside a module; and a handful of modules
knowingly reproduce a real upstream bug verbatim rather than "fixing" it,
because the reference implementation — not this project's own judgment — is
the source of truth for what a real user's playbook may already depend on.
