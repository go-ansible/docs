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
| `ansible-playbook` | Runs a playbook via [`playbook.Engine`](playbook.md); `-f`/`--forks` sets the concurrency cap, and `vars_prompt:` prompts at a real terminal (masked input for `private` vars, falling back to defaults when not a TTY); `--check`/`-C` predicts changes without making them and `--diff`/`-D` shows them as unified diffs |
| `ansible-inventory` | Dumps the parsed inventory: `--list` (JSON), `--host` (one host's merged vars), `--graph` (tree), with `--export`, `--vars`, `--limit` and `--output` — see [below](#ansible-inventory) |
| `ansible-vault` | Encrypt/decrypt/view/edit/rekey — wraps [`vault`](vault.md) |
| `ansible-galaxy` | `install` a role from a git URL via [`go-git`](https://github.com/go-git/go-git), plus `list` and `remove` of installed roles — **no galaxy.ansible.com HTTP API**, out of scope |
| `ansible-pull` | Clones/pulls a git repo and runs a playbook from it against the local machine (pull-mode counterpart to `ansible-playbook`) |
| `ansible-doc` | Prints each module's own Go doc comment via a build-time codegen tool — real content, not real Ansible's structured `DOCUMENTATION` YAML |
| `ansible-config` | `list`/`dump`/`view` — reads real `ANSIBLE_*` environment variables and, for the settings this port supports, an actual `ansible.cfg` `[defaults]` section (host var > env var > `ansible.cfg` > compiled default; `$ANSIBLE_CONFIG` > `./ansible.cfg` > `~/.ansible.cfg` > `/etc/ansible/ansible.cfg`, first found wins outright) |
| `ansible-console` | Interactive REPL with real's prompt and banner: `cd`, `list`, `become`/`nobecome`, `forks`, `remote_user`, `verbosity`, and any registered module by name or a bare shell command — see [below](#ansible-console) |

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
bundling every binary, on every version tag — all six 64-bit architectures
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

## Diagnostics, and an inventory that cannot be read

Diagnostics use real Ansible's prefixes on **stderr**: `[ERROR]: ` for
something that stops the run, `[WARNING]: ` for something that does not.

An unusable inventory is the second kind. Real never dies because of the
inventory — an absent, unreadable or unparseable source warns and the run
continues with only the implicit localhost, **exiting 0** — and so does no
`-i` at all, which is why `ansible localhost -m ping` works there with no
inventory. This port behaves the same way; it used to exit 1 and to require
`-i`, which broke the script around a drop-in replacement.

Four independent warnings, each with its own condition:

| warning | fires when |
|---|---|
| `Failed to parse inventory: <cause>` | the source exists and parsing failed |
| `Unable to parse <abspath> as an inventory source` | the source is unusable |
| `No inventory was parsed, only implicit localhost is available` | no source parsed at all |
| `provided hosts list is empty, only localhost is available. …` | the inventory has no hosts |

They are not one warning in four places. An **empty `.ini` parses**, so it
gets the last one only; a **missing** one gets the middle two but no cause
line, because real's plugins report a parse failure only once they have
actually tried to parse. The path is reported absolute even when `-i` was
given a relative one. The last warning is suppressed when the host pattern is
one the implicit localhost answers to, so ad-hoc `ansible -i /nope.ini
localhost -m ping` prints two warnings where the same command with a pattern
of `h1` prints three.

A host pattern that matches nothing is likewise a warning and not a failure,
for a play's `hosts:` and for ad-hoc alike. `--limit` is the exception, and
real's own: a limit that leaves the whole inventory with nothing to target is
`[ERROR]: Specified inventory, host pattern and/or --limit leaves us with no
hosts to target.` and exit 1 — but only when the inventory was not empty to
begin with.

A **directory** is an unusable source when it holds no inventory file at all —
every entry a subdirectory, `group_vars`/`host_vars`, or a dotfile. It is
named like a missing file, with no cause line, because real never reached a
parser for it. A directory holding an *empty* file still parses: it is the
directory that must hold a source, not the source that must hold hosts.

One difference from real remains here. A source that fails to parse gets a
one-line cause where real prints a nine-line block naming the plugin it tried,
a source position and an excerpt, which needs per-node positions this port's
parsers do not record.

## ansible-inventory

`--list`, `--host` and `--graph`, with `--export`, `--vars`, `--limit` and
`--output`. The output shapes follow real's, which are more particular than
they look:

- a group is emitted only if it has **hosts, children, or (with `--export`)
  vars** — which is why an empty `ungrouped` never appears although it always
  exists;
- a host with **no variables at all** is left out of `_meta.hostvars` rather
  than carried as an empty object;
- without `--export`, a host carries its **merged** vars and groups show none;
  with it, a host carries only its **own** and groups keep theirs;
- `--limit` filters hosts but **not structure**: a child stays listed even when
  the limit emptied it, while the emptied group loses its own entry;
- `--graph` prints children first, then hosts, then vars;
- **no action at all exits 5**, not 1 or 2;
- orders are **document order, not alphabetical** — `"all": {"children":
  ["ungrouped", "prod"]}`, `"prod": {"children": ["web", "db"]}` for a
  `[prod:children]` section written web-then-db.

Unlike `ansible-playbook`, it prints **two** warnings for an unusable
inventory rather than three: the "provided hosts list is empty" one comes from
real's own `get_host_list`, which this binary never calls because it resolves
no host list of its own.

`--yaml` and `--toml` are real output formats that are **not implemented**.
They are refused explicitly, so a command line asking for one fails rather
than silently receiving JSON.

One difference from real: `--host localhost` omits `ansible_python_interpreter`,
which real reports for the implicit localhost. It is a path to a Python this
port never runs, so stating one would be a claim about the target that is not
true here.

## ansible-galaxy list and remove

Neither needs the Galaxy API — they read and delete directories — so both are
implemented, and match real across the eight invocations they were measured
on. Three rules are worth knowing because they are not what you would guess:

- a directory is a role only when it holds `meta/main.yml`; a plain directory
  is skipped, and so is one carrying only a `meta/.galaxy_install_info`;
- the version shown comes from `meta/.galaxy_install_info`, which the
  installer writes — a role declaring `galaxy_info.version` in `meta/main.yml`
  still lists as `(unknown version)`;
- `--roles-path` does **not** replace the default search path
  (`~/.ansible/roles`, `/usr/share/ansible/roles`, `/etc/ansible/roles`), it
  goes in front of it, which is why `list -p somewhere` still warns about the
  defaults that do not exist.

Roles are listed in the order the filesystem returns them, not sorted, because
that is what real does.

## ansible-console

The prompt is real's, and carries four things — the user the console would
connect as, the current pattern, how many hosts it matches, and the fork
count. It ends in `#` rather than `$` while become is on, the way a root
shell prompt does:

```
david@all (3)[f:5]$
david@all (3)[f:5]#     under become
```

A banner is printed at startup and `Ansible-console was exited.` when the
session ends, including when the input simply runs out — which is what a
piped script does.

Two behaviours are worth knowing because the obvious guess is wrong, and both
are real's:

- a bare `become` is **refused** with a request for a value rather than
  toggling, since the prompt already shows the state. Anything not plainly
  affirmative turns it off, so a typo cannot leave a session escalated;
- a bare `cd` goes to `*`, not to `all`.

Two differences from real remain: its `help` lists every module alongside the
built-in commands, and a failing command prints the structured `[ERROR]` block
this port cannot reproduce without per-node source positions.

## What is not implemented

`ansible-galaxy` here only clones a role from a git URL — no
galaxy.ansible.com API client, no collection installs (a standalone `galaxy`
repository was planned early on for this; the actual implementation turned
out simple enough to fold directly into `cli` instead, and the standalone
repository was never developed further). `ansible-doc` prints this port's own
Go doc comments, not real Ansible's structured `DOCUMENTATION`/`EXAMPLES`/
`RETURN` YAML blocks. `ansible-config`'s `ansible.cfg` support covers only the
settings this port itself reads (see [`ConfigDefaults`](playbook.md)) — an
unrecognized-but-valid Ansible setting in the file is silently ignored rather
than surfaced, matching how unset environment variables were already
treated.
