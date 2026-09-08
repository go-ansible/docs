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
explicit separator), and `fileglob` (real filesystem globbing, filtered
to regular files only — `filepath.Glob` plus an `os.Stat` check, matching
real Ansible's own `[g for g in glob.glob(pathname) if os.path.isfile(g)]`).

**Tests** — `changed`, `success`/`succeeded`, `failed`/`failure`, `skipped`
(each reads a registered task result's flags, e.g. `is changed`), and
`version` (Ansible's `version_compare`-equivalent `is version(...)` test).

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
