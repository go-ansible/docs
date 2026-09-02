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

## Filter and test library

Filters and tests are registered on top of gonja's Jinja2 built-ins, not
instead of them. As of this writing the Ansible-specific set includes:

**Filters** — `to_json`/`to_nice_json`/`from_json`, `to_yaml`/`to_nice_yaml`/`from_yaml`,
`regex_replace`/`regex_search`/`regex_findall`/`regex_escape` (backed by
[`go-regexp/engine`](https://github.com/go-regexp/engine), a PCRE-compatible
engine, since Ansible's regex filters rely on Python `re` semantics that
Go's `regexp` doesn't fully match), `bool`, `mandatory`, `ternary`, `combine`,
`dict2items`/`items2dict`, `type_debug`, `quote`, `basename`/`dirname`,
`b64encode`/`b64decode`, `md5`/`sha1`/`hash`.

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
