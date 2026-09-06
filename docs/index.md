# go-ansible documentation

**A pure-Go, `CGO_ENABLED=0`, functional-parity port of [Ansible](https://www.ansible.com/).**

go-ansible reimplements Ansible's engine and modules as Go libraries instead of
wrapping the Python `ansible-core` binary. No Python interpreter, no C
extensions, no `pip install` — each component is an importable Go package that
compiles into your own static binary, and the wire formats it reads and writes
are byte-compatible with the real `ansible-core` tooling.

## What exists today

All eight components are real, tested Go libraries with CI running on every
push and pull request:

| Component | What it does |
|---|---|
| [`vault`](components/vault.md) | Reads and writes [Ansible Vault](https://docs.ansible.com/ansible/latest/vault_guide/index.html) 1.1 files — AES-256-CTR with a PBKDF2-HMAC-SHA256 key and an encrypt-then-MAC tag, byte-compatible with `ansible-vault` |
| [`inventory`](components/inventory.md) | Parses Ansible-compatible INI and YAML inventories into the group/host graph, including `group_vars`/`host_vars` and host-pattern matching |
| [`vars`](components/vars.md) | Ansible's variable precedence ladder — the fixed merge order from role defaults up through `-e`/`--extra-vars` |
| [`template`](components/template.md) | Jinja2-compatible templating with Ansible's filter and test library layered on top, including Ansible's native-type rendering rule for a bare `{{ expr }}` |
| [`facts`](components/facts.md) | Fact gathering — the `setup` module equivalent — in one shell round trip, no Python required |
| [`modules`](components/modules.md) | The Ansible module execution protocol plus **561 modules**: all of `ansible.builtin` and `ansible.posix`, and a curated 485 of `community.general` |
| [`playbook`](components/playbook.md) | The playbook/role/task/handler engine tying all of the above together against a real play — loops, conditionals, blocks, `become`, roles, both execution strategies, retries, `run_once`, forks, `vars_prompt`, `async`/`poll`, dynamic inventory, `ansible.cfg` |
| [`cli`](components/cli.md) | All 8 real `ansible-*` binaries, each a thin wrapper over `playbook`/`vault`/`inventory` |

See the [component overview](components/index.md) for how these fit together,
[Roadmap](roadmap.md) for what's still ahead beyond today's module/collection
surface, and the **[engine feature matrix](https://go-ansible.github.io/)**
on the landing page for exactly which playbook directives are implemented,
re-checked against the code rather than assumed.

## Why pure Go

Being pure Go buys three things Ansible's own Python implementation cannot:

- **A single static binary.** No interpreter to install, no virtualenv, no
  `ansible[core]` version pinned against a specific Python. Cross-compile once,
  ship one file.
- **CGO-free by construction.** Every dependency here is pure Go — vault's
  crypto comes from `golang.org/x/crypto`, not a C OpenSSL binding — so the
  usual cross-compilation and static-linking headaches don't apply. `cli`
  publishes a multi-arch `FROM scratch` OCI image on every version tag,
  something real Ansible's own Python interpreter and pip dependencies can
  never reach.
- **Byte-for-byte compatibility, not a reinterpretation.** `vault` reproduces
  the exact wire format of `ansible.parsing.vault.VaultAES256`; files written
  by one decrypt with the other. That's the bar for every component here: not
  "does something similar," but "reads what Ansible wrote, writes what Ansible
  reads" — verified against a real installed `ansible-core`, not just internal
  review.

## Repositories

| Repo | Role |
|---|---|
| [`vault`](https://github.com/go-ansible/vault) | Ansible Vault-compatible AES256 encryption for secrets |
| [`inventory`](https://github.com/go-ansible/inventory) | Ansible-compatible inventory: INI/YAML parsers, groups, host/group vars, patterns |
| [`vars`](https://github.com/go-ansible/vars) | Ansible variable precedence engine |
| [`template`](https://github.com/go-ansible/template) | Jinja2-compatible templating with Ansible's filter and test library |
| [`facts`](https://github.com/go-ansible/facts) | Fact gathering, pure Go CGO=0 |
| [`modules`](https://github.com/go-ansible/modules) | Module execution protocol plus the 561-module core library |
| [`playbook`](https://github.com/go-ansible/playbook) | Playbook/task/handler execution engine |
| [`cli`](https://github.com/go-ansible/cli) | All 8 CLI binaries |
| [`brand`](https://github.com/go-ansible/brand) | Logo, favicon and social banner |
| [`docs`](https://github.com/go-ansible/docs) | This documentation, published at [go-ansible.github.io/docs](https://go-ansible.github.io/docs/) |

The low-level SSH/local/`become` connection layer lives outside this org, in
the project-neutral
[`go-remoteexec/transport`](https://github.com/go-remoteexec/transport)
(shared with `go-puppet-bolt/bolt`); `modules`, `playbook`, and `cli` depend
on it directly.

📖 **[go-ansible.github.io](https://go-ansible.github.io/)**
