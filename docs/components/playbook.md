# playbook

[![CI](https://github.com/go-ansible/playbook/actions/workflows/ci.yml/badge.svg)](https://github.com/go-ansible/playbook/actions/workflows/ci.yml)

`github.com/go-ansible/playbook` is the engine that drives
[`inventory`](inventory.md), [`vars`](vars.md), [`template`](template.md),
[`modules`](modules.md) and [`facts`](facts.md) together against a real play
— the piece the original [Roadmap](../roadmap.md) once described as "not yet
built." It now implements real per-host execution: `when`/`loop`/`register`,
`block`/`rescue`/`always` with genuine per-host recovery, `notify`/handlers,
`become`, roles, `include_tasks`/`import_tasks`/`include_role`/`import_role`/
`import_playbook`, `delegate_to`, `serial` batching, tag filtering, both
the `linear` (default) and `free` execution strategies, `until`/`retries`/
`delay`, `run_once`, a real `Forks` concurrency cap, `vars_prompt`, and
`async`/`poll` for `command`/`shell`.

See the **[engine feature matrix](https://go-ansible.github.io/)** on the
landing page for the current, code-checked status of every playbook
directive — that table is regenerated from `engine.go`/`playbook.go`
directly, so it stays authoritative in a way a second, hand-maintained copy
here would not.

## API

```go
type Engine struct {
	Inventory *inventory.Inventory
	Modules   *modules.Registry
	Template  *template.Engine
	ExtraVars map[string]any
	Connect   Connector

	BaseDir  string
	RunTags  []string
	SkipTags []string
	OnResult func(Result)
}

func New(inv *inventory.Inventory) *Engine
func (e *Engine) RunPlaybook(ctx context.Context, pb Playbook) (*RunResult, error)

type Playbook []Play

func Parse(data []byte) (Playbook, error)
func ParseFile(path string) (Playbook, error)

type Result struct {
	Host, Task, Module     string
	Changed, Failed, Skipped bool
	Msg   string
	Extra map[string]any
}

type RunResult struct{ Plays []PlayResult }

func (rr *RunResult) Failed() bool
func (rr *RunResult) Summary() map[string]*HostSummary // real Ansible's PLAY RECAP, by host
```

`New` returns an `Engine` pre-wired with the built-in module registry
([`modules.Default()`](modules.md)), a fresh template engine, and
`DefaultConnect` — real SSH/local/become connections via
[`go-remoteexec/transport`](https://github.com/go-remoteexec/transport). Set
`Connect` to a custom `Connector` to run against a fake or instrumented
connection in tests.

## Example

```go
package main

import (
	"context"
	"fmt"
	"log"

	"github.com/go-ansible/inventory"
	"github.com/go-ansible/playbook"
)

func main() {
	inv, err := inventory.ParseYAML([]byte(`
all:
  hosts:
    localhost:
      ansible_connection: local
`))
	if err != nil {
		log.Fatal(err)
	}

	pb, err := playbook.ParseFile("site.yml")
	if err != nil {
		log.Fatal(err)
	}

	e := playbook.New(inv)
	rr, err := e.RunPlaybook(context.Background(), pb)
	if err != nil {
		log.Fatal(err)
	}
	if rr.Failed() {
		log.Fatalf("run failed: %+v", rr.Summary())
	}
	fmt.Println(rr.Summary())
}
```

## What is not implemented

`include_tasks`/`import_tasks`/`include_role`/`import_role` resolve
**statically at parse time**, not real Ansible's dynamic, per-host,
possibly-templated resolution — documented as narrower, not silently
different. The namespace/collection metadata system, Python-style inventory
*plugins*, and lookup/callback plugins are out of scope entirely (executable
inventory *scripts* — the `--list`/`--host` protocol — are supported; see
[inventory](inventory.md)). Any playbook `strategy` other than `linear`/
`free` (`debug`, `host_pinned`, a strategy plugin) is rejected with an
explicit parse error rather than silently treated as `linear`.
`async`/`poll` only genuinely backgrounds `command`/`shell` on the target
— the only two modules whose entire work reduces to one remote invocation —
and an overrunning job is detected on timeout but not actively killed, since
real Ansible's `killpg` has no portable POSIX equivalent without `setsid`
(absent on macOS). `run_once` broadcasts its `register:` value across a
batch but is not coordinated across `strategy: free`'s independent per-host
lanes.
