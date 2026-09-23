# playbook

[![CI](https://github.com/go-ansible/playbook/actions/workflows/ci.yml/badge.svg)](https://github.com/go-ansible/playbook/actions/workflows/ci.yml)

`github.com/go-ansible/playbook` is the engine that drives
[`inventory`](inventory.md), [`vars`](vars.md), [`template`](template.md),
[`modules`](modules.md) and [`facts`](facts.md) together against a real play
— the piece the original [Roadmap](../roadmap.md) once described as "not yet
built." It now implements real per-host execution: `when`/`loop`/`register`,
`block`/`rescue`/`always` with genuine per-host recovery, `notify`/handlers,
`become`, roles, `include_tasks`/`import_tasks`/`include_role`/`import_role`/
`import_playbook`, `delegate_to`, `serial` batching, tag filtering, both
the `linear` (default) and `free` execution strategies, `until`/`retries`/
`delay`, `run_once`, a real `Forks` concurrency cap, `vars_prompt`, and
`async`/`poll` for `command`/`shell`.

See the **[engine feature matrix](https://go-ansible.github.io/)** on the
landing page for the current, code-checked status of every playbook
directive — that table is regenerated from `engine.go`/`playbook.go`
directly, so it stays authoritative in a way a second, hand-maintained copy
here would not.

## API

```go
type Engine struct {
	Inventory *inventory.Inventory
	Modules   *modules.Registry
	Template  *template.Engine
	ExtraVars map[string]any
	Connect   Connector

	BaseDir   string
	RunTags   []string
	SkipTags  []string
	OnResult  func(Result)
	Callbacks []Callback
}

// Callback is a reporting plugin — see "Callback plugins" below.
type Callback interface {
	OnPlayStart(play Play)
	OnTaskResult(r Result)
	OnStats(rr *RunResult)
}

type BaseCallback struct{} // no-op defaults; embed and override what you need

func NewDefaultCallback(w io.Writer, color bool) *DefaultCallback

func New(inv *inventory.Inventory) *Engine
func (e *Engine) RunPlaybook(ctx context.Context, pb Playbook) (*RunResult, error)

type Playbook []Play

func Parse(data []byte) (Playbook, error)
func ParseFile(path string) (Playbook, error)

type Result struct {
	Host, Task, Module     string
	Changed, Failed, Skipped bool
	Msg   string
	Extra map[string]any
}

type RunResult struct{ Plays []PlayResult }

func (rr *RunResult) Failed() bool
func (rr *RunResult) Summary() map[string]*HostSummary // real Ansible's PLAY RECAP, by host
```

`New` returns an `Engine` pre-wired with the built-in module registry
([`modules.Default()`](modules.md)), a fresh template engine, and
`DefaultConnect` — real SSH/local/become connections via
[`go-remoteexec/transport`](https://github.com/go-remoteexec/transport). Set
`Connect` to a custom `Connector` to run against a fake or instrumented
connection in tests.

## Callback plugins

Real Ansible reports a run through **callback plugins**
(`ansible.plugins.callback.CallbackBase` and its `v2_*` hooks), loading one
`stdout`-type plugin alongside any number of `notification`-type ones.
`Callback` is this port's equivalent, and `Engine.Callbacks` is a list for
that same reason. Embed `BaseCallback` to implement only the hooks you care
about, exactly the way a real callback plugin overrides only the `v2_*`
methods it needs.

`DefaultCallback` is this port's `ansible.builtin.default`: PLAY and TASK
banners, a colored line per result, and a PLAY RECAP. It is what
[`cli`](cli.md)'s `ansible-playbook` and `ansible-pull` install, and it
serializes its own hooks — results genuinely arrive from one goroutine per
host, so an implementation that keeps state or writes to a shared stream
must do the same. Its banners are not padded out with asterisks the way
real Ansible's `Display.banner` pads them to the terminal width.

Three hooks, where real Ansible has 24, because a hook nothing in this port
can raise and nothing can consume would be an empty promise:

| hook | real equivalent |
| --- | --- |
| `OnPlayStart` | `v2_playbook_on_play_start` |
| `OnTaskResult` | `v2_runner_on_ok`/`_failed`/`_skipped` collapsed into one, since `Result` already carries which it is |
| `OnStats` | `v2_playbook_on_stats` — raised once per `RunPlaybook` call, **including when a play errors out**, so a recap still covers whatever did run |

Two absences worth naming, both disclosed rather than stubbed:
`v2_playbook_on_start` prints nothing at real Ansible's own default
verbosity (it only fires above `-v`), and this port has no verbosity concept
to gate it on; and handler runs are indistinguishable from ordinary task
runs here, so there is nothing to raise real Ansible's separate
`v2_playbook_on_handler_task_start` from.

The PLAY RECAP line itself is byte-for-byte real Ansible's, in both
colour modes, and neither half was obvious. Uncoloured, the host name
sits in a 26-column field. Coloured, **only the host name is coloured**
— and each count column separately, but only when that count is *not
zero*, which is why a real recap shows a plain `unreachable=0` beside a
coloured `failed=1`. The host's own field is 37 columns of the
*already-coloured* string, so its 11 escape characters absorb the
difference and it lands on the same 26 visible columns. The colours
(ok green, changed yellow, unreachable **bright** red, failed red,
skipped cyan, rescued green, ignored bright purple) were captured from
a real coloured run rather than read off the configuration's colour
names. The same distinction runs through the result lines: an
`UNREACHABLE!` fatal is bright red where an ordinary `FAILED!` one is
plain red.

`Engine.OnResult` remains as the one-hook shorthand for a caller that only
wants results and no play or recap events.

## Measured against real Ansible

The engine's control flow has been run side by side with **real
ansible-core 2.21.4**: the same playbook through both, across two hosts,
appending markers to a log so that *ordering* is compared and not just
outcomes — which is what `block`/`rescue`/`always` and handler timing
actually turn on.

Most of it held on the first run: block/rescue/always ordering and
membership, a handler firing exactly once for two notifying tasks, a
handler correctly *not* firing for an unchanged task, `run_once` running
once across both hosts, `changed_when`/`failed_when`/`ignore_errors`, and
every `when:` form. Two things did not, both about loops, and both are
fixed:

- **`register` on a looped task produced no `results` list**, so
  `{{ r.results | map(attribute='stdout') }}` — everyday usage — failed
  outright. A looped task now registers exactly what real Ansible
  registers: `changed`, `failed`, `msg` (`"All items completed"`) and
  `results`, with **none** of the module's own fields at the top level,
  and one entry per iteration carrying the module fields plus `item` and
  `ansible_loop_var`. A task that is not looping still registers its
  fields flat.
- **`loop_control.index_var` was parsed nowhere** and rendered empty. It
  is now bound per iteration, 0-based.

Two differences are known and not yet addressed:

- *(fixed)* The PLAY RECAP now carries `unreachable`/`rescued`/`ignored`
  and counts as real Ansible counts — see below.

A fourth pass measured what the default callback actually *prints*, and
found four things wrong at once:

| | real | here (before) |
| --- | --- | --- |
| `debug: {msg: hello}` | dumps `{"msg": "hello"}` | printed nothing |
| `debug: {var: d}` | the variable's **value** | the variable's **name** |
| play with `hosts: all`, no name | `PLAY [all]` | `PLAY` |
| unnamed task | `TASK [debug]` | no banner |

A debug task printing nothing is the notable one — real Ansible dumps any
result carrying `_ansible_verbose_always`, pretty-printed at four spaces
with its own `_ansible_*` keys stripped, and that marker is the whole
reason a debug task shows anything. `debug: {var: X}` resolves in the
engine alongside `assert`, which is exactly why real Ansible makes both
action plugins rather than modules.

The unnamed-play banner is worth recording as a caution: it was wrong
because of an earlier *reading* of `default.py` here — the `"PLAY"`
fallback is real, but `get_name()` has already substituted the hosts
pattern by the time it is reached. Measuring caught what reading did not.

A failure `ignore_errors` swallowed now prints `...ignoring`, so a red
line that did not stop the run is not mistaken for one that did.

A fifth pass took the failure line itself. It read
`failed: [h] => non-zero return code: 3` — a code and nothing else, not
the command, not its stderr. Real Ansible inlines the whole result, which
is how a reader sees *why* it failed, and gives an unreachable host its
own prefix:

```
fatal: [h1]: FAILED! => {"changed": true, "cmd": "...", "rc": 3, "stderr": "to-stderr", ...}
fatal: [nope]: UNREACHABLE! => {..., "unreachable": true}
```

That JSON comes from [`template`](template.md)'s exported `ToJSON` rather
than a second emitter, so a failure line and the `to_json` filter cannot
drift apart. **Sharing it immediately found three defects that had
nothing to do with failure lines**, each invisible until a real command's
whole result was printed: `to_json` was escaping `<`, `>` and `&` (so
every URL and shell redirect it wrote came out mangled), `shell` reported
`cmd` as a one-element list where real Ansible gives the string, and a
non-zero exit used wording of our own. Printing more of the truth is a
good way to find out what is wrong with it.

A third pass measured the **PLAY RECAP and the exit code**, and found the
counting wrong in three ways at once. For a play with one of each
outcome:

```
real: ok=4 changed=1 unreachable=0 failed=0 skipped=1 rescued=1 ignored=1
here: ok=3 changed=1                failed=2 skipped=1
```

None of the three rules is guessable from the column names: a **changed**
task counts under `ok` as well; an **ignored** failure counts under `ok`
*and* `ignored`, not `failed`; and a **rescued** one counts under
`rescued`, not `failed`. Rescue is not derivable from the results at all
— it is a property of the *block*, and the failing task inside looks
identical either way — so `PlayResult` records it as the block unwinds.

Fixing the columns exposed a second defect: the recap said `failed=0`
while `ansible-playbook` still exited 2. `RunResult.Failed` now answers
the question the exit code actually asks — the host's **final** state,
not the history — so a run whose only failures were ignored or rescued
exits 0, as real `ansible-playbook` does, while a genuine failure still
exits 2.

A second pass covered **roles, the include/import family and the variable
precedence ladder**, and found five more, all now fixed:

- **`set_fact` sat below play vars.** It wrote into the same layer as
  gathered facts, so a play var and a `set_fact` of the same name
  resolved to the play var. `set_fact` and `include_vars` now write to
  the rung the ladder already reserved for them. A consequence worth
  knowing: `meta: clear_facts` no longer clears a `set_fact`, which
  matches real Ansible — it only did before because the two shared a
  layer.
- **Role vars did not outlive their role.** Every role variable read as
  undefined the moment the role ended. Whether they should depends on
  how the role was invoked, measured by running each form on its own: a
  `roles:` entry and `import_role` are static and their variables
  persist for the play, while `include_role` is dynamic and scopes them.
  Only `include_role` is scoped now.
- **`meta/main.yml` was never read**, so role `dependencies:` never ran.
  They now run depth-first before the role's own tasks, each keeping its
  own defaults and vars; a cycle is reported by name.
- **A relative `src:` inside a role was not found.** `copy: {src:
  hello.txt}` in a role failed with "no such file or directory" —  every
  role that ships a file. `copy`/`template`/`script`/`unarchive` now
  search the role's own `files/` (`templates/` for `template`) first.

That second finding nearly shipped wrong. A first probe ran
`include_role` and `import_role` in the same file and appeared to show
`include_role`'s variables persisting; splitting them apart showed it was
the *static* `import_role` injecting them play-wide from the start.

## Check mode

`Engine.CheckMode` — `ansible-playbook --check` — runs a playbook without
changing anything. How real Ansible does it was measured before any of it
was written, and the answer shaped the whole design: a module that
**supports** a dry run is run and reports what it *would* do; one that
does **not** is **skipped**, never run for real.

```
TASK [copy (supports check mode)]     changed: [h1]     ← created nothing
TASK [command (does NOT support it)]  skipping: [h1]    ← never executed
TASK [debug]                          ok: [h1]
```

That asymmetry is what makes the module side safe to fill in one module
at a time: the default for anything unported is **inaction**, not
"modify anyway". A partially-supported check mode is therefore not
dangerous — which is the opposite of what it looks like from the outside,
and worth stating plainly.

The flag reaches a module through its own arguments, under real Ansible's
wire name `_ansible_check_mode`, so adding check mode changed no
signature anywhere in [`modules`](modules.md).

**Supported today**:

- `copy`, `template`, `file`, `lineinfile`, `blockinfile` and `replace` —
  each already decided whether it *would* change before touching
  anything, so a dry run answers the same question and stops short of the
  write. `template` still renders, since a broken template should fail the
  check rather than wait for the real run. `file` mutates from six
  branches, so all six go through one helper that is a no-op under check
  mode — routing them through one place is what makes the absence of a
  stray mutation *auditable* rather than hoped about.
- `command` and `shell` decline with real Ansible's own *"Command would
  have run if not in check mode"* and report **skipped**, since neither
  can know what the command would have done.
- `debug`, `fail`, `stat`, `find` and `slurp` change nothing either way,
  so a check run executes them — real Ansible reports `debug` as `ok` in
  a check run, not `skipping`.

**Everything else is skipped**, and a test asserts that an unported
writing module does *not* claim support — it failed, correctly, on the
change that added the four editing modules, until their tests existed.

## Diff mode

`Engine.DiffMode` — `ansible-playbook --diff` / `-D` — shows what each
task changed, as a unified diff printed before the line saying what
happened. It composes with check mode: `--diff --check` shows the diff of
a change deliberately not made.

```
TASK [edit existing file]
--- before: /etc/app.conf (content)
+++ after: /etc/app.conf (content)
@@ -1,2 +1,3 @@
 alpha
 beta
+gamma

changed: [h1]
```

Like `--check`, the flag reaches a module through its own arguments,
under real Ansible's wire name `_ansible_diff`. A module that does not
report a diff simply prints nothing extra, so turning the flag on can
never fail a run. `copy`, `template`, `lineinfile`, `blockinfile` and
`replace` report one today.

### Why the renderer is a port, not a reimplementation

Go has no unified diff in its standard library, and writing "a" unified
diff is not enough: the output has to be the one real Ansible prints.
Real Ansible calls Python's `difflib`, so `difflib` is what was ported —
`SequenceMatcher` with its autojunk rule, `get_opcodes`,
`get_grouped_opcodes`, `_format_range_unified` — together with
`CallbackBase._get_diff`'s header lines, skip messages and trailing blank
line.

A plain longest-common-subsequence walk was written first and is *wrong*:
`SequenceMatcher` places an inserted duplicate of an existing line
**before** that line where an LCS walk places it after. Both are valid
diffs of the same length; only one matches. Three more details that a
reimplementation gets wrong, each caught by measurement rather than by
reading:

- A one-line hunk range prints no length — `@@ -1 +0,0 @@`, not
  `@@ -1,1 +0,0 @@`.
- Lines keep their terminators, so a file whose last line lacks a newline
  differs from one where it does; real Ansible appends
  `\ No newline at end of file` to that line before diffing.
- A replace emits every removal and **then** every addition, never
  interleaved.

Verified against 373 cases whose expected output was produced by calling
real ansible-core 2.21.4's own `_get_diff` — two shapes measured from a
live run, eleven hand-picked edge cases, 300 randomised pairs, and 60
large ones crossing `difflib`'s 200-element autojunk threshold. The
randomised half earned its place by finding the `SequenceMatcher`
divergence on its twentieth case; each of the three details above was
confirmed load-bearing by breaking it, failing 19, 217 and all cases.

Running the same playbook through this port and through real
ansible-core produces identical transcripts, both with and without
`--check`, apart from the banner asterisk padding this port does not
emit.

### One disclosed divergence

Real Ansible has no single convention for the `---`/`+++` headers — each
module names its sides its own way, and each is reproduced: `(content)`
suffixes for `lineinfile` and `blockinfile`, bare paths for `replace` and
for `copy` with `content:`, and the **source** path for `copy` with
`src:`.

`template` is the exception. Real Ansible renders to a temporary file and
names *that* as the after side — `~/.ansible/tmp/ansible-local-.../x.j2`,
a path that differs between two runs on the same machine. This port
renders in memory and has no such file, so it names the destination
instead.

## Tags

Tag selection is a port of real Ansible's `evaluate_tags`
(`ansible/playbook/taggable.py`), which is more than an intersection
test:

- A task with no tags carries the implicit tag `untagged` — that is what
  lets `--tags untagged` and `--skip-tags untagged` select it at all.
- `all`, `tagged`, `untagged`, `always` and `never` are special names on
  **both** sides.
- The run side is evaluated **before** the skip side, not the reverse.
- An empty run list means `all`, which is what excludes `never`.

That last substitution is what real Ansible does in its **CLI** rather
than its config (`ansible/cli/__init__.py`, whose own comment explains
why: making `["all"]` the config default would turn `--tags foo` into
`["all", "foo"]`). It is applied in the engine here instead, so every
entry point gets it — **a task tagged `never` running by default is a
safety failure**, and guarding a destructive task is the only thing
`never` is for.

A task excluded by tags produces **no output at all** — no banner, no
`skipping:` line, and nothing in the recap. That is the difference
between *selection* and *skipping*: tags decide which tasks are in the
play, whereas `when:` skips a task that is. This port reported the first
as the second, which also made the recap's own `skipped` count wrong.

Tag *inheritance* — play tags and block tags reaching the tasks inside —
was already correct, and is covered by its own measured table.

## Host selection

`hosts:` and `--limit` share one pattern language, and its terms are
applied by **kind** rather than left to right, which is what real
Ansible's `order_patterns` does: every plain term first, then every `&`
intersection, then every `!` exclusion. A pattern with no plain term at
all gets an implicit `all` to subtract from.

That ordering is not a detail. Applying terms in written order makes
`hosts: "!db-primary"` select **exactly the host it names** — a play
written to avoid one machine would run on that machine and nowhere
else. It also let a plain term written after an exclusion resurrect an
excluded host, so `web:!h1:h1` selected all five hosts where real
selects four.

`--limit` (`-l`) takes the same language and **intersects** with a
play's own `hosts:` rather than replacing it: a play already narrower
than the limit keeps its own narrower set, and the limit can only ever
remove hosts. A limit that leaves nothing to target is a hard error,
checked once before any play runs and against `all` — which is why a
play whose own `hosts:` matches nothing is not an error while a
`--limit` matching nothing is.

### serial

`serial:` accepts a count, a percentage, or a list of either, matching
real Ansible's own list-typed attribute — `serial: 2` and `serial: [2]`
are the same play. The list is consumed in order and its **last entry
repeats** until every host has run, so `[1, 2]` over five hosts gives
batches of 1, 2 and 2. A percentage is of the play's **total** host
count, and one that works out to zero becomes one.

The percentage is computed in floating point and truncated, exactly as
Python's `int((pct / 100.0) * total)` does, because the two disagree:
`0.29 * 100` is `28.999999999999996`, so 29% of 100 hosts batches as
28/28/28/16, not 29/29/29/13.

Only the plain-integer form worked before; a percentage or a list parsed
as zero and the play ran **every host at once, in silence** — the exact
opposite of what `serial` is for.

## Rolling updates and reporting

`serial:` batching was already correct; what was missing was that real
Ansible **re-banners the play and the task for each batch**, which is
what makes the boundaries of a rolling update visible while it runs.

Other reporting details, each measured rather than assumed:

- `run_once:` always runs on the play's **first** host. This port used
  whichever host won a race, so a `run_once` task that registered a
  variable recorded a different `inventory_hostname` from run to run.
- A delegated task reports where it ran — `ok: [h1 -> h5]` — on its ok,
  changed and fatal lines, but **not** on `skipping:`, because a skipped
  task never connected anywhere.
- A task being retried prints
  `FAILED - RETRYING: [h1]: name (2 retries left).` after each failed
  attempt, in `COLOR_DEBUG`'s dark gray, *after* its task banner.
  Without it a loop with a `delay:` is indistinguishable from a hang.
- A play whose pattern matched nothing says `skipping: no hosts matched`.
- The recap ends with a blank line, and an **ignored failure that
  changed something counts under `changed`**.

## Inspecting a playbook without running it

`--list-tasks`, `--list-tags`, `--list-hosts` and `--syntax-check` print
what a playbook would do and run nothing. The listings apply the run's
own tag filter, through the engine's exported `TagsSelect`, and name
tasks through its exported `DisplayName` — a listing that disagreed with
the run it describes would be worse than none, so neither rule is
written twice.

A block's contents are listed; its `rescue:` and `always:` are not, even
though they will run. Role tasks are listed and bannered alike as
`r1 : role-task`. Host *order* in `--list-hosts` is not matched: real
ansible-core's own order is non-deterministic — three runs of one
playbook gave three different orders — so only the set is.

## Resuming and forcing

`--start-at-task` skips until a task whose name matches, then runs from
there. The match is exact or a shell glob, case-sensitive, and the
started state carries across plays *and* across playbook files, so
`a.yml b.yml --start-at-task x` runs all of `b.yml` when `x` is in
`a.yml`.

`--force-handlers` runs notified handlers on hosts that already failed —
without it a failed host runs nothing further, which can leave a service
stopped because the handler that would have restarted it never ran.

`--flush-cache` is accepted and does nothing: this port keeps no fact
cache. It is accepted so a command line written for real
ansible-playbook still runs, and says so rather than implying a cache
exists.

## no_log, and the keywords this port refuses

`no_log: true` is honoured: the outcome of a task is still reported,
its contents are not, and a **failing** task — the case where a result
is dumped in full, and so the worst moment to leak — reports only real
Ansible's own censored result. A looped task's item is censored too,
since the item is frequently the secret itself.

Real Ansible exposes 42 task keywords; this port honours most, and
**refuses the rest by name**. A key the parser does not recognise is
taken for the module, so `no_log: true` beside `debug:` used to fail
with *"ambiguous module"* — a message that reads like the playbook is
malformed when it is this port that is incomplete. Now it says which
keyword, and how to get the same effect where there is a way:
task-level `connection:` → set it on the play, or `ansible_connection`
on the host; `throttle:` → use `serial:`.

The list shrinks as the port catches up: `environment:`,
`any_errors_fatal:`, `check_mode:`, `no_log:`, `module_defaults:` and
`ignore_unreachable:` were all on it and are now honoured.

They are refused rather than ignored on purpose. Silently accepting
`connection: local` would run the task somewhere other than the
playbook says.

## environment

`environment:` sets variables in the environment of the command a task
runs, at play and task level. A play's entries reach every task; a
task's own are **merged over** them, key by key, so a task adds to its
play rather than replacing it and wins only where both set the same
name. Values are templated, so `TMPL: "hello-{{ who }}"` works.

It was accepted at play level and then **ignored** until this was
measured — the command ran without the variable and nothing said so —
and refused at task level for one release, on the grounds that running
something other than the playbook says is worse than declining.

The variables are applied as a shell `export` preceding the command,
rather than as `FOO=bar cmd` assignments: a command may itself be
`cd somewhere && real-command`, where assignments written in front
would apply to the `cd` alone.

A YAML `true` becomes the string `True`, capitalised, because that is
what Python's `str()` produces and what a script testing
`[ "$FLAG" = "True" ]` expects.

## An undefined variable is an error

Real Ansible fails on an undefined variable — in a module argument, in
a string with one interpolated into it, in a `when:`. This port
rendered it as null, so a **misspelled variable name silently did the
wrong thing**:

```yaml
debug:   {msg: "{{ pakcage_name }}"}   # printed null, reported ok
command: "echo {{ pakcage_name }}"     # ran `echo `, reported changed
```

Both now stop the play, as they do there. `default()` still guards,
which is what a real playbook uses:

```yaml
msg: "{{ maybe_missing | default('fallback') }}"
```

The innermost message matches real's wording — `'x' is undefined` —
but not its whole chain: real names the module and the argument
(*"Finalization of task args for 'ansible.builtin.debug' failed:
Error while resolving value for 'msg'"*), which this port does not
reproduce.

## with_* loops

`with_<name>` is the `<name>` **lookup plugin**, which is exactly how
real implements it: `with_dict` *is* the `dict` lookup, run with
`wantlist` forced on.

Supported through their plugins: `with_items`, `with_list`,
`with_flattened`, `with_dict`, `with_nested`, `with_together`,
`with_indexed_items`, `with_sequence`, `with_subelements`, plus
`with_env`, `with_file` and `with_pipe`. A `with_` key is recognised
by asking the plugin set whether such a lookup exists, so a module
whose name merely starts with `with_` is still a module — and a new
lookup gets its `with_` form for nothing.

The value supplies the lookup's **terms**: a list spreads into several
(`with_nested: [[1,2],[a,b]]` is two terms), anything else is one
(`with_dict: "{{ d }}"`).

Three of these read the opposite of how they sound, so they were
measured rather than assumed: `list` does **not** flatten while
`items` flattens one level; `together` pads short lists with `null`
(`zip_longest`, not `zip`); and `sequence` yields **strings**.

### Loop labels

Each iteration prints `(item=...)` using Python's `str()` — a dict
shows as `{'k': 'v'}`, `None` rather than `<nil>`. `loop_control.label`
replaces what is shown, which is the point of it:

```yaml
loop_control:
  label: "user {{ item.name }}"    # not the whole record
```

The label changes only the printing: `register:` and the result dict
still carry the real item.

### One disclosed divergence

`dict2items`, the `dict` lookup and a printed dict all iterate in
**key order**, where real preserves the mapping's document order. This
port decodes YAML mappings into Go maps, which have no order at all —
sorting is deterministic and matches real whenever the document was
already in key order. (Before this, a Go map's randomised iteration
made `loop: "{{ d | dict2items }}"` print in a different order on
about one run in eight.)

## What a rescue can see

`ansible_failed_task` and `ansible_failed_result` are set the moment a
block starts **rescuing**, which is how a rescue says what it caught:

```yaml
rescue:
  - debug:
      msg: "{{ ansible_failed_task.name }} failed: {{ ansible_failed_result.msg }}"
```

Neither existed until this was measured, so the idiom the Ansible
docs give for a rescue block failed on an undefined variable.

They are *not* set by a failure `ignore_errors` swallowed, nor by one
with no rescue to catch it. Real stores them as nonpersistent facts,
so they stay readable in `always:` and after the block ends — not
scoped to the rescue.

`ansible_failed_task` is real's whole task attribute dump, all 43
keys. The ones this port models carry its own value; the keywords it
refuses can only ever be at real's default, so reporting that default
is accurate rather than invented. An **unset** attribute reports
`null`, as it does there, not a Go zero — and `register` is not the
name but real's own map-to-sentinel shape.

## Handlers: names and listen topics

A `notify:` resolves through two separate matches:

- the **first** handler with that name, and only that one — a second
  handler sharing the name never runs;
- then **every** handler whose `listen:` carries the name, in
  definition order, deduplicated by handler name.

Both can fire at once: a handler named `restart` plus two others
listening to `restart` all run. Handlers run in the order they are
**defined**, not the order they were notified, and a handler notified
twice still runs once.

`listen:` used to be a parse error, so a role using the ordinary
"notify a topic, several handlers answer" pattern would not load.

The duplicate-name rule was measured rather than read: real's own
source says *"last handler loaded with the same name wins"*, and
running it shows the **first** one winning — the reversal that comment
describes is over handler *blocks*, and a play's handlers are one
block.

## action, local_action and args

Three ways to name a module, or its arguments, somewhere other than
the module's own key. All three are honoured:

```yaml
- action: command id -un                 # first word is the module
- action: {module: copy, content: hi, dest: /tmp/x}
- local_action: command id -un           # = action + delegate_to: localhost
- command: echo hello
  args: {chdir: /tmp}
```

A string value's **first word** names the module and the rest is
`_raw_params` — which differs from the module-key form, where the
whole string is `_raw_params`. `args:` sits **under** the module key's
own arguments: it supplies what the module key does not set and loses
to it where both do.

Real emits a `[DEPRECATION WARNING]` for the **mapping** form of
`action`/`local_action` (removed in 2.23). This port has no
deprecation-warning mechanism at all, named here rather than faked.

## become

Privilege escalation resolves the same way `connection:` does, and the
middle term is the one worth stating plainly:

> **host var > task keyword > play keyword**

So `ansible_become: false` on a host beats `become: true` on a task —
the host wins, and nothing escalates. Both directions of that were
wrong here until it was measured: a host that said *not* to escalate
was escalated on anyway, and a host that asked for escalation was
ignored, so the task ran unprivileged and silently did something
other than what was asked. `ansible_become_user` and
`ansible_become_method` rank the same way.

`become_user:` is templated, so `become_user: "{{ deploy_user }}"`
works; it used to reach `sudo` as the literal braces.

One difference is disclosed rather than hidden. Real Ansible becomes
an unprivileged user by copying its temp files to the target and
`chmod`-ing them, so a failed escalation there reports *"Failed to set
permissions on the temporary files…"*; this port invokes the become
program directly and reports what it said. The **user resolved** is
identical — that is what the measurement checks — but the wording of a
failure is not.

## yes and no are booleans

Real Ansible parses with PyYAML, a **YAML 1.1** implementation.
`gopkg.in/yaml.v3`, which this port uses, implements YAML 1.2. The two
disagree about a set of words that appear constantly in real
playbooks:

| written | PyYAML (real) | YAML 1.2 |
| --- | --- | --- |
| `yes` `Yes` `YES` `on` `On` `ON` | `True` | the string |
| `no` `No` `NO` `off` `Off` `OFF` | `False` | the string |
| `true` `True` `TRUE` / `false` … | boolean | boolean |
| `y` `n` | the string | the string |

So `vars: {flag: yes}` with `when: flag` *failed* here — *"Conditional
result was derived from value of type str"* — on a playbook real
Ansible runs without complaint, and `force: yes` reached a module as a
string. Those words are now read as booleans.

The set is PyYAML's own implicit resolver, read out of
`yaml.resolver.Resolver` at run time rather than copied from the
spec — which is how bare `y`/`n` got their row above: the YAML 1.1
spec lists them, PyYAML does not resolve them.

Only **plain** scalars are converted. `"yes"` and `'yes'` stay strings
in both YAML versions, and that is how a playbook asks for the word.
A `!vault` value is resolved *before* it is decrypted, so a secret
whose plaintext is `no` stays the string `no`.

One YAML 1.1 rule is **not** implemented, named here rather than left
to be discovered: sexagesimal integers. `1:30` is `90` in real
Ansible and the string `"1:30"` here.

## Variables on a block and on an include

`vars:` on a `block:` reaches every task inside it and goes out of
scope with it, sitting between the play's variables and the task's
own. The same applies to an `include_tasks`/`import_tasks`, which is
how an included file is parameterised:

```yaml
- name: set it up
  include_tasks: setup.yml
  vars:
    port: 8080
```

Both were accepted and **dropped** until this was measured — the
variable arrived undefined, silently.

A **dynamic** include also announces itself, as it does there: a TASK
banner and an `included: <absolute path> for <host>` line, counted as
ok in the recap. A static import announces nothing. `include_tasks:`
takes either a bare path or the mapping form `{file: path}`.

## Unreachable hosts, and connecting per task

Real Ansible establishes a connection when a task needs one, not once
before the play. Five things follow from that, and this port now does
the same:

`ignore_unreachable:` keeps a host in the play when it cannot be
reached — the result still prints `UNREACHABLE!`, the recap counts it
ok+ignored rather than unreachable, and the run can still exit 0. It is
decided **per task**, at play or task level, which is what makes an
ignored-unreachable `ping` followed by an ordinary `command` report
UNREACHABLE *twice* and drop the host only on the second:

```
TASK [dead]        fatal: [hx]: UNREACHABLE! => {...}
                   ...ignoring                       ← ignore_unreachable
TASK [some debug]  ok: [hx]                          ← needs no connection
TASK [needs one]   fatal: [hx]: UNREACHABLE! => {...}← host dropped here
```

The tasks that run anyway are the ones real Ansible never opens a
connection for: `add_host`, `assert`, `debug`, `fail`, `group_by`,
`include_vars`, `meta`, `pause`, `set_fact`, `set_stats`,
`validate_argument_spec`. That list is the reference's own — its action
plugins that call neither `_execute_module` nor
`_low_level_execute_command` — and each was then run against an
unreachable host to confirm it.

A connect failure surfaces under the first task that needs a
connection, rather than under a step of its own. That is usually
**`Gathering Facts`**, real's banner for the implicit gather step,
which this port called `(gather_facts)` — a divergence on the second
line of every transcript of a play that gathers facts, i.e. the
default.

`ansible-playbook` exits **4** when a host was unreachable and **2**
when one merely failed, and unreachable **wins** rather than
combining: one failed host plus one unreachable host exits 4, not 6.
This port returned 2 for both, so a caller telling them apart — a retry
loop, a deployment gate — could not.

## meta

`meta:` is not a module. Real Ansible's strategy executes it directly:
it prints a TASK header with **nothing underneath**, and counts toward
nothing in the recap. `meta: flush_handlers` banners *before* its
handlers run. `noop`, `flush_handlers` and `clear_facts` are supported;
any other action is refused by name.

## module_defaults

`module_defaults:` gives a module its arguments once, for every task in
scope, at play, block or task level:

```yaml
- hosts: web
  module_defaults:
    file: {mode: "0640"}
  tasks:
    - file: {path: /etc/app.conf, state: touch}   # created 0640
```

The defaults sit **under** a task's own arguments, which keep winning.
Values are templated like any argument, and a fully-qualified key
matches a task written with the bare name — a play-level
`ansible.builtin.copy` default applies to a task that says `copy:`.

One semantic is easy to assume backwards, so it was measured rather
than reasoned about: a nearer level **replaces** an outer level's whole
entry for a module rather than merging into it key by key. A play-level
`copy: {mode, content}` under a task-level `copy: {mode}` loses the
content, and the task fails *"src (or content) is required"* — in real
Ansible too. This port does the same thing.

This keyword parsed and did nothing until it was measured against real
ansible-core: a play asking for `mode: "0640"` got `0600`, the module's
own default. A playbook that sets a permission was quietly getting
another one.

A `group/...` key names an action group, which this port has no concept
of, and is refused by name rather than dropped.

## check_mode on a play or a task

`check_mode: true` puts a play or a task into a dry run without
`--check`, and `check_mode: false` takes a task **out** of one — which
is the more useful direction: it is how a playbook reads the state it
needs in order to predict the rest.

Precedence was measured rather than guessed: task over play over the
`--check` flag.

A play saying `check_mode: true` used to write the file anyway. That is
the worst shape this class of defect takes — the keyword whose entire
purpose is to not touch anything was the one being ignored.

## order

`order:` sets the order hosts run in within a play: `inventory` (the
default), `reverse_inventory`, `sorted`, `reverse_sorted` and `shuffle`.
It was accepted and ignored.

The order is only observable when one host runs at a time, so at
`--forks 1` this port now runs its hosts inline rather than through a
goroutine each — with five forks, five hosts interleave in whatever
order they finish, there as here.

## Loops

A `when:` on a looping task is evaluated **per item**, with `item`
bound. It used to be evaluated once before the loop, where `item` does
not exist, which was wrong in both directions:

| `when:` | real ansible-core | this port before |
|---|---|---|
| `item != 2` | runs 1 and 3 | ran all three |
| `item == 1` | runs 1 | skipped the whole task |

A playbook that said to skip an item acted on it anyway — for a loop
over packages, users or hosts, that is doing the thing it was told not
to.

A looped task counts **once per host** in the recap however many items
it ran: three changed items report `changed=1`, not 3. Iterations fold
before counting — changed if any changed, skipped only if every one
was. Each line carries its item, `=> (item=x)`, and a skipped one keeps
the trailing space real Ansible leaves there.

## Stopping a rolling update

`any_errors_fatal:` stops the whole play the moment any host fails, and
`max_fail_percentage:` stops it once more than that share of the batch
has failed. Both are checked after every task, as real Ansible checks
them, and both stop the **play** rather than the current batch — a
rolling update that gives up must not roll on.

`max_fail_percentage` compares `failed / batch_size` **strictly
greater** than the percentage, so one host of five failing (20%)
continues at `20` and stops at `19`. A task's own `any_errors_fatal:`
overrides the play's, and `run_once:` implies it: real Ansible treats a
failing `run_once` task as fatal for everyone, since the one host stood
in for all of them.

Both **parsed and were ignored** until this was measured, which is the
worst shape for a safety brake: a play saying *if any host fails, stop*
carried on deploying to the rest of the fleet, silently.

## Connection settings on a play

`connection:`, `remote_user:` and `port:` apply to every host in the
play. A **host variable wins over them** — a host declaring
`ansible_connection=ssh` keeps ssh even under a play that says `local`
— which is the precedence real Ansible applies.

`connection: local` was ignored too, so a play meant to run on the
control node was connected to over SSH and every task came back
`UNREACHABLE`.

The same three keywords on a *task* are still refused by name: a
per-task connection needs its own connection, the way `delegate_to`
already builds one.

### Known gaps

- Real ansible-core 2.21 prints a structured `[ERROR]` block naming the
  playbook file, line, column and a source excerpt. This port has no
  source-position tracking, so it prints only the `fatal:` line.
- Real also emits `[WARNING]: Could not match supplied host pattern` on
  **stderr**; `DefaultCallback` holds a single output stream.
- For a command whose executable does not exist, real reports
  `changed=false` (it execs directly and gets `ENOENT`) where this port
  reports `changed=true` with `rc=127`, because the command reaches the
  target through a shell. `command` does *not* interpret shell
  metacharacters here — that part matches. Distinguishing the two needs
  an exec-versus-shell distinction at the transport, and keying off
  `rc == 127` would misfire on a program that legitimately exits 127.

## Conditionals

A conditional must evaluate to a **boolean**. A `when:` yielding a
string, number, list, dict or null is an error, as it is in real
ansible-core 2.21.

The rule is worth keeping for the reason it was introduced: a
non-boolean conditional is usually a template used where one is not
supported, and it then reads as **always true** — so the task runs every
time, silently. That is an action difference, not a reporting one.

`Engine.AllowBrokenConditionals`, and the `ANSIBLE_ALLOW_BROKEN_CONDITIONALS`
environment variable, relax it to ordinary truthiness. Real Ansible has
the same switch, defaults it off the same way, and plans to remove it in
2.23 — a playbook that needs it is one real ansible-core also refuses,
so it is there to unblock a migration rather than to be left on.

A conditional wrapped in templating delimiters — `when: "{{ flag }}"` —
is accepted, with the same deprecated status it has upstream. This port
used to FAIL such a task: every conditional is evaluated by wrapping it
in `{{ }}`, so one already wrapped became `{{ {{ flag }} }}`. Only a
conditional that is entirely one expression is unwrapped; `{{ a }} and
{{ b }}` is left as written.

## Example

```go
package main

import (
	"context"
	"fmt"
	"log"

	"github.com/go-ansible/inventory"
	"github.com/go-ansible/playbook"
)

func main() {
	inv, err := inventory.ParseYAML([]byte(`
all:
  hosts:
    localhost:
      ansible_connection: local
`))
	if err != nil {
		log.Fatal(err)
	}

	pb, err := playbook.ParseFile("site.yml")
	if err != nil {
		log.Fatal(err)
	}

	e := playbook.New(inv)
	rr, err := e.RunPlaybook(context.Background(), pb)
	if err != nil {
		log.Fatal(err)
	}
	if rr.Failed() {
		log.Fatalf("run failed: %+v", rr.Summary())
	}
	fmt.Println(rr.Summary())
}
```

## What is not implemented

`include_tasks`/`import_tasks`/`include_role`/`import_role` resolve
**statically at parse time**, not real Ansible's dynamic, per-host,
possibly-templated resolution — documented as narrower, not silently
different. The namespace/collection metadata system, Python-style inventory
*plugins*, and lookup/callback plugins are out of scope entirely (executable
inventory *scripts* — the `--list`/`--host` protocol — are supported; see
[inventory](inventory.md)). Any playbook `strategy` other than `linear`/
`free` (`debug`, `host_pinned`, a strategy plugin) is rejected with an
explicit parse error rather than silently treated as `linear`.
`async`/`poll` only genuinely backgrounds `command`/`shell` on the target
— the only two modules whose entire work reduces to one remote invocation —
and an overrunning job is detected on timeout but not actively killed, since
real Ansible's `killpg` has no portable POSIX equivalent without `setsid`
(absent on macOS). `run_once` broadcasts its `register:` value across a
batch but is not coordinated across `strategy: free`'s independent per-host
lanes.
