# inventory

[![CI](https://github.com/go-ansible/inventory/actions/workflows/ci.yml/badge.svg)](https://github.com/go-ansible/inventory/actions/workflows/ci.yml)

`github.com/go-ansible/inventory` parses Ansible-compatible inventories —
both **YAML** and **INI** — into the group/host graph, including
`group_vars`/`host_vars` directories, and matches Ansible's host-pattern
syntax against it.

## Data model

```go
type Host struct {
	Name string
	Vars map[string]any
}

type Group struct {
	Name     string
	Hosts    map[string]*Host
	Children map[string]*Group
	Parents  map[string]*Group
	Vars     map[string]any
}

type Inventory struct {
	Hosts  map[string]*Host
	Groups map[string]*Group
}
```

`New()` returns an inventory pre-seeded with the two groups Ansible always
has: `all` (every host) and `ungrouped` (every host in no other group) —
computed the way Ansible computes it: a host only reaches `ungrouped` (and
`all`) if it belongs to no group other than those two.

## Loading

```go
func Load(path string) (*Inventory, error)
```

`path` may be a single YAML file (`.yml`/`.yaml`), a single INI file (any
other extension, or none), or a directory — in which case every regular file
directly inside it (except `group_vars/` and `host_vars/`) is parsed and
merged in name order, and then `group_vars/<name>.yml` (or a
`group_vars/<name>/*.yml` directory) and the equivalent `host_vars/` siblings
are merged in, **group vars before host vars**, matching Ansible's precedence.

## Dynamic inventory scripts

`Load` detects an executable `path` (`IsScript`, checking the file mode's
executable bit) and runs it as a dynamic inventory script instead of parsing
it as YAML/INI, against the real script protocol: `path --list` for the full
group/host graph, `path --host <name>` for one host's vars when a group
entry doesn't already provide them via the `_meta.hostvars` optimization.
Each group in `--list`'s output may be either the shorthand array of
hostnames or the full `{hosts, vars, children}` object form — both parse.
`group_vars`/`host_vars` directories are not consulted for a script-backed
inventory, matching real Ansible.

## Group ancestry and merge order

```go
func (inv *Inventory) GroupsForHost(hostName string) []*Group
```

Returns every group a host belongs to, directly or through a parent group,
ordered `all` first and the most specific group last — the order Ansible
merges group vars in, so that a child group's value for a key overrides its
parent's.

## Host-pattern matching

```go
func (inv *Inventory) Match(pattern string) ([]*Host, error)
```

Resolves Ansible's host-pattern language: `all`, a literal group or host
name, a glob, a numeric or alphabetic range (`web[01:50]`), and
colon/comma-separated combinations with `!` exclusion and `&` intersection —
e.g. `webservers:!web3:&datacenter1` reads as "every host in `webservers`
that is also in `datacenter1`, except `web3`." Results are sorted by name for
deterministic output.

## What this does not do

`inventory` only builds and queries the graph — it does not itself decide
variable precedence between a host's group vars, its own vars, and anything
from a play or the command line. That ladder lives in
[`vars`](vars.md), which consumes `inventory`'s output as one of its layers.
