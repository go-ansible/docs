# vault

[![CI](https://github.com/go-ansible/vault/actions/workflows/ci.yml/badge.svg)](https://github.com/go-ansible/vault/actions/workflows/ci.yml)

`github.com/go-ansible/vault` implements the [Ansible Vault](https://docs.ansible.com/ansible/latest/vault_guide/index.html)
1.1 file format: **AES-256 in CTR mode**, a key derived with
**PBKDF2-HMAC-SHA256** (10 000 rounds), and an **encrypt-then-MAC**
HMAC-SHA256 authentication tag, hex-encoded and wrapped at 80 columns.

The wire format is fixed by the reference implementation
(`ansible.parsing.vault.VaultAES256` in `ansible-core`) and reproduced here
byte-for-byte, so a file written by this package decrypts with the real
`ansible-vault` command, and vice versa.

## API

```go
func IsVault(data []byte) bool

func Encrypt(plaintext []byte, password string, vaultID string) (string, error)

func Decrypt(vaultText string, password string) ([]byte, error)

func MaybeDecrypt(data []byte, password string) ([]byte, error)

func UnmarshalYAML(data []byte, password string, out any) error
```

- `IsVault` checks the first line for the `$ANSIBLE_VAULT` header — cheap
  enough to call on every value read from a playbook or inventory to decide
  whether it needs decrypting at all.
- `Encrypt` generates a random 32-byte salt, derives the AES key / HMAC key /
  CTR counter from `password` and that salt, PKCS7-pads and encrypts
  `plaintext`, and returns the full vault text (header line plus the
  80-column-wrapped hex body). Passing `vaultID` reproduces `ansible-vault
  --vault-id`'s `$ANSIBLE_VAULT;1.1;AES256;vaultID` header.
- `Decrypt` parses the header, verifies the cipher is `AES256` (the only one
  implemented), recomputes the HMAC over the ciphertext with
  [`crypto/subtle.ConstantTimeCompare`](https://pkg.go.dev/crypto/subtle#ConstantTimeCompare)
  and fails closed with `ErrHMACMismatch` on any mismatch — almost always a
  wrong password — before ever attempting to decrypt.
- `MaybeDecrypt` is what a caller reading YAML files actually wants:
  plaintext through untouched, ciphertext decrypted. Real Ansible accepts
  an encrypted file anywhere it accepts a plaintext one, and the call site
  cannot know in advance which it has. Encrypted content with **no**
  password is an error rather than a silent pass-through — handing the
  ciphertext back as content would surface much later as an unreadable
  YAML parse, far from the cause.
- `UnmarshalYAML` decodes YAML carrying secrets in **either** shape real
  Ansible accepts — the whole file encrypted, or individual
  `!vault`-tagged scalars inside an otherwise-plaintext file — and
  otherwise decodes exactly as `yaml.Unmarshal` would. The second shape
  is what `encrypt_string` produces, and how one secret lives in a vars
  file everybody else can read:

  ```yaml
  api_key: !vault |
            $ANSIBLE_VAULT;1.1;AES256
            3865...
  region: eu-west
  ```

  Each tagged scalar is decrypted in place, so the caller gets a plain
  value and never sees the ciphertext. A missing password is an error
  only if something actually needs decrypting.

## Interoperable with real Ansible, in both directions

Verified rather than assumed, against real ansible-core 2.21.4: each tool
decrypts and views what the other encrypted, for both `decrypt` and `view`.
That matters because a vault format that is *almost* right is worse than
none — it would encrypt secrets nobody else can ever read back.

## Where it is wired in

Every YAML file the ecosystem loads goes through `MaybeDecrypt`, so an
encrypted one works anywhere a plaintext one does:

- [`inventory`](inventory.md): `LoadWithVault` — the inventory file itself,
  and any `group_vars`/`host_vars` merged alongside it. `group_vars/*/vault.yml`
  is the canonical place real Ansible users keep secrets.
- [`playbook`](playbook.md): `ParseFileWithVault` for what a parse pulls in
  (the playbook, `vars_files`, a role's own `defaults`/`vars`/`tasks`/`meta`),
  and `Engine.VaultPassword` for `include_vars`, which is read at run time.
- [`cli`](cli.md): `ansible-playbook --vault-password-file`, plus
  `--ask-vault-password` and the `--vault-pass-file`/`--ask-vault-pass`
  spellings. The password is resolved before anything is read, since the
  inventory itself may be encrypted, and only when the caller says one
  exists — a plaintext run never stops to prompt.

`Load` and `ParseFile` are the same calls with an empty password, so
plaintext loading is unchanged.

## Example

```go
package main

import (
	"fmt"
	"log"

	"github.com/go-ansible/vault"
)

func main() {
	text, err := vault.Encrypt([]byte("super-secret-value"), "hunter2", "")
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(text) // $ANSIBLE_VAULT;1.1;AES256\n...

	plain, err := vault.Decrypt(text, "hunter2")
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(string(plain)) // super-secret-value
}
```

## What is not implemented

Only cipher `AES256` (Vault format 1.1) is implemented — that has been the
only cipher `ansible-vault` writes since Ansible 2.3, so it is also the only
one worth reading. `Decrypt` returns an explicit error naming the cipher if it
ever encounters anything else, rather than silently producing garbage.

`ansible-vault` implements every subcommand real Ansible has: `encrypt`,
`decrypt`, `view`, `rekey`, `encrypt_string`, `create` and `edit`.

`create` and `edit` open `$EDITOR` (defaulting to `vi`, split on spaces so
`EDITOR="code -w"` works). `create` refuses an existing file rather than
destroying it, and is gated on stdout being a terminal with the same
`--skip-tty-check` escape real Ansible offers. `edit` keeps the file's own
vault id, and an edit that changes nothing leaves the file untouched —
re-encrypting identical content would rewrite it with a fresh salt and look
like a change in version control.

The temporary file those two use holds the secret in the clear, so it is
created `0600`, lives in the system temp directory rather than beside the
target — which may be inside a repository — and is removed on every path
out, including when the editor fails.

These two are the only part of this ecosystem whose behaviour was taken
from reading real Ansible's source rather than from running it: both
refuse to start without a terminal, so there is nothing to observe from a
script. Everything around them is verified against real `ansible-vault`.

`encrypt_string` deliberately shipped *after* the reading side. Writing a
`!vault` scalar this ecosystem could not read back would have been worse than
not writing one at all — so `UnmarshalYAML` and its wiring landed first, and
both directions are verified: real Ansible runs a playbook whose `vars_files`
is what this wrote, and this runs one whose `vars_files` is what real
`ansible-vault` wrote.
