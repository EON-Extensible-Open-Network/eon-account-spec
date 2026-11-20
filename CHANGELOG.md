<!--
SPDX-License-Identifier: Apache-2.0
SPDX-FileCopyrightText: 2026 EON contributors
-->

# Changelog

## 0.0.0 — 20 November 2026

The first draft (plan madde 46):

- **The HTTP API:**
  - sign-up, sign-in and sessions;
  - password change;
  - optional e-mail and phone;
  - recovery with the recovery key or a verified contact;
  - the vault;
  - assertions;
  - export and deletion.
- **The cryptography:** the OPAQUE suite `opaque-3dh-ristretto255-sha512-argon2id-v0`, key
  derivation, the sealed box, and pairwise assertions.
- **The transparency document** every server serves.
- **Schemas** for errors, server info, assertion payloads and the transparency document.
  Each has valid and invalid examples, checked in CI.
