# Roadmap

## Shipped

Four components are real, tested Go libraries with CI running on every push
and pull request:

- [`vault`](components/vault.md) — Ansible Vault 1.1 compatible AES-256
  encryption
- [`inventory`](components/inventory.md) — INI/YAML inventory parsing, group
  ancestry, host-pattern matching
- [`vars`](components/vars.md) — the variable precedence ladder
- [`template`](components/template.md) — Jinja2-compatible templating with
  Ansible's filter and test library

## In progress

The following repositories exist in the [go-ansible](https://github.com/go-ansible)
organization but do not yet have working code — this documentation makes no
capability claims about them, and they are intentionally not linked from the
components pages above until they do:

- **`modules`** — the Ansible module execution protocol (the JSON-over-stdin/stdout
  contract a module implements) plus the core module library
- **`playbook`** — the playbook/role/task/handler engine: loops, conditionals,
  blocks, strategies. This is the piece that will actually drive `vars` and
  `template` together against a real play, rather than each being exercised
  standalone as they are today.
- **`facts`** — fact gathering, the Go equivalent of the `setup` module
- **`galaxy`** — role and collection installer, `requirements.yml`, a Galaxy
  API client
- **`cli`** — the `ansible`, `ansible-playbook`, `ansible-vault`, and
  `ansible-galaxy` binaries, once there is an engine underneath them worth
  shipping a CLI for

## Design principles carried through every component

- **Byte/wire compatible, not merely similar.** `vault` reproduces
  `ansible.parsing.vault.VaultAES256`'s exact format; a file it writes
  decrypts with the real `ansible-vault` and vice versa. Every future
  component holds itself to the same bar against the real `ansible-core`
  behavior, not against this project's own prior guesses.
- **Pure Go, `CGO_ENABLED=0`.** No component here, or planned, depends on a
  C library or an external interpreter.
- **Each concern its own module.** `vault` has no dependency on the other
  three; a program that only needs Vault-compatible decryption does not pull
  in a Jinja2 engine it will never call. The same separation is intended for
  `modules`, `playbook`, `facts`, and `galaxy` as they land.
