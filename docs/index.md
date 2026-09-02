# go-ansible documentation

**A pure-Go, `CGO_ENABLED=0`, functional-parity port of [Ansible](https://www.ansible.com/).**

go-ansible reimplements Ansible's engine and modules as Go libraries instead of
wrapping the Python `ansible-core` binary. No Python interpreter, no C
extensions, no `pip install` — each component is an importable Go package that
compiles into your own static binary, and the wire formats it reads and writes
are byte-compatible with the real `ansible-core` tooling.

## What exists today

Four components are real, tested Go libraries with CI:

| Component | What it does |
|---|---|
| [`vault`](components/vault.md) | Reads and writes [Ansible Vault](https://docs.ansible.com/ansible/latest/vault_guide/index.html) 1.1 files — AES-256-CTR with a PBKDF2-HMAC-SHA256 key and an encrypt-then-MAC tag, byte-compatible with `ansible-vault` |
| [`inventory`](components/inventory.md) | Parses Ansible-compatible INI and YAML inventories into the group/host graph, including `group_vars`/`host_vars` and host-pattern matching |
| [`vars`](components/vars.md) | Ansible's variable precedence ladder — the fixed merge order from role defaults up through `-e`/`--extra-vars` |
| [`template`](components/template.md) | Jinja2-compatible templating with Ansible's filter and test library layered on top, including Ansible's native-type rendering rule for a bare `{{ expr }}` |

See the [component overview](components/index.md) for how these fit together,
or [Roadmap](roadmap.md) for what's still ahead — module execution, the
playbook/role engine, fact gathering, a Galaxy client, and the `ansible-*` CLI
binaries are in progress in their own repositories and are not yet part of
this documentation.

## Why pure Go

Being pure Go buys three things Ansible's own Python implementation cannot:

- **A single static binary.** No interpreter to install, no virtualenv, no
  `ansible[core]` version pinned against a specific Python. Cross-compile once,
  ship one file.
- **CGO-free by construction.** Every dependency here is pure Go — vault's
  crypto comes from `golang.org/x/crypto`, not a C OpenSSL binding — so the
  usual cross-compilation and static-linking headaches don't apply.
- **Byte-for-byte compatibility, not a reinterpretation.** `vault` reproduces
  the exact wire format of `ansible.parsing.vault.VaultAES256`; files written
  by one decrypt with the other. That's the bar for every component here: not
  "does something similar," but "reads what Ansible wrote, writes what Ansible
  reads."

## Repositories

| Repo | Role |
|---|---|
| [`vault`](https://github.com/go-ansible/vault) | Ansible Vault-compatible AES256 encryption for secrets |
| [`inventory`](https://github.com/go-ansible/inventory) | Ansible-compatible inventory: INI/YAML parsers, groups, host/group vars, patterns |
| [`vars`](https://github.com/go-ansible/vars) | Ansible variable precedence engine |
| [`template`](https://github.com/go-ansible/template) | Jinja2-compatible templating with Ansible's filter and test library |
| [`brand`](https://github.com/go-ansible/brand) | Logo, favicon and social banner |
| [`docs`](https://github.com/go-ansible/docs) | This documentation, published at [go-ansible.github.io/docs](https://go-ansible.github.io/docs/) |

📖 **[go-ansible.github.io](https://go-ansible.github.io/)**
