# facts

[![CI](https://github.com/go-ansible/facts/actions/workflows/ci.yml/badge.svg)](https://github.com/go-ansible/facts/actions/workflows/ci.yml)

`github.com/go-ansible/facts` is the Go equivalent of Ansible's `setup`
module: it gathers a target's facts — OS family, distribution, architecture,
hostname, and more — in a **single shell round trip**, not by shipping and
running a Python fact-gathering script the way real Ansible does.

## API

```go
func Gather(ctx context.Context, conn remoteexec.Connection) (map[string]any, error)
```

`Gather` runs one composed shell command over `conn` and parses its output
into a flat `map[string]any` of fact names to values — the same shape
[`playbook`](playbook.md) merges via `vars.InjectFacts` into both
`ansible_facts.<name>` and the flattened `ansible_<name>` convention real
Ansible playbooks rely on.

## Example

```go
package main

import (
	"context"
	"fmt"
	"log"

	"github.com/go-ansible/facts"
	remoteexec "github.com/go-remoteexec/transport"
)

func main() {
	conn := remoteexec.NewLocal()
	gathered, err := facts.Gather(context.Background(), conn)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(gathered["os_family"], gathered["architecture"])
}
```

## What is not implemented

Real Ansible's `setup` module gathers hundreds of facts via a `facter`/`ohai`-
style plugin system with configurable `gather_subset:` filtering. This
package gathers a fixed, commonly-used core set in one round trip rather than
reproducing that plugin architecture — the facts it does return match real
Ansible's naming and values exactly, verified against a real `ansible -m
setup` run, but the *set* is narrower by design.
