<!--
SPDX-License-Identifier: Apache-2.0
SPDX-FileCopyrightText: 2026 EON contributors
-->

# EON Account API — v0 (unstable)

What an EON Account server answers and what a client sends. The cryptography that fills
the opaque fields is in [cryptography.md](cryptography.md); what a server may keep is in
[transparency.md](transparency.md). Plan: madde 46.

## Transport

- **HTTPS** only, apart from loopback addresses used for testing.
- **Bodies** are JSON, UTF-8, at most 2 MiB.
- **Binary values** (OPAQUE messages, sealed boxes, tokens) are base64url without padding.
- **Times** are Unix seconds.
- **Authentication** is `Authorization: Bearer <token>`. In the tables, **✓** marks a
  route that requires the header.
- **Errors** are `{"error": code, "message": text}`
  ([error schema](../schemas/error.v0.schema.json)). The message is English, written for a
  log, and a client never shows it as it is. The codes are `invalid`, `taken`,
  `unauthorised`, `forbidden`, `not_found`, `conflict`, `too_large`, `rate_limited`,
  `closed` and `unavailable`.
- **No route reveals whether an account exists** for a given username, e-mail or phone,
  except registration itself, where `taken` is the point.

## Names

- **A username** is 3 to 32 characters long, lower case:
  `^[a-z0-9][a-z0-9._-]{1,30}[a-z0-9]$`. Clients lower-case what the person types. The
  username is public: it is what other people see. It cannot be changed in v0.
- **A login** is a username, or a verified e-mail address (the login contains `@`).
- **A contact** is an e-mail address, or a phone number in E.164 form (`+905551112233`).
- **A password** is never sent to the server in any form (OPAQUE). Its minimum length,
  10 characters, can therefore only be enforced by clients, and every EON client enforces
  it.

## Server

| Route | | Answer |
|---|---|---|
| `GET /v0/info` | – | `{api: 0, name, version, registration: "open" \| "closed", contacts: {email, phone}, suite}` ([schema](../schemas/info.v0.schema.json)) |
| `GET /v0/keys` | – | `{issuer, keys: [{kid, alg: "Ed25519", public}]}` — the keys that sign assertions |
| `GET /v0/transparency` | – | what this server keeps ([transparency.md](transparency.md)) |

## Signing up

| Route | | Body | Answer |
|---|---|---|---|
| `POST /v0/register/start` | – | `{username, request}` | `200 {response}`; `409 taken`; `403 closed` |
| `POST /v0/register/finish` | – | `{username, record, vault: {password_wrap, recovery_wrap, recovery_proof}, device}` | `201 {token, expires_at, username}`; `409 taken` if someone took the name in between |

- `request` is OPAQUE's `RegistrationRequest`; `response` is the `RegistrationResponse`;
  `record` is the `RegistrationUpload`.
- `device` is a name the person may read later in their session list, such as
  "EON Stream on Windows". It is at most 64 characters, and clients do not put a machine
  name or anything identifying in it.

## Signing in

| Route | | Body | Answer |
|---|---|---|---|
| `POST /v0/login/start` | – | `{login, request}` | `200 {login_id, response}` |
| `POST /v0/login/finish` | – | `{login_id, finalization, device}` | `200 {token, expires_at, username, vault: {password_wrap}}`; `401 unauthorised` |

- `login/start` answers the same way for a login that does not exist (OPAQUE's dummy
  response).
- A `login_id` is valid for 60 seconds and can be used once.
- A wrong password and an unknown login both end in `401` at `finish`.

## The session

| Route | | Body | Answer |
|---|---|---|---|
| `GET /v0/me` | ✓ | – | `{username, created_at, email, phone, vault: {entries, bytes}}` — `email` is `null` or `{address, verified}`, and `phone` likewise with `number` |
| `GET /v0/me/sessions` | ✓ | – | `{sessions: [{id, device, created_at, last_used, current}]}` — `last_used` is rounded down to the day |
| `DELETE /v0/me/sessions/{id}` | ✓ | – | `204` |
| `POST /v0/logout` | ✓ | – | `204` |

A session ends 180 days after its last use. A token that has ended, or is unknown, gets
`401`.

## Changing the password

| Route | | Body | Answer |
|---|---|---|---|
| `POST /v0/me/password/start` | ✓ | `{request}` | `200 {response}` |
| `POST /v0/me/password/finish` | ✓ | `{record, password_wrap}` | `204` — every other session ends |

