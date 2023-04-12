---
description: Documentation for Turso, an edge-hosted, distributed database based on libSQL, an open-source and open-contribution fork of SQLite.
keywords:
  - turso
  - database
  - documentation
  - libsql
  - sqlite
---

# Turso documentation

## About Turso

[Turso] is an edge-hosted, distributed database based on [libSQL], an
open-source and open-contribution fork of [SQLite]. It was designed to minimize
query latency for applications where queries come from anywhere in the world. In
particular, it works well with edge functions provided by cloud platforms such
as CloudFlare, Netlify, and Vercel, by putting your data geographically close to
the code that accesses it.

## New to Turso?

To get started, please see the following steps of the getting started guide:

* [Install the CLI](tutorials/get-started-turso-cli/step-01-installation)
* [Create your first database](http://localhost:3000/tutorials/get-started-turso-cli/step-03-create-database)

## Recommended reading

To learn more about Turso, we suggest the following path through this
documentation:

- Understand the [concepts] behind Turso.
- Follow a [walkthrough of the Turso CLI] to experience how it works.
- Learn how to access Turso using the [libSQL client libraries].


[Turso]: https://turso.tech/
[libSQL]: https://libsql.org/
[SQLite]: https://sqlite.org/
[concepts]: /concepts
[walkthrough of the Turso CLI]: /tutorials/get-started-turso-cli
[libSQL client libraries]: /reference/client-access
