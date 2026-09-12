# migrate

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

Versioned schema migrations for `std.sql` drivers — sqlx-migrate's and
alembic's shape, in novo-lang.

A migration is a value, a plan is a value, and **running one is the
caller's loop.**  This package reads the directory, computes the
checksums, works out what would run and in what order, refuses when
something is wrong, and produces the SQL for the version table.  It does
not execute a statement.

Four modules, and a reader should know which one they are on.

| surface | module | reach for it when |
| --- | --- | --- |
| the **plan** | `mgplan` | anything. Start here |
| the **version table** | `mgstate` | you need the SQL, or to read its rows |
| the **directory** | `mgdir` | your migrations are files |
| the **refusals** | `mgerror` | something stopped |

## Adding it, and checking it

```bash
novo pkg add migrate             # into your novo.toml
novo pkg build                   # type- and effect-check the package
novo test --isolate tests/mgplan_tests.nv
```

`novo test` is red today and that is the point of the release: every
assertion fails with `not implemented: migrate.<module>.<fn>`.  They
turn green one at a time as bodies land.

## The one example that will work

```novo
use mgdir
use mgplan
use mgstate

// What would run, and nothing runs.
fn pending(dir: Str, applied_rows: [[Str]]) -> [Str] [fs]
    match mgdir.read_auto(dir)
        Err(f) => [mgerror.describe(f)]
        Ok(ms) =>
            match mgstate.rows_to_applied(applied_rows)
                Err(f) => [mgerror.describe(f)]
                Ok(applied) =>
                    match mgplan.plan(ms, applied, mgplan.default_options())
                        Err(f) => [mgerror.describe(f)]
                        Ok(p)  => mgplan.render(p)
```

The caller ran `mgstate.select_sql(...)` against its own database and
handed the rows in.  To apply the plan it walks `p.steps`, runs each
one's `sql`, and inserts the row `mgstate.record_params` builds.

## The layer, and why

`host`, and **exactly one module earns it**: `mgdir`, which lists a
directory and reads files.  `mgplan`, `mgstate` and `mgerror` are `[]`
throughout.

Nothing here prints, and nothing here connects to a database.

## The load-bearing interface

`mgplan.plan` — **it takes what the directory holds and what the
database has applied, and answers an ordered list of statements, or a
fault.  It runs nothing.**

```novo norun:pseudo
pub fn plan(available: [MgMigration], applied: [MgApplied],
            options: MgPlanOptions) -> Result<MgPlan, mgerror.MgFault>
```

Three things fall out of that signature, and each is worth more than it
looks:

- **A dry run is the same call as a real one.**  There is no second code
  path, so the thing a deployment reviews and the thing that runs cannot
  drift apart.  "The dry run worked and the real one did not" is the
  failure mode every migration tool with two paths has.
- **Every refusal happens before the first statement.**  A runner that
  discovered a changed checksum halfway has left half of version 7
  applied and version 7 unrecorded, and nothing from the outside can
  tell which half.
- **The order is assertable with no database.**
  `tests/mgplan_tests.nv` builds a directory's worth of migrations,
  applies some, and asserts on what would run — no database, no file, no
  clock.

## Why this package does not run anything

Two reasons, and the first is the language as it stands today.

**A library that called `dyn Database` would be charged the union of
every `Database` impl in the *consuming* program.**  SPEC § 5.6 is
explicit: a `dyn Trait` call costs the union over every impl, because
the call may land on any of them.  So:

```novo norun:pseudo
fn run(db: dyn Database) -> Str [fs]      // in a library
    match db.exec("...")                   // charged [fs, io] today
```

