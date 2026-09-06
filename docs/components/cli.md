# cli

[![CI](https://github.com/go-ansible/cli/actions/workflows/ci.yml/badge.svg)](https://github.com/go-ansible/cli/actions/workflows/ci.yml)

`github.com/go-ansible/cli` ships all **8 real Ansible command-line tools**,
each a thin `run(args []string) int` wired directly onto
[`playbook`](playbook.md), [`inventory`](inventory.md), [`vault`](vault.md)
and [`modules`](modules.md) — no shelling out to a Python `ansible-core`
underneath, no subprocess boundary at all.

| Binary | What it does |
|---|---|
| `ansible` | Ad-hoc module execution against a pattern (`ansible all -m ping`) |
| `ansible-playbook` | Runs a playbook via [`playbook.Engine`](playbook.md) |
| `ansible-vault` | Encrypt/decrypt/view/edit/rekey — wraps [`vault`](vault.md) |
| `ansible-galaxy` | Installs a role from a git URL via [`go-git`](https://github.com/go-git/go-git) — **no galaxy.ansible.com HTTP API**, out of scope |
| `ansible-pull` | Clones/pulls a git repo and runs a playbook from it against the local machine (pull-mode counterpart to `ansible-playbook`) |
| `ansible-doc` | Prints each module's own Go doc comment via a build-time codegen tool — real content, not real Ansible's structured `DOCUMENTATION` YAML |
| `ansible-config` | `list`/`dump`/`view` — real `ANSIBLE_*` environment variable support, no `ansible.cfg` file parsing |
| `ansible-console` | Interactive REPL: `cd PATTERN`, `list`, `become`/`nobecome`, runs any registered module by name or a bare shell command |

## Building

Each binary lives at `cmd/<name>` and builds the ordinary way:

```sh
go build ./cmd/ansible-playbook
./ansible-playbook -i inventory.ini site.yml
```

`ansible`'s pattern-first argument convention (`ansible all -m ping`, not
`ansible -m ping all`) doesn't fit Go's standard `flag` package, which
expects flags before positional arguments — a small `extractPattern`
preprocessor in `cmd/ansible/main.go` pulls the one non-flag token out before
handing the rest to `flag`, so both argument orderings work as they do with
real `ansible`.

## Container image

`cli` also publishes a multi-arch `FROM scratch` OCI image,
[`ghcr.io/go-ansible/cli`](https://github.com/go-ansible/cli/pkgs/container/cli),
bundling all 8 binaries, on every version tag — all six 64-bit architectures
(amd64/arm64/riscv64/loong64/ppc64le/s390x). The build stage cross-compiles
from the runner's own native architecture instead of running under QEMU for
every target, which is what makes loong64 possible at all: the official
`golang` image itself publishes no `linux/loong64` manifest. A `CGO_ENABLED=0`
Go binary needs no libc and no interpreter — real Ansible fundamentally
cannot reach `scratch`, since it needs Python plus several pip-installed
packages just to be the controller. See
[BENCHMARKS.md](https://github.com/go-ansible/.github/blob/main/BENCHMARKS.md)
for the measured comparison.

**Known limitation, not hidden**: `command`/`shell` (and any module that
execs a shell on a `local` connection) need `/bin/sh`, which does not exist
in a scratch image. `debug`/`copy`/`template`/`set_fact` and friends work; a
playbook that also targets a real host over SSH is unaffected either way,
since the shell requirement is the *target's*, not this image's.

## What is not implemented

`ansible-galaxy` here only clones a role from a git URL — no
galaxy.ansible.com API client, no collection installs (a standalone `galaxy`
repository was planned early on for this; the actual implementation turned
out simple enough to fold directly into `cli` instead, and the standalone
repository was never developed further). `ansible-doc` prints this port's own
Go doc comments, not real Ansible's structured `DOCUMENTATION`/`EXAMPLES`/
`RETURN` YAML blocks. `ansible-config view` always fails honestly — there is
no `ansible.cfg` file support at all.
