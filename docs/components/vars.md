# vars

[![CI](https://github.com/go-ansible/vars/actions/workflows/ci.yml/badge.svg)](https://github.com/go-ansible/vars/actions/workflows/ci.yml)

`github.com/go-ansible/vars` implements Ansible's variable precedence: a
fixed ladder of named layers, merged low-to-high so a value set in a higher
layer always wins over the same key set in a lower one.

This package does not itself know about roles, plays, or tasks — a playbook
engine assigns each layer's content as it walks the play; `vars` only owns
the merge order.

## The precedence ladder

```go
const (
	RoleDefaults Layer = iota // role defaults/main.yml — the floor
	Inventory                 // merged inventory group_vars + host_vars
	Facts                     // gathered facts (ansible_facts.*) + flattened ansible_* aliases
	PlayVars                  // play vars:, vars_files:, vars_prompt:
	RoleVars                  // role vars/main.yml
	BlockVars                 // block-level vars:
	TaskVars                  // task-level vars:, loop item vars
	Registered                // register:, set_fact
	RoleParams                // role/include_role parameters
	ExtraVars                 // -e / --extra-vars — always wins
)
```

This is a simplified but **order-faithful** subset of the ~22 precedence
levels documented for `ansible-core`: every distinction that changes
real-world playbook behavior is kept, and the rest collapse into whichever
layer they resolve to before a task runs — for example `vars_prompt:` and
`vars_files:` both fold into `PlayVars`, since both end up as play-scoped
variables by the time a task sees them.

## API

```go
func New() *Context

func (c *Context) Set(layer Layer, vals map[string]any)
func (c *Context) SetVar(layer Layer, key string, value any)

func (c *Context) Merged() map[string]any
func (c *Context) Get(key string) (any, bool)
func (c *Context) Which(key string) (Layer, bool)

func (c *Context) Child() *Context
```

- `Merged` flattens every layer into one map, lowest precedence first, so a
  higher layer overwrites a lower one on key conflict.
- `Which` reports the highest-precedence layer currently setting a key — a
  diagnostic for "why did this variable win?", the same question Ansible's
  own `-vvv` debugging exists to answer.
- `Child` copies a `Context` for a nested scope (a block inside a play, a
  task inside a block, one iteration of a loop) so mutating the child's
  `TaskVars`/`BlockVars`/`Registered` layers never leaks back to the parent.

## Fact injection

```go
func InjectFacts(facts map[string]any) map[string]any
```

Ansible exposes gathered facts two ways at once: nested under
`ansible_facts` (the canonical form) and flattened as top-level
`ansible_<name>` aliases — so both `ansible_facts.os_family` and
`ansible_os_family` resolve in a template. `InjectFacts` produces both forms
from one fact map, ready to hand to `Set(Facts, ...)`.

## Where this fits

`vars.Context.Merged()` is the map a templating pass renders `{{ }}`
expressions against — see [`template`](template.md) — and `Inventory` is
populated from [`inventory`](inventory.md)'s `GroupsForHost`/host-vars
output before a task ever runs.