fails to compile with `[fs]`, needs `[fs, io]` in a program whose only
engine is the standard library's `SqliteDb` — and needs `[fs, io, net,
time]` the moment that program adds postgres-nv.  **A library cannot
declare a row that depends on what its consumer links.**  A migration
runner that owned the loop would therefore be a library that stops
compiling when somebody adds an engine, which is the opposite of
engine-agnostic.

The contract change that would lift it is the same one `Connection`
records in its own header: **an effect parameter on `Database`** (`trait
Database[e]`, SPEC § 5.6), so each impl supplies what it costs and a
generic runner binds it — `fn run<D: Database[e]>(db: D, p: MgPlan) ->
… [e]`.  That is a change to a vendor-connect contract and belongs in a
feature file; this package names it rather than working around it, and
it is the widening this lane found.

**The second reason is that the plan shape is better anyway**, and the
three bullets above are why.  When the contract change lands, this
package grows a convenience runner *on top of* `plan` — and `plan` stays
exactly as it is.

## The refusal this package exists for

**A migration that has been applied and then edited.**

It is invisible without a checksum: the developer who edited it has the
new schema, because they re-ran from an empty database; every machine
that already applied the old one has the old schema; and the version
table says both are up to date.  It surfaces months later as a column
that exists on staging and not in production.

`MgMigration.checksum` is the SHA-256 of the migration's `up`, recorded
in the version table when it runs and compared on every plan afterwards.
`mgerror.MgChecksumChanged` carries **both** digests, because a report
that said only "changed" leaves a reader unable to tell an edit from a
line-ending conversion.

The checksum is over `up` **alone**.  Editing a `down` that has never
run changes nothing that has happened, and refusing for it would make
fixing a broken rollback impossible without rewriting history.

Three more refusals, all at planning time and all needing a person:

| | |
| --- | --- |
| two migrations claim one version | whichever ran first would be recorded, and the other would never run on any machine that had already migrated |
| a migration arrives below the highest applied | two people branched and merged in either order, and the schema's final shape must not depend on which merged second — `allow_out_of_order` is the switch, and it is off |
| the version table names a migration the directory does not have | a runner that ignored it cannot answer "is this database up to date" at all |

And one refusal on the way down: **a rollback through an irreversible
migration does not start.**  A `DROP COLUMN` has no inverse that keeps
the data, so `MgMigration.down` is allowed to be empty — and a rollback
that stopped halfway would leave a schema nobody has a name for.

## A schema version is a sequence, not a semver

The plan asked for this in writing, so here it is.

changelog-nv takes **semver-nv** because a release version *is* a
semantic version: three numbers with rules about which one moves when, a
pre-release ordering, and a caret range that decides compatibility.

A schema version is none of that.  It is `20260911143000` or `0007`; it
compares by ordinary ordering; it has no pre-release and no build
metadata; nothing is "compatible" with it; and the only question ever
asked of two of them is which came first.  Modelling it as a semantic
version would invite exactly the question that has no meaning here —
whether migration `2.0.0` is a breaking change.

So `MgVersion` is a `Str`, and `mgplan.compare_versions` is where the
sequence's own rule lives.  **That rule is public because there is
exactly one way to get it wrong and it is late:** comparing `"10"` and
`"9"` as text sorts `10` first and runs the migrations in the wrong
order on exactly the tenth one — by which point every test suite has
passed.

## The version table

Six columns, and its shape is a compatibility promise the moment
anybody uses it:

| column | type | |
| --- | --- | --- |
| `version` | `TEXT PRIMARY KEY` | the sequence |
| `name` | `TEXT NOT NULL` | the human name |
| `checksum` | `TEXT NOT NULL` | SHA-256 of `up`, lowercase hex |
| `applied_at` | `TEXT NOT NULL` | the caller's clock reading |
| `took_ms` | `INTEGER NOT NULL` | 0 when unmeasured |
| `succeeded` | `INTEGER NOT NULL` | 0 or 1 |

**`TEXT` for the version and not an integer**, deliberately: the
timestamp convention writes a number nobody wants to read, and the
sequence convention writes `0007`, whose leading zeros an integer column
silently drops — after which the file name stops matching its row.

**`TEXT` for the time** because this package has no clock and no
calendar.  The value is the caller's; a caller that wants it comparable
passes ISO-8601, which sorts correctly as text, which is why ISO-8601 is
written that way.

**The table name is an argument and is checked rather than escaped.**
Two applications sharing a database need two tables, and sqlx and
alembic each chose a different default.  `mgstate.is_safe_table_name`
refuses anything that would need quoting — because quoting an identifier
is dialect-specific, and query-builder-nv's own notes record that even
sql-engine-nv's tokenizer cannot do it.

**Two processes migrating at once is the failure nobody tests for**, and
it is what happens the first time a deployment starts three copies of a
service at the same moment: all three read an empty version table, all
three run migration 1.  There is no portable advisory lock —
PostgreSQL has `pg_advisory_lock`, MySQL has `GET_LOCK`, SQLite has the
file lock it already takes — so `mgstate.lock_sql` answers the statement
for a named style and **empty** for a driver with no such thing.  A
caller that gets empty knows it has to arrange exclusion itself, rather
than assuming it has some.

## The directory, both conventions

A port that picked one convention could not read the other's directory,
so both are here and `mgdir.detect` guesses — because the realistic
first use of this package is pointing it at a directory somebody else
created.

- **paired** — `0001_create_users.up.sql` and `.down.sql`.  sqlx's
  default, and the one where a `down` is a first-class file.
- **single file** — `0001_create_users.sql` with `-- +migrate Up` and
  `-- +migrate Down` markers.  One file, which keeps a migration and its
  inverse together — and which means a **mistyped marker silently
  produces a migration with no `down`**, so `mgdir.split_markers`
  reports which markers it actually saw rather than assuming.

A file that does not parse **stops the read** rather than being skipped.
A migration silently skipped is a migration that runs on one machine and
not on another, which is the class of failure this whole package exists
to make impossible.

The read is sorted by `mgplan.compare_versions` and **not** by the
directory: a filesystem answers names in whatever order it likes, and
even a sorted one puts `10` before `9`.

## One dependency, and two refusals

**crypto-nv**, for SHA-256, and it is the whole reason this package can
refuse anything.

**Not semver-nv** — see above.

**Not sql-engine-nv.**  This package does not parse SQL: a migration's
body is text the driver runs, and a parser here would refuse valid
statements for whichever dialect it did not implement while adding
nothing — the driver is going to parse it anyway, and its error message
is the one a caller can act on.  It is also why an `MgStep` is one
migration and not one statement: splitting SQL into statements needs a
parser, and a driver that cannot run several statements at once has a
caller who will need the dialect to split them.

## What is out of scope, out loud

**Running anything** — see above, and it is the first thing to revisit
when `Database` takes an effect parameter.

**Generating migrations from a schema diff.**  alembic's autogenerate
needs to introspect a live database and compare it with a model
definition, which is two packages this grid does not have yet.

**Transactional migrations.**  Whether a migration runs inside a
transaction is the driver's decision and the dialect's — PostgreSQL can
do DDL in one and MySQL cannot — so the caller decides, and
`MgApplied.succeeded` is what records a migration that failed partway
when it could not.

**Seed data and repeatable migrations.**  Flyway's `R__` prefix runs a
file whenever its checksum changes, which is the opposite of this
package's central rule; it is a real feature and it wants its own
decision rather than an inversion of this one.

## The reference implementations

sqlx-migrate for the directory conventions, the checksum rule and the
version table's shape; alembic for the up/down/status/stamp vocabulary.

The implementation lane's gate is a real database and a real directory:
a corpus of migration directories from both conventions, applied against
the standard library's `SqliteDb`, with the version table read back and
compared.

## Status

Interface only.  Four modules, 42 public functions, every body a
`todo()`.

- `novo pkg build` — clean, 4 modules checked.
- `novo test` — two suites, all red, every failure `not implemented`.
- `scripts/shard_audit.sh --strict` — `effect-budget`, `dep-layer`,
  `no-discharge-in-core`, `doc-examples` and `docs-pub` green; `test`
  red by design.
