# Changelog

All notable changes to migrate are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.2 — 2026-09-15

- README rewritten to the package README style guide (docs/writing-a-readme.md).
- `MgFault` now declares the `impl Error` its own `Result` positions
  require.  `Result<T, E>` has carried the bound `E: Error` since SPEC
  § 3.4, and the compiler enforced it only when `E` was declared in the
  module that named it — so `Result<_, mgerror.MgFault>` was accepted
  across modules with no impl anywhere.  The impl is the signature this
  package always meant; nothing else about the interface changed.

## 0.0.1 — 2026-09-11

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `mgplan` — a migration and a plan as values, `compare_versions` (the
  sequence's own rule), the checksum over `up`, plans for up, down and
  redo, and a status report that answers where a plan refuses.
- `mgstate` — the six-column version table: its DDL, its select, its
  insert with the driver's own placeholder, the row parsing, and the
  advisory lock that is empty for a driver without one.
- `mgdir` — both directory conventions, the marker split that reports
  what it found, the name parsing that refuses rather than skips, and
  `create` for a `new` command.
- `mgerror` — ten faults, every one of them raised at planning time,
  and `needs_a_person` for the four that are two people's work
  disagreeing.

### Known

- **The load-bearing interface is `mgplan.plan`**, which answers
  ordered statements and runs nothing.  A dry run is the same call as
  a real one, every refusal happens before the first statement, and
  the order is assertable with no database.
- **This package does not run anything, and the first reason is the
  language**: a library that called `dyn Database` is charged the
  union of every impl in the CONSUMING program, which a library cannot
  know — `[fs, io]` with only `SqliteDb` linked and `[fs, io, net,
  time]` once postgres-nv is added.  The contract change that would
  lift it is an effect parameter on `Database`, which is the widening
  this lane found.  The second reason is that the plan shape is better
  anyway.
- **A migration edited after it was applied is the refusal this
  package exists for**, and the fault carries both digests.
- **The checksum is over `up` alone**, so a broken `down` can be
  fixed.
- **A schema version is a sequence and not a semver**, which is why
  semver-nv is not a dependency and why `compare_versions` is public:
  comparing `"10"` and `"9"` as text fails on exactly the tenth
  migration.
- **The table name is checked, not escaped**, because quoting an
  identifier is dialect-specific.
- **Both directory conventions are read** and `detect` guesses, and a
  file that does not parse stops the read rather than being skipped.
- **One dependency**, crypto-nv, for SHA-256.  Not semver-nv and not
  sql-engine-nv.
- **Out of scope and said so**: running, schema-diff generation,
  transactional migrations, and repeatable migrations.
