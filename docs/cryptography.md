<!--
SPDX-License-Identifier: Apache-2.0
SPDX-FileCopyrightText: 2026 EON contributors
-->

# EON Account cryptography — v0 (unstable)

Every value in this document is fixed for v0. A change is a new `suite`, announced in
`GET /v0/info`.

## The OPAQUE suite: `opaque-3dh-ristretto255-sha512-argon2id-v0`

Sign-up and sign-in use OPAQUE ([RFC 9807](https://www.rfc-editor.org/rfc/rfc9807)), so the
password never reaches the server.

| Part | Value |
|---|---|
| OPRF | ristretto255-SHA512 |
| Key exchange | 3DH over ristretto255, SHA-512 |
| Key stretching | Argon2id, m = 65536 KiB, t = 3, p = 1, run on the client |
| Credential identifier | `u:` followed by the username |
| Identities, context | the defaults (the client's and server's public keys; empty context) |

The reference implementation is [`opaque-ke`](https://crates.io/crates/opaque-ke) 4. Its
`ServerSetup` belongs to the server and never changes. Replacing it invalidates every
account.

## Keys a client holds

```
export_key       from OPAQUE, at sign-up and at every sign-in (64 bytes)
password_key   = HKDF-SHA256(ikm = export_key, salt = none, info = "eon-account v0 password key")   32 bytes
vault_key      = 32 random bytes, made once at sign-up
recovery_key   = 20 random bytes, made once at sign-up, shown once
recovery_wrap_key = HKDF-SHA256(ikm = recovery_key, info = "eon-account v0 recovery key")       32 bytes
recovery_proof    = HKDF-SHA256(ikm = recovery_key, info = "eon-account v0 recovery proof")     32 bytes
```

- The server stores `SHA-256(recovery_proof)` and compares the hash of what it receives in
  constant time.
- **How the recovery key is shown.** It is written as RFC 4648 base32 (`A–Z`, `2–7`), 32
  characters in eight groups of four: `ABCD-EFGH-…`. Clients accept it with or without
  hyphens, in either case.

## Sealed box

```
box = 0x01 ‖ nonce (24 random bytes) ‖ XChaCha20-Poly1305(key, nonce, plaintext, aad)
```

The associated data says what the box is for. A box moved anywhere else, or between
accounts, therefore fails to open:

| Box | Key | Associated data |
|---|---|---|
| `password_wrap` | `password_key` | `eon-account v0 wrap password ` ‖ username |
| `recovery_wrap` | `recovery_wrap_key` | `eon-account v0 wrap recovery ` ‖ username |
| a vault entry | `vault_key` | `eon-account v0 vault ` ‖ username ‖ ` ` ‖ app ‖ `/` ‖ name |

`password_wrap` and `recovery_wrap` each hold the 32-byte `vault_key`.

**What the server can see.** It sees an entry's length and the time it last changed.
Clients that care may pad. EON Stream rounds every entry up to a multiple of 4 KiB, and the
padding is ignored after decryption: the plaintext is JSON followed by zero bytes.

## Sessions

- **Tokens.** A token is 32 random bytes. The server stores `SHA-256(token)`, so a copy of
  the database is not a set of logins.
- **The vault key on the device.** A client keeps the vault key on the device so that a
  restart does not ask for the password again. It keeps it in a sealed box under
  `HKDF-SHA256(ikm = token, info = "eon-account v0 device key")`, with the associated data
  `eon-account v0 device`. Signing out deletes the box and the token together.

## Assertions

```
assertion = base64url(payload) "." base64url(signature)
payload   = the JCS (RFC 8785) serialisation of
            {"v": 0, "iss": issuer, "aud": audience, "sub": pairwise, "iat": t, "exp": t + 300, "kid": key id}
            plus "name": username, only when share_username was true
signature = Ed25519(server assertion key, payload bytes)
pairwise  = base64url(first 16 bytes of HMAC-SHA256(pairwise secret, account id ‖ 0x00 ‖ audience))
```

- **Fields.** `issuer` is the server's own origin. The payload has no floating-point values
  and no non-ASCII keys, so JCS reduces to sorted keys and no whitespace.
- **Checking an assertion.** A service checks it in five steps:
  1. The signature is valid against a key from the issuer's `GET /v0/keys`.
  2. `aud` is the service's own origin.
  3. `exp` has not passed, allowing 60 seconds of clock difference.
  4. `v` is `0`.
  5. The issuer is one the service has decided to trust.
- **What a service learns.** It learns `sub` and, if the person chose, `name`. It does not
  learn the account id, a contact or a session.

The [assertion payload schema](../schemas/assertion.v0.schema.json) describes the decoded
payload.
