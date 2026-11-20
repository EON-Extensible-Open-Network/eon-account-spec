<!--
SPDX-License-Identifier: Apache-2.0
SPDX-FileCopyrightText: 2026 EON contributors
-->

# EON Account — contracts

EON Account is one account for all of EON, and it puts anonymity first: a username and a
password are enough. This repository holds what a server and a client agree on, written
down so that anyone can implement either side.

| Contract | Document | Schema |
|---|---|---|
| The HTTP API | [docs/api.md](docs/api.md) | [error](schemas/error.v0.schema.json), [info](schemas/info.v0.schema.json) |
| OPAQUE sign-in, keys, the encrypted vault, assertions for other services | [docs/cryptography.md](docs/cryptography.md) | [assertion](schemas/assertion.v0.schema.json) |
| What a server keeps, published by the server itself | [docs/transparency.md](docs/transparency.md) | [transparency](schemas/transparency.v0.schema.json) |

**Version 0 is unstable.** Every change comes with valid and invalid examples in
`examples/` and a line in [CHANGELOG.md](CHANGELOG.md).

```
npm ci
npm run validate     # every example in examples/valid passes, every one in examples/invalid fails
```

- The client library is
  [eon-account-client](https://github.com/EON-Extensible-Open-Network/eon-account-client).
- The reference server is
  [eon-account-server](https://github.com/EON-Extensible-Open-Network/eon-account-server).
- The decision is madde 46 of the
  [EON plan](https://github.com/EON-Extensible-Open-Network/eon-docs).

The project runs no server until the plan's legal gate (madde 30) is met.

Licence: [Apache-2.0](LICENSE). It is permissive on purpose, so that other
implementations are welcome.
