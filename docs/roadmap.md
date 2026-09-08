# Roadmap

## Shipped

All eight planned components are real, tested Go libraries with CI running
on every push and pull request:

- [`vault`](components/vault.md) — Ansible Vault 1.1 compatible AES-256
  encryption
- [`inventory`](components/inventory.md) — INI/YAML inventory parsing, group
  ancestry, host-pattern matching
- [`vars`](components/vars.md) — the variable precedence ladder
- [`template`](components/template.md) — Jinja2-compatible templating with
  Ansible's filter and test library
- [`facts`](components/facts.md) — fact gathering, the `setup` module
  equivalent
- [`modules`](components/modules.md) — the module execution protocol plus
  **566 registered modules** (all of `ansible.builtin` and `ansible.posix`,
  and a curated 490 of `community.general`)
- [`playbook`](components/playbook.md) — the playbook/role/task/handler
  engine driving all of the above against a real play: loops, conditionals,
  blocks with genuine per-host recovery, `become`, roles (nested variable
  scoping composes to any depth), both the `linear` and `free` execution
  strategies, `until`/`retries`/`delay`, `run_once`, a real `Forks`
  concurrency cap, `vars_prompt`, `async`/`poll` (for `command`/`shell`),
  fully-qualified collection names, dynamic inventory scripts, and
  `ansible.cfg` file support
- [`cli`](components/cli.md) — all 8 real `ansible-*` binaries, plus a
  multi-arch `FROM scratch` OCI image on every version tag

A standalone `galaxy` repository was planned early on for a role/collection
installer with its own Galaxy API client; the actual need turned out simple
enough (clone a role from a git URL via `go-git`) to fold directly into
`cli`'s `ansible-galaxy` binary instead, so that repository was never
developed further and is not part of the current architecture.

## What's still ahead

- **~87 more `community.general` modules.** The remainder is dominated by
  named flagship platforms confirmed to have no comparable official CLI
  (UTM, OneView, ManageIQ, PagerDuty, Datadog, Slack, WDC's own Redfish
  family), confirmed-dead or no-CLI platforms, defunct/EOL products, and pure
  notification-protocol modules with no CLI concept at all (IRC/Jabber/
  Matrix/Telegram/Discord/...). A platform excluded today is not excluded
  forever — several exclusions were later reversed after that platform
  shipped a genuine official CLI (Huawei Cloud's KooCLI, HPE's `ilorest`,
  Lenovo's `OneCli`, among others).
- **The rest of `redfish_command`/`redfish_config`/`redfish_info`.** The
  vendor-neutral trio now ships via DMTF's own `redfishtool` (its `-c
  cfgFile` option reads credentials from a file rather than argv), covering
  real Systems/Chassis power, boot override, indicator LED, sessions,
  account management, and Manager power/logs/network-protocol/host-interface
  config, plus (in `redfish_info`) inventory, config, and health-report info
  across all 7 real categories — Systems, Chassis, Accounts, Sessions,
  Update, Manager, and Service. Manager and Chassis are now complete (every
  real command wired), Systems has 13 of its 14 real commands, and Update
  has 3 of 4. Real `redfish_command.py` alone declares ~35 commands across
  6 categories, and `redfish_info` a further ~39, so real gaps remain in
  each: virtual media (needs vendor-specific empty-slot matching this port
  has no hardware to verify against), storage/RAID configuration, and two
  specific commands genuinely unreachable through further CLI-substitution
  work — `GetBiosRegistries` (needs vendor-aware HPE iLO4/iLO5 workarounds
  this port has no hardware to verify against) and `GetUpdateStatus`
  (blocked on an architectural ceiling: redfishtool's own `raw` subcommand
  exposes only a response's JSON body, never the distinguishing HTTP status
  code real Ansible's own status logic depends on) — disclosed, not
  silently assumed to work.
- **Cloud-provider collections** — `amazon.aws`, `azure.azcollection`,
  `google.cloud`, and similar. These need real Go SDK bindings per provider's
  REST API, not shell composition over a CLI the way the rest of this port
  works — a fundamentally different kind of work, not yet started, and not
  scoped without a fresh decision to take it on.
- **Real Ansible's structured module documentation** (`DOCUMENTATION`/
  `EXAMPLES`/`RETURN` YAML) — `ansible-doc` here prints this port's own Go
  doc comments instead, real content in a different shape.
- **The namespace/collection metadata system**, real
  `ansible-galaxy collection install` (this port's `ansible-galaxy` only
  clones a role from a git URL — no galaxy.ansible.com API), and
  lookup/callback plugins.
- **Active kill-on-timeout for `async`/`poll`.** An overrunning backgrounded
  job is detected but not killed — real Ansible's `async_wrapper.py` sends
  `killpg` to the whole process group, and there is no portable POSIX
  equivalent without `setsid`, which macOS doesn't have.

See the **[engine feature matrix](https://go-ansible.github.io/)** on the
landing page for the current, code-checked status of every playbook
directive, and the [org profile](https://github.com/go-ansible) for the full,
itemized list of `community.general` inclusions and exclusions with their
reasons.

## Design principles carried through every component

- **Byte/wire compatible, not merely similar.** `vault` reproduces
  `ansible.parsing.vault.VaultAES256`'s exact format; a file it writes
  decrypts with the real `ansible-vault` and vice versa. Every component
  holds itself to the same bar against the real `ansible-core` behavior, not
  against this project's own prior guesses — cross-validated against a real
  installed `ansible-core` wherever that's possible.
- **Pure Go, `CGO_ENABLED=0`.** No component here depends on a C library or
  an external interpreter.
- **Each concern its own module.** `vault` has no dependency on the other
  seven; a program that only needs Vault-compatible decryption does not pull
  in a Jinja2 engine or a playbook engine it will never call.
- **Fail loud, don't silently approximate.** Where this port's architecture
  cannot reach real Ansible's behavior — `ansible.posix.synchronize`'s
  controller-side rsync, a SaaS platform with no official CLI — the module or
  feature fails with an explicit, honest error rather than a plausible-
  looking partial implementation.
