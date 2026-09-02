# Component overview

Ansible's own engine wires several distinct concerns together at run time: it
reads an inventory, gathers facts, resolves variables through a fixed
precedence order, renders Jinja2 templates against the result, and — via
modules — carries out the actual work. go-ansible keeps that same separation,
one Go module per concern, so each piece is independently importable and
independently tested against Ansible's real behavior rather than against a
reimplementation of a reimplementation.

```
inventory  →  hosts, groups, group_vars/host_vars
                        │
                        ▼
   vars    →  merges inventory vars with facts, play/role/task vars,
              and extra-vars through Ansible's fixed precedence ladder
                        │
                        ▼
 template  →  renders {{ }} / {% %} against the merged variable context,
              using Ansible's filter and test library
```

Each component is usable on its own — `vault` has no dependency on the other
three, and `inventory` and `vars` can be used together without `template` — so
a program that only needs Vault-compatible decryption, say, does not pull in a
Jinja2 engine it will never call.

| Component | Depends on | Status |
|---|---|---|
| [`vault`](vault.md) | — | Real, tested, CI |
| [`inventory`](inventory.md) | — | Real, tested, CI |
| [`vars`](vars.md) | — (consumes `inventory`'s output, but has no import dependency on it) | Real, tested, CI |
| [`template`](template.md) | `go-regexp/engine`, `nikolalohinski/gonja/v2` | Real, tested, CI |

What is not yet built — module execution, the playbook/role/task engine that
would actually drive `vars` and `template` together, fact gathering, and the
`ansible-*` CLI binaries — is tracked in [Roadmap](../roadmap.md).
