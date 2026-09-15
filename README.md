# migrate

A schema migration is one numbered change to a database's schema, kept
in a file, applied once, and recorded in a table so that it is never
applied twice. This package works out which of them a database still
needs and in what order, and it produces the SQL for the table that
records them. It runs nothing: the caller's own driver executes the
statements. The shape is
[sqlx's migrator](https://docs.rs/sqlx/latest/sqlx/migrate/index.html)
for the files, the checksum and the table, and
[alembic](https://alembic.sqlalchemy.org/en/latest/) for the up, down,
status and stamp vocabulary.

**Status: NOT IMPLEMENTED — interface only.** Every function is
declared with its full signature, but every body is a `todo()` that
panics when called. The package is published so its design can be
reviewed and depended on before it is implemented. Version 0.1.0 will
be the first working release.

## What it is

A **migration** has a **version**, a name, the statements that apply
it, called the **up**, and the statements that undo it, called the
**down**. The version is a sequence: `0007`, or a timestamp such as
`20260911143000`. It is not a semantic version. The only question ever
asked of two versions is which came first.

The **version table** is a table in the database being migrated. It
holds one row per migration that has been applied. A migration is
pending when it is in the directory and not in that table.

The **checksum** is the SHA-256 of a migration's up, recorded in the
version table when the migration runs and compared against the file on
every plan afterwards. It is what catches a migration that has been
applied and then edited. That edit is invisible without it: the person
who made it has the new schema, because they re-ran from an empty
database, and every machine that already applied the old text has the
old schema, while the version table says both are up to date.

A **plan** is the ordered list of statements a run would execute, as a
value. `mgplan.plan` answers one, or a refusal. It does not execute
anything. Three things follow from that. A dry run is the same call as
a real one, so the two cannot drift apart. Every refusal arrives
before the first statement, so no database is left half-migrated.
And the order can be asserted in a test with no database, no file and
no clock in it.

A **rollback** runs the downs in reverse. A migration whose down is
empty is **irreversible**: a `DROP COLUMN` has no inverse that keeps
the data. A rollback that would pass through one is refused before it
starts.

Only one module here touches the machine. `mgdir` lists a directory
and reads files, and that is the whole of the package's `[fs]`.
`mgplan`, `mgstate` and `mgerror` perform no input or output at all.
Nothing here prints, and nothing here connects to a database.

The version table has six columns, and its shape is a compatibility
promise from the moment anything uses it.

| Column | Type | Contents |
| --- | --- | --- |
| `version` | `TEXT PRIMARY KEY` | The sequence, as written in the file name |
| `name` | `TEXT NOT NULL` | The human name |
| `checksum` | `TEXT NOT NULL` | SHA-256 of the up, lowercase hex |
| `applied_at` | `TEXT NOT NULL` | The caller's clock reading |
| `took_ms` | `INTEGER NOT NULL` | 0 when the runner did not measure |
| `succeeded` | `INTEGER NOT NULL` | 0 or 1 |

The DDL is written to the intersection of SQLite's and PostgreSQL's
dialects: `TEXT`, `INTEGER`, `NOT NULL` and `PRIMARY KEY`, and nothing
else.

A directory is in one of two conventions, and `mgdir.detect` guesses
which from what the directory holds.

| Convention | Files | Where the down lives |
| --- | --- | --- |
| Paired | `0001_create_users.up.sql` and `0001_create_users.down.sql` | Its own file |
| Single file | `0001_create_users.sql` | After a `-- +migrate Down` marker line in the same file |

A version is everything before the first separator in the file name,
and the separator is `__` or `_`. `V2__add_index.up.sql` is version
`2`, named `add_index`.

## Install

```
novo pkg add migrate
```

## Example

```novo
use mgdir
use mgerror
use mgplan
use mgstate

fn main() [io, fs]
    // The rows the caller's own driver read back after running
    // `mgstate.select_sql(mgstate.default_table())`. Empty here: this
    // database has had nothing applied to it yet.
    let rows: [[Str]] = []

    // Read the migrations out of the directory, guessing the layout.
    match mgdir.read_auto("migrations")
        Err(f) => println(mgerror.describe(f))
        Ok(available) =>
            match mgstate.rows_to_applied(rows)
                Err(f) => println(mgerror.describe(f))
                Ok(applied) =>
                    // Work out what would run. Nothing runs: a refusal
                    // arrives here, before the first statement.
                    match mgplan.plan(available, applied, mgplan.default_options())
                        Err(f) => println(mgerror.describe(f))
                        Ok(p)  =>
                            // The same plan a real run would execute,
                            // one line per migration.
                            for line in mgplan.render(p)
                                println(line)
```

To apply the plan, walk `p.steps`, run each step's `sql` with your own
driver, and insert the row `mgstate.record_params` builds.

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: migrate.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `mgplan` | The migration, the plan, the version ordering, the planner for an up, a down and a redo, and the status report. |
| `mgstate` | The version table: the SQL that creates it, reads it, inserts into it and deletes from it, the parsing of the rows it answers, and the advisory-lock statements. |
| `mgdir` | The two directory conventions: parsing a file name, splitting a single file at its markers, reading a directory, and writing a new migration's files. |
| `mgerror` | Every reason a plan is refused, the sentence for each, and which of them need a person rather than a retry. |

## How to choose an entry point

**`mgplan.plan` is the ordinary call.** It answers what an upgrade
would run. `mgplan.plan_down` answers what a rollback to a given
version would run, in reverse order. `mgplan.plan_redo` undoes the
last applied migration and re-applies it, which is the development
loop and the one place a down is exercised often enough to be trusted.

**`mgplan.status` answers instead of refusing.** `plan` raises on a
changed checksum, because a run must not proceed. `status` reports the
same fact as an `MgModified` row, because it is the call a person
makes when something is already wrong.

**`mgdir.read_auto` guesses the directory's convention;
`mgdir.read` takes it as an argument.** Use `read_auto` on a directory
somebody else created. Use `read` when your project has decided.

**`mgstate` is for a caller with its own driver.** `create_table_sql`,
`select_sql`, `insert_sql` and `delete_sql` produce the statements;
`record_params` produces the values; `rows_to_applied` parses what
comes back. A caller whose driver takes numbered placeholders passes
`placeholder_for("postgres", n)` rather than building the SQL itself.

## The rules a user needs

1. **Nothing here runs a statement.** `plan` answers a value and the
   caller's driver executes it. The reason is a language rule: a
   library that called a database through a `dyn` trait would be
   charged the union of the effects of every driver in the consuming
   program (SPEC section 5.6), so its declared effects would change
   when the program added an engine. An effect parameter on the
   database trait is the change that would allow a runner here, and it
   has not landed.
2. **Compare versions with `mgplan.compare_versions`, never as text.**
   A version that is all digits compares numerically, so `"10"` comes
   after `"9"`. Text order puts `10` first and runs the migrations in
   the wrong order on exactly the tenth one, which is late enough that
   every test suite has passed.
3. **A version is a `Str`, and the column is `TEXT`.** The sequence
   convention writes `0007`, and an integer column silently drops the
   leading zeros, after which the file name no longer matches its row.
4. **The checksum is over the up alone.** Editing a down that has
   never run changes nothing that has happened. Refusing for it would
   make fixing a broken rollback impossible without rewriting the
   applied history.
5. **Every refusal arrives before the first statement.** A run that
   discovered a problem halfway has left half of version 7 applied and
   version 7 unrecorded, and nothing from the outside can tell which
   half.
6. **A file that does not parse stops the read.** It is not skipped. A
   migration silently skipped is a migration that runs on one machine
   and not on another. `MgBadFileName` names the file and what was
   wrong with it.
7. **The read is sorted by version, not by the directory.** A
   filesystem answers names in whatever order it likes, and even a
   sorted one puts `10` before `9`. `rows_to_applied` re-sorts the
   database's answer for the same reason.
8. **A migration below the highest applied version is refused by
   default.** Two people branched, each added a migration, and one
   merged second. `MgPlanOptions.allow_out_of_order` turns the refusal
   off. Leaving it off is what keeps the schema's final shape from
   depending on the order two people happened to merge in.
9. **The applied time is the caller's string.** This package has no
   clock and no calendar. A caller that wants the column to sort
   correctly passes an ISO-8601 string, which is why ISO-8601 is
   written the way it is.
10. **The table name is checked, not quoted.**
    `mgstate.is_safe_table_name` accepts letters, digits and
    underscores, not starting with a digit, up to 63 characters.
    Anything else is refused, because quoting an identifier is
    dialect-specific.
11. **Every value goes into a statement as a parameter.** The name
    comes from a file name and the checksum from a hash, and neither
    is concatenated into SQL. The placeholder is the driver's: SQLite
    writes `?` and PostgreSQL writes `$1`, so `insert_sql` takes it as
    an argument.
12. **There is no portable lock, and `mgstate.lock_sql` answers empty
    when there is none.** Three copies of a service starting at once
    all read an empty version table and all run migration 1.
    PostgreSQL has `pg_advisory_lock` and MySQL has `GET_LOCK`;
    SQLite's own file lock already does it. A caller that gets an
    empty statement back knows it has to arrange exclusion itself.
    `mgstate.lock_key_for` derives the key from the table name, so two
    applications sharing a database do not exclude each other.
13. **A rollback through an irreversible migration does not start.**
    `MgMigration.down` is allowed to be empty, and
    `mgerror.MgRollbackBlocked` is what a rollback past one answers.
14. **One step is one migration, not one statement.** Splitting SQL
    into statements needs a parser, and this package does not have
    one. A driver that cannot run several statements in one call has a
    caller who will have to split them, and that caller knows the
    dialect.

Five refusals stop a plan, and each of them needs a person.

| Refusal | What happened |
| --- | --- |
| `MgChecksumChanged` | An applied migration's text has been edited. Carries both digests, so a reader can tell an edit from a line-ending conversion |
| `MgDuplicateVersion` | Two migrations claim one version. Whichever ran first would be recorded and the other would never run |
| `MgOutOfOrder` | A migration arrived below the highest applied version. See rule 8 |
| `MgAppliedMissing` | The version table names a migration the directory does not have |
| `MgRollbackBlocked` | A rollback would pass through an irreversible migration |

## What is not included

- **Running anything.** See rule 1. When the database trait takes an
  effect parameter, a runner is added on top of `plan`, and `plan`
  does not change.
- **Parsing SQL.** A migration's body is text the driver runs. A
  parser here would refuse valid statements for whichever dialect it
  did not implement, and the driver is going to parse the text anyway.
- **A clock.** The applied time is an argument. See rule 9.
- **Printing.** Every report is a list of strings, and the program
  decides where they go.
- **Generating a migration from a schema difference.** Alembic's
  autogenerate introspects a live database and compares it with a
  model definition. Both halves are packages that do not exist yet.
- **Transactional migrations.** Whether a migration runs inside a
  transaction is the driver's decision and the dialect's: PostgreSQL
  can run DDL in one and MySQL cannot. The caller decides, and
  `MgApplied.succeeded` records a migration that failed partway when
  it could not.
- **Repeatable migrations and seed data.** Flyway's `R__` prefix
  re-runs a file whenever its checksum changes, which is the opposite
  of rule 4.
- **A semantic version for the schema.** A schema version has no
  pre-release, no build metadata and no compatibility rule. Modelling
  it as a semantic version invites the question of whether migration
  `2.0.0` is a breaking change, which has no meaning.

## Related packages

- [postgres-nv](https://novo-lang.org/packages/postgres-nv),
  [mysql-nv](https://novo-lang.org/packages/mysql-nv) and
  [sqlite-nv](https://novo-lang.org/packages/sqlite-nv) are the
  drivers that run what this package plans. Each one's parameter
  placeholder is what `mgstate.placeholder_for` names.
- [query-builder-nv](https://novo-lang.org/packages/query-builder-nv)
  builds SELECT, INSERT, UPDATE and DELETE as values, for the queries
  an application runs. This package builds the four statements the
  version table needs and nothing else.
- [crypto-nv](https://novo-lang.org/packages/crypto-nv) is the
  SHA-256 behind every checksum here, and behind the advisory-lock
  key.
- `std.sql` in the standard library opens an SQLite database and runs
  statements against it. It is the driver a first program uses with
  this package.

## Tests

```bash
novo test tests/mgplan_tests.nv   # the planner, the ordering and the refusals
novo test tests/mghost_tests.nv   # the directory conventions and the version table
```

The reference implementations are sqlx's migrator for the file
conventions, the checksum rule and the table's shape, and alembic for
the up, down, status and stamp vocabulary. The single-file markers
`-- +migrate Up` and `-- +migrate Down` are sql-migrate's and goose's
spelling.

No test opens a database. A suite builds a directory's worth of
migrations as values, declares some of them applied, and asserts on
what would run. The suite checks that `"10"` plans after `"9"`, that
an edited migration is refused rather than re-run, that two migrations
claiming one version are refused, that a migration below the highest
applied is refused unless the option is set, that a rollback through
an irreversible migration does not start, and that a file name that
does not parse stops the read instead of being skipped.

The tests compile today and fail at run, each on the
`not implemented: migrate.<module>.<fn>` panic that is its body. That
is the expected state of an interface release. They turn green one at
a time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `mgplan.default_options`, `.compare_versions`, `.sorted` | no |
| `mgplan.migration`, `.checksum_of`, `.is_reversible` | no |
| `mgplan.plan`, `.plan_down`, `.plan_redo`, `.is_empty`, `.render` | no |
| `mgplan.status`, `.current_version`, `.is_up_to_date`, `.render_status` | no |
| `mgstate.default_table`, `.is_safe_table_name`, `.column_names` | no |
| `mgstate.create_table_sql`, `.select_sql`, `.insert_sql`, `.delete_sql` | no |
| `mgstate.record_params`, `.placeholder_for`, `.rows_to_applied`, `.applied_of_step` | no |
| `mgstate.lock_sql`, `.unlock_sql`, `.lock_key_for` | no |
| `mgdir.up_marker`, `.down_marker`, `.parse_name`, `.file_name_for`, `.split_markers` | no |
| `mgdir.detect`, `.read`, `.read_auto`, `.create`, `.next_sequence` | no |
| `mgerror.describe`, `.version_of`, `.needs_a_person` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