The vault key does not change, so the vault keeps its contents. The client wraps the same
vault key under the new password key.

## E-mail and phone (optional)

| Route | | Body | Answer |
|---|---|---|---|
| `PUT /v0/me/email` | ✓ | `{address}` | `202` |
| `POST /v0/me/email/verify` | ✓ | `{code}` | `204`; `400 invalid`; `409 taken` if another account verified the address meanwhile |
| `DELETE /v0/me/email` | ✓ | – | `204` |
| `PUT /v0/me/phone` | ✓ | `{number}` | `202` |
| `POST /v0/me/phone/verify` | ✓ | `{code}` | as for e-mail |
| `DELETE /v0/me/phone` | ✓ | – | `204` |

`PUT` always answers `202`, and what the message contains depends on the address:

- **If no other account has verified the address**, the message carries a six-digit code,
  valid for 15 minutes and for five attempts.
- **If another account has verified it**, the message carries no code. It says that the
  address already belongs to an EON Account and names that account's username. **This is
  how "one account per address" is kept** without the server telling the person who asked
  whether the address is known.

A server without a way to send mail, or texts, answers `503 unavailable`.

## Recovery

**With the recovery key** (the vault is kept):

| Route | | Body | Answer |
|---|---|---|---|
| `POST /v0/recover/start` | – | `{username, proof, request}` | `200 {response, recovery_wrap}`; `401 unauthorised` |
| `POST /v0/recover/finish` | – | `{username, proof, record, password_wrap, device}` | `200 {token, expires_at, username}` — every other session ends |

**With a verified contact** (the vault starts empty, because the old one cannot be
opened):

| Route | | Body | Answer |
|---|---|---|---|
| `POST /v0/reset/request` | – | `{contact}` | `202`, always |
| `POST /v0/reset/verify` | – | `{contact, code}` | `200 {reset_token, username}`; `400 invalid` |
| `POST /v0/reset/start` | – | `{reset_token, request}` | `200 {response}` |
| `POST /v0/reset/finish` | – | `{reset_token, record, vault: {password_wrap, recovery_wrap, recovery_proof}, device}` | `200 {token, expires_at, username}` — every vault entry and every other session are deleted |

A reset token is valid for 15 minutes.

## The vault

Each entry is a sealed box the server cannot open, named by the application that wrote it
and a name it chose. Both names follow the pattern `^[a-z0-9][a-z0-9-]{0,31}$`.

| Route | | Body | Answer |
|---|---|---|---|
| `GET /v0/vault` | ✓ | – | `{entries: [{app, name, version, size, updated_at}], limits: {entry, entries, total}}` |
| `GET /v0/vault/{app}/{name}` | ✓ | – | `200 {version, data}`; `404` |
| `PUT /v0/vault/{app}/{name}` | ✓ | `{base_version, data}` | `200 {version}`; `409 {error: "conflict", version}`; `413 too_large` |
| `DELETE /v0/vault/{app}/{name}` | ✓ | – | `204` |

- **Versions.** `base_version` is the version the client last read, or `0` for a new
  entry. A stale base gets `409`, and the client then reads, merges and writes again.
- **Limits.** The reference server allows 1 MiB per entry, 64 entries and 8 MiB in all.
- **What the server checks.** Only that `data` has the sealed-box shape. Its contents are
  never inspected.

## Other services

| Route | | Body | Answer |
|---|---|---|---|
| `POST /v0/assertions` | ✓ | `{audience, share_username}` | `200 {assertion, expires_at}` |

- `audience` is the origin of the service the assertion is for, such as
  `https://community.example.org`.
- The assertion lasts five minutes. The service checks it against `GET /v0/keys` and then
  issues its own session. Its format is described in
  [cryptography.md](cryptography.md#assertions).

## The person's own data

| Route | | Answer |
|---|---|---|
| `GET /v0/me/export` ✓ | everything held about the account as one JSON document: the account, its contacts, its sessions (without token hashes) and every vault entry with its data |
| `DELETE /v0/me` ✓ | `204`; the account, its contacts, sessions and vault are deleted at once, and the username becomes free again after 30 days |

## What a server must not do

- Write an IP address, a user agent or a request line to disk. Rate limits are kept in
  memory.
- Keep a table, column or file that [transparency.md](transparency.md) does not list, or
  keep anything longer than it says.
- Read, index or summarise vault data.
- Reveal through any route other than registration whether a username exists. No route
  may reveal whether an e-mail address or phone number belongs to an account.
- Give a service an identifier that another service also receives.
