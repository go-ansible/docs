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
