<!--
SPDX-License-Identifier: Apache-2.0
SPDX-FileCopyrightText: 2026 EON contributors
-->

# Contributing to eon-account-spec

The organisation-wide guide —
[`.github/CONTRIBUTING.md`](https://github.com/EON-Extensible-Open-Network/.github/blob/main/CONTRIBUTING.md)
— applies: Code of Conduct, DCO sign-off (`git commit -s`), Conventional Commits with the
`spec` scope.

- **Add examples with every contract change.** Each change needs at least one file in
  `examples/valid` and one in `examples/invalid`, named
  `<assertion|error|info|transparency>-*.json`.
- **A cryptographic change is a new suite**, never an edit to an existing one.
- **Say what it costs:** who has to change, and what happens to those who do not.
- Contributions are licensed under Apache-2.0.
