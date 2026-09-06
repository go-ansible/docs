# Component overview

Ansible's own engine wires several distinct concerns together at run time: it
reads an inventory, gathers facts, resolves variables through a fixed
precedence order, renders Jinja2 templates against the result, and — via
modules — carries out the actual work, all orchestrated by the playbook/role/
task engine. go-ansible keeps that same separation, one Go module per
concern, so each piece is independently importable and independently tested
against Ansible's real behavior rather than against a reimplementation of a
reimplementation.

```
inventory  →  hosts, groups, group_vars/host_vars
     │                            │
     │                            ▼
     │                 facts  →  gathers a target's system facts
     │                            (the `setup` module equivalent)
     │                            │
     ▼                            ▼
   vars    →  merges inventory vars with facts, play/role/task vars,
              and extra-vars through Ansible's fixed precedence ladder
                        │
                        ▼
 template  →  renders {{ }} / {% %} against the merged variable context,
              using Ansible's filter and test library
                        │
                        ▼
 modules   →  executes a task's module (563 registered) against a
              real target connection, using the rendered arguments
                        │
                        ▼
 playbook  →  the engine tying all of the above together: per-host
              execution, blocks/rescue/handlers/roles/strategies
                        │
                        ▼
   cli     →  all 8 real ansible-* binaries, thin wrappers over playbook
```

Each component is still usable on its own — `vault` has no dependency on the
other seven, and `inventory`/`vars`/`template` can be exercised together
without ever touching `modules` or `playbook` — so a program that only needs
Vault-compatible decryption, say, does not pull in a Jinja2 engine or a
playbook engine it will never call.

| Component | Depends on | Status |
|---|---|---|
| [`vault`](vault.md) | — | Real, tested, CI |
| [`inventory`](inventory.md) | — | Real, tested, CI |
| [`vars`](vars.md) | — (consumes `inventory`'s output, but has no import dependency on it) | Real, tested, CI |
| [`template`](template.md) | `go-regexp/engine`, `nikolalohinski/gonja/v2` | Real, tested, CI |
| [`facts`](facts.md) | `go-remoteexec/transport` | Real, tested, CI |
| [`modules`](modules.md) | `go-remoteexec/transport` | Real, tested, CI — 563 modules |
| [`playbook`](playbook.md) | `inventory`, `vars`, `template`, `modules`, `facts`, `go-remoteexec/transport` | Real, tested, CI — the engine tying the rest together |
| [`cli`](cli.md) | `playbook`, `vault`, `inventory`, `go-git` | Real, tested, CI — all 8 `ansible-*` binaries |

All eight are shipped and tagged. See [Roadmap](../roadmap.md) for what's
still ahead beyond the current module/collection surface, and the
[engine feature matrix](https://go-ansible.github.io/) on the landing page
for exactly which playbook directives are implemented today.
