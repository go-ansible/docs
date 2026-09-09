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

	BaseDir   string
	RunTags   []string
	SkipTags  []string
	OnResult  func(Result)
	Callbacks []Callback
}

// Callback is a reporting plugin — see "Callback plugins" below.
type Callback interface {
	OnPlayStart(play Play)
	OnTaskResult(r Result)
	OnStats(rr *RunResult)
}

type BaseCallback struct{} // no-op defaults; embed and override what you need

func NewDefaultCallback(w io.Writer, color bool) *DefaultCallback

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

## Callback plugins

Real Ansible reports a run through **callback plugins**
(`ansible.plugins.callback.CallbackBase` and its `v2_*` hooks), loading one
`stdout`-type plugin alongside any number of `notification`-type ones.
`Callback` is this port's equivalent, and `Engine.Callbacks` is a list for
that same reason. Embed `BaseCallback` to implement only the hooks you care
about, exactly the way a real callback plugin overrides only the `v2_*`
methods it needs.

`DefaultCallback` is this port's `ansible.builtin.default`: PLAY and TASK
banners, a colored line per result, and a PLAY RECAP. It is what
[`cli`](cli.md)'s `ansible-playbook` and `ansible-pull` install, and it
serializes its own hooks — results genuinely arrive from one goroutine per
host, so an implementation that keeps state or writes to a shared stream
must do the same. Its banners are not padded out with asterisks the way
real Ansible's `Display.banner` pads them to the terminal width.

Three hooks, where real Ansible has 24, because a hook nothing in this port
can raise and nothing can consume would be an empty promise:

| hook | real equivalent |
| --- | --- |
| `OnPlayStart` | `v2_playbook_on_play_start` |
| `OnTaskResult` | `v2_runner_on_ok`/`_failed`/`_skipped` collapsed into one, since `Result` already carries which it is |
| `OnStats` | `v2_playbook_on_stats` — raised once per `RunPlaybook` call, **including when a play errors out**, so a recap still covers whatever did run |

Two absences worth naming, both disclosed rather than stubbed:
`v2_playbook_on_start` prints nothing at real Ansible's own default
verbosity (it only fires above `-v`), and this port has no verbosity concept
to gate it on; and handler runs are indistinguishable from ordinary task
runs here, so there is nothing to raise real Ansible's separate
`v2_playbook_on_handler_task_start` from.

`Engine.OnResult` remains as the one-hook shorthand for a caller that only
wants results and no play or recap events.

## Measured against real Ansible

The engine's control flow has been run side by side with **real
ansible-core 2.21.4**: the same playbook through both, across two hosts,
appending markers to a log so that *ordering* is compared and not just
outcomes — which is what `block`/`rescue`/`always` and handler timing
actually turn on.

Most of it held on the first run: block/rescue/always ordering and
membership, a handler firing exactly once for two notifying tasks, a
handler correctly *not* firing for an unchanged task, `run_once` running
once across both hosts, `changed_when`/`failed_when`/`ignore_errors`, and
every `when:` form. Two things did not, both about loops, and both are
fixed:

- **`register` on a looped task produced no `results` list**, so
  `{{ r.results | map(attribute='stdout') }}` — everyday usage — failed
  outright. A looped task now registers exactly what real Ansible
  registers: `changed`, `failed`, `msg` (`"All items completed"`) and
  `results`, with **none** of the module's own fields at the top level,
  and one entry per iteration carrying the module fields plus `item` and
  `ansible_loop_var`. A task that is not looping still registers its
  fields flat.
- **`loop_control.index_var` was parsed nowhere** and rendered empty. It
  is now bound per iteration, 0-based.

Two differences are known and not yet addressed:

- `command` results lack the `start`/`end`/`delta` timing fields real
  Ansible includes.
- The PLAY RECAP has no `unreachable`/`rescued`/`ignored` columns, so a
  failure that was rescued or ignored is still counted under `failed`
  where real Ansible separates them.

Real ansible-core 2.21 also **requires a conditional to evaluate to a
boolean** — a `when:` yielding a dict or list is an error there
(`ALLOW_BROKEN_CONDITIONALS` relaxes it), where this engine applies
ordinary truthiness. That is a deliberate upstream tightening rather than
a defect here, recorded so the difference is known.

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
