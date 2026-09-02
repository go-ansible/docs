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
