<!--
SPDX-License-Identifier: Apache-2.0
SPDX-FileCopyrightText: 2026 EON contributors
-->

# What an EON Account server keeps — v0

Every server serves `GET /v0/transparency`: a document listing what it keeps, why, for how
long, and who can read it, in English and Turkish
([schema](../schemas/transparency.v0.schema.json)). The application shows it as "What we
keep", so a person reads it from the server they actually use, not from a web page that
may describe another one.

**A server keeps nothing it does not list.** The reference server builds the document from
the same constants as its schema, and its tests fail if a table appears that the document
does not name.

## The reference server's list

| Kept | Why | For how long | Who can read it |
|---|---|---|---|
| **The account:** a random id, the username, when it was made, the OPAQUE record | To let you sign in. The record cannot be used to sign in as you, and it does not reveal your password. | Until you delete the account | The server |
| **The wrapped vault key** (twice: under your password and under your recovery key) and a hash of your recovery proof | To let you open your vault on a new device, and to recover it with your recovery key | Until you delete the account | Nobody: the server holds them but cannot open them |
| **Your e-mail address and phone number**, only if you added them, encrypted, plus a keyed hash for looking them up | To sign in with them and to reset a forgotten password | Until you remove them or delete the account | The server, only to send you a message you asked for |
| **A code sent to you**, as a hash | To check the code you type | 15 minutes, or until used | The server |
| **Sessions:** a hash of each token, the device name your application gave, when it started, the day it was last used | To keep you signed in, and to let you see and end your sessions | Until you sign out or end it, or 180 days after its last use | You and the server |
| **Your vault:** encrypted entries, their size, version and when each last changed | So your progress and scores follow you to another device | Until you delete the entry or the account | **Only you.** The server cannot decrypt it |
| **A deleted username**, as a hash | So nobody can take your name the day you leave and pretend to be you | 30 days | The server |

**Held in memory only, never written:**

- the attempts used to slow down guessing, counted per login, per contact and per network
  address;
- a sign-in in progress (up to 60 seconds).

All of it is gone when the server restarts.

**Never kept:**

- your password, in any form;
- network (IP) addresses;
- browser or application identifiers (user agents);
- request logs;
- what you watch, read or download;
- anything about anything outside EON applications;
- device fingerprints;
- advertising identifiers.

## A server that keeps more

An operator may be required by law to keep more than this list. For example, a hosting
provider in Turkey must keep traffic records under Law 5651; whether an account server is
one is an open question (plan madde 46j). Such a server **must** list the extra data in
its document, with the reason and the period. A client shows the list as the server gives
it.
