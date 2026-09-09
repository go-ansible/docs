# template

[![CI](https://github.com/go-ansible/template/actions/workflows/ci.yml/badge.svg)](https://github.com/go-ansible/template/actions/workflows/ci.yml)

`github.com/go-ansible/template` renders Ansible's templating semantics on
top of a Jinja2-compatible engine
([`nikolalohinski/gonja/v2`](https://github.com/NikolaLohinski/gonja)):
string interpolation with `{{ }}`, control structures with `{% %}`, Ansible's
own filter and test library layered on Jinja2's built-ins, and Ansible's rule
that a value which is a **single** `{{ expr }}` with nothing else around it
renders to the expression's *native type* (list, dict, int, bool…), not a
string.

## API

```go
func New() *Engine

func (e *Engine) Render(src string, data map[string]any) (string, error)
func (e *Engine) Eval(exprSrc string, data map[string]any) (any, error)
func (e *Engine) EvalBool(exprSrc string, data map[string]any) (bool, error)
func (e *Engine) RenderValue(raw any, data map[string]any) (any, error)

// Set to receive warnings a render has nowhere else to send — currently
// only a lookup called with errors=warn. Left nil, such a lookup
// degrades to errors=ignore.
Engine.OnWarning func(msg string)

func IsTemplate(s string) bool
```

- `Render` renders a full template string (text mixed with `{{ }}`/`{% %}`)
  to its string form — the equivalent of Ansible templating a whole file.
- `Eval` evaluates a bare Jinja2 expression (no `{{ }}` wrapper — the raw text
  of an Ansible `when:` condition, say) and returns its native Go value.
- `EvalBool` evaluates an expression and applies Ansible/Jinja2 truthiness —
  exactly the semantics a `when:` condition needs.
- `RenderValue` is the one most callers want: given an arbitrary decoded YAML
  value (a task's `args:` map, say), it walks maps and lists recursively,
  templates every string it finds, and — critically — applies the
  whole-expression rule per string, so `image: "{{ tag }}"` renders to a
  string but `count: "{{ replicas }}"` can come back as a Go `int` if
  `replicas` is one, rather than the string `"3"`. Non-string scalars pass
  through unchanged. `IsTemplate` is the cheap pre-check `RenderValue` uses
  internally to skip strings with no `{{`, `{%`, or `{#` at all.
- `omit` is a real global, matching Ansible's own sentinel
  (`ansible._internal._templating._utils.Omit`): `{{ x | default(omit) }}`
  evaluates to `template.Omit` when `x` is undefined, and `RenderValue`
  drops the containing map key or list item entirely rather than passing
  the sentinel through — so `playbook`'s existing
  `Template.RenderValue(task.Args, ...)` call already makes
  `some_arg: "{{ x | default(omit) }}"` omit `some_arg` from the module
  call outright when `x` is unset, with no changes needed outside this
  package. A bare top-level `omit` result (nothing to drop it from) is an
  error, matching real Ansible's own `AnsibleValueOmittedError`.

## Filter and test library

Filters and tests are registered on top of gonja's Jinja2 built-ins, not
instead of them. As of this writing the Ansible-specific set includes:

**Filters** — `to_json`/`to_nice_json`/`from_json`, `to_yaml`/`to_nice_yaml`/`from_yaml`,
`regex_replace`/`regex_search`/`regex_findall`/`regex_escape` (backed by
[`go-regexp/engine`](https://github.com/go-regexp/engine), a PCRE-compatible
engine, since Ansible's regex filters rely on Python `re` semantics that
Go's `regexp` doesn't fully match), `bool`, `mandatory`, `ternary`, `combine`,
`dict2items`/`items2dict`, `type_debug`, `quote`,
`b64encode`/`b64decode`, `md5`/`sha1`/`checksum` (an alias for `sha1`, matching
real Ansible), `hash` (generic — defaults to sha1, takes an algorithm name:
md5/sha1/sha224/sha256/sha384/sha512), `union`/`intersect`/`difference`/
`symmetric_difference` (set theory over lists — `unique` itself is left to
gonja's own built-in, which already matches what real Ansible's `unique`
delegates to), `log`/`pow`/`root`, `human_readable`/`human_to_bytes` (byte-exact
port of `ansible.module_utils.common.text.formatters`' size tables),
`rekey_on_member`, `to_uuid` (RFC 4122 UUID v5, Ansible's own default
namespace), `product`/`permutations`/`combinations`/`zip`/`zip_longest`
(Go has no `itertools` equivalent — each is a hand-traced, order-exact port
of the real Python algorithm, not just its result set), `comment`
(plain/erlang/c/cblock/xml styles, plus a full set of
overridable decoration/prefix/postfix parameters), and the path filters
`basename`/`dirname`/`path_join`/`splitext`/`expanduser`/`expandvars`/
`realpath`/`relpath`/`normpath`/`commonpath`/`win_basename`/`win_dirname`/
`win_splitdrive` — ported from real Python's own `posixpath`/`ntpath`/
`genericpath` source (not just `os.path`'s documented behavior), since the
two disagree on a few edge cases Go's `path`/`path/filepath` packages don't
reproduce (e.g. `os.path.basename('/foo/bar/')` is `''`, not `'bar'`),
`extract` (chained container lookups by key/index), `flatten`
(nested-list collapsing, with a `levels` depth limit), `subelements`
(pairs each element of a list/dict with every item its dotted accessor
finds), `split` (Python's `str.split()` semantics exactly, including
the different behavior of the default whitespace-run split vs. an
explicit separator), `fileglob` (real filesystem globbing, filtered
to regular files only — `filepath.Glob` plus an `os.Stat` check, matching
real Ansible's own `[g for g in glob.glob(pathname) if os.path.isfile(g)]`),
and `to_datetime`/`strftime` (Python `%`-directive date parsing/formatting,
translated to Go's reference-time layout tokens). `to_datetime` returns a
Unix-timestamp `float64` rather than a datetime object — a deliberate,
disclosed divergence: gonja's own binary-operator evaluator only
implements arithmetic for numeric values, so a `time.Time`-shaped result
would make `(a | to_datetime) - (b | to_datetime)` — real Ansible's own
dominant use of this filter, elapsed-time math between two parsed dates —
silently wrong instead of working. The float representation makes that
subtraction (and `<`/`>` comparisons) work correctly through gonja's
existing numeric operators, at the cost of `{{ x | to_datetime }}`
printing a raw number instead of Python's own formatted datetime repr.
`password_hash` (crypt(3)/MCF password hashing — md5/sha256/sha512-crypt
and bcrypt/blowfish, matching real Ansible's own hashtype names, defaults,
and implicit rounds/cost — real Ansible's own sha256/sha512 defaults are
535000/656000 rounds, deliberately far above crypt(3)'s own spec default
of 5000) via [`go-encryptions/unixcrypt`](https://github.com/go-encryptions/unixcrypt),
a small shared library extracted from `go-puppet/puppet`'s own `pw_hash()`
implementation (Puppet's stdlib equivalent) so the crypt(3) algorithms are
implemented once, not duplicated per consumer.

**Tests** — `changed`, `success`/`succeeded`, `failed`/`failure`, `skipped`
(each reads a registered task result's flags, e.g. `is changed`), and
`version` (Ansible's `version_compare`-equivalent `is version(...)` test).

**Lookups** — `lookup(name, ...)` plus `query(...)`/`q(...)` (the same
thing with `wantlist` forced on), registered as Jinja *global functions*
exactly as real Ansible registers them, not as filters. The two universal
keyword arguments every lookup call accepts are handled by the shared
dispatcher rather than per plugin: `wantlist` (default false) and
`errors` (`strict`/`warn`/`ignore`, default `strict`). Result shaping
matches real Ansible's own: one result unwraps to a scalar, several
all-string results join with a bare comma, anything else stays a list.
Plugins shipped so far are `env`, `pipe` (runs on the controller, like
real Ansible) and `file`. A lookup receives the live variable context of
its call site, which this port supplies by building the lookup globals
per render rather than once per engine — gonja's own context type cannot
enumerate what is in scope at call time, so the caller's own variable map
is captured directly instead. Two real, disclosed narrowings versus
upstream: `pipe` uses the process's working directory rather than a
per-play basedir, and `file` resolves a relative path against that same
working directory rather than through a role's `files/` search path,
which go-ansible does not model yet. `errors=warn` reports through the
caller-supplied `Engine.OnWarning` hook, since this package has no
display layer of its own; with no hook installed it degrades to
`errors=ignore`.

## Measured against real Ansible

The filter and lookup surface has been run side by side with **real
ansible-core 2.21.4**: one playbook through both engines, each writing
`key=value` lines to a file, then diffed. That matters because until then
every filter had been checked against a reference value *derived from
reading ansible-core's source* — and a reference derived from a reading
can be wrong in exactly the way the reading was.

Most of it held. Every `itertools` port (`product`, `permutations`,
`combinations`, `zip`, `zip_longest`), `subelements`, `rekey_on_member`,
`flatten`, `split`, the set-theory filters and every lookup matched
exactly, ordering included. Five filters did not, and were fixed in
v0.11.0: `type_debug` was reporting Go type names rather than Python's
(so `when: x | type_debug == 'dict'` was silently false), `ternary` was
missing its third `none_val` argument, `to_json` used Go's compact
separators and ignored `indent`, `to_yaml` and `to_nice_yaml` were the
same function, and `root(3)` was a ULP out because Go's `math.Pow`
differs from the C `pow()` Python calls.

Three differences remain, all below the filter layer and disclosed rather
than papered over:

- **Dict key order.** Real Ansible emits a dict in insertion order,
  because Python dicts keep it. A Go map does not, and the order is gone
  before a value ever reaches a filter — it was lost when the YAML was
  decoded. Keys are sorted instead, so output is at least deterministic.
- **Backslashes in string literals.** Real Ansible treats them literally
  (`'a\b'` renders `a\b`, `'a\\b'` renders `a\\b`); gonja applies escape
  processing and rejects an unknown escape outright, so `'C:\Users\foo'`
  is a parse error here and ordinary in real Ansible. This is in gonja's
  lexer, not in this package.
- **`to_nice_yaml` sequence indentation.** yaml.v3 indents a block
  sequence under its key where PyYAML puts the dashes at the key's own
  indentation. The two parse identically.

## Example

```go
package main

import (
	"fmt"
	"log"

	"github.com/go-ansible/template"
)

func main() {
	e := template.New()

	// A whole-expression template preserves the native type.
	v, err := e.RenderValue("{{ replicas }}", map[string]any{"replicas": 3})
	if err != nil {
		log.Fatal(err)
	}
	fmt.Printf("%T %v\n", v, v) // int 3

	s, err := e.Render("Deploying {{ app }} to {{ env | upper }}", map[string]any{
		"app": "api", "env": "prod",
	})
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(s) // Deploying api to PROD
}
```

## Where this fits

`template.Engine.Render`/`RenderValue` take a `map[string]any` — that's
[`vars.Context.Merged()`](vars.md)'s return type, so the variable context
built from an [`inventory`](inventory.md) plus the precedence ladder in
`vars` is exactly what a caller hands to this engine to render a task's
arguments.
