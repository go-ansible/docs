# Contributing

## Standards

- **Pure Go, `CGO_ENABLED=0`.** No dependency on a C library or an external
  interpreter, anywhere in the module graph.
- **Compatibility is verified against the real thing.** A change to `vault`'s
  wire format, `inventory`'s parsing, `vars`'s precedence order, or
  `template`'s rendering rules should be checked against `ansible-core`'s
  actual behavior — the reference implementation, not this project's own
  prior code, is the source of truth for what "correct" means.
- **CI is the gate.** Every one of `vault`/`inventory`/`vars`/`template` runs
  `go vet`, `go build`, and `go test` on Linux and macOS, plus native
  amd64/arm64 and a QEMU-emulated riscv64/loong64/ppc64le/s390x matrix, on
  every pull request. A change that doesn't pass CI doesn't merge.

## Working on the docs

This site is built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/)
and versioned with [mike](https://github.com/jimporter/mike). Locally:

```sh
pip install -r requirements.txt
mkdocs serve
```

opens a live-reloading preview at `http://127.0.0.1:8000/`. `mkdocs build
--strict` is what CI runs — a broken internal link or an unknown config key
fails the build rather than logging a warning nobody reads, so run it before
opening a pull request.

Pushes to `main` publish automatically via `.github/workflows/docs.yml`,
which runs `mike deploy --push --update-aliases 0.1 latest` — no need to run
`mike` by hand.

## Working on a library

Each component (`vault`, `inventory`, `vars`, `template`) is its own
repository with its own `go.mod`, so it can be developed and tested in
isolation:

```sh
git clone https://github.com/go-ansible/<component>
cd <component>
CGO_ENABLED=0 go test ./...
```

Pull requests are welcome. Since compatibility with `ansible-core`'s real
behavior is the bar, a test that demonstrates the reference behavior — a
transcript from `ansible-vault`, an inventory file plus `ansible-inventory
--list` output, and so on — is the most useful thing to include alongside a
fix.
