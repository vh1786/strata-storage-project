# Strata: one storage layer for Ada services, SQLite at the edge, PostgreSQL in the data centre

*Project document, 21 October 2026. "Strata" is a working name, to be
checked before publication.*

> **Status of every claim in this document.**
> - **Measured** (21 October 2026): the Alire index and each crate's
>   version, origin and licence (`alr show`); PostgreSQL **not** installed
>   on the measuring machine.
> - **[to verify]**: every statement about what GNATCOLL.SQL and ADO
>   provide beyond their names (M1 reads both code bases); SQL dialect
>   details of SQLite and PostgreSQL; libpq's multi-host connection
>   options; the high-availability managers named. Nothing is built on
>   such a claim before M1 settles it.

---

## 1. Summary

Many Ada services start on **SQLite**: one file, no server, perfect on
an appliance or a single machine. They grow into deployments that need
**PostgreSQL**: several writers, a standby, backups by the database team,
an operator who wants SQL access. Moving between the two is where projects
lose months. Two dialects, two concurrency models, two sets of types, a
data migration and a test suite to run twice.

**A deployment runs one engine, never both at once.** The engine is a
property of the deployment, typically of the edition: the smaller
editions on SQLite, the enterprise edition on PostgreSQL, mandatory there. The
application is written once and built or configured for one engine; the
only moment both are involved is the **upgrade** of a deployment from the
SQLite edition to a PostgreSQL edition, which the data mover (§5.5)
carries out once.

Strata is a thin, typed layer **on top of an existing Ada database
library** (GNATCOLL.SQL or ADO, chosen in M1). It gives an application:

1. **one schema description** from which the SQL for both engines and the
   Ada accessors are generated;
2. **one query API**, typed, where the dialect differences are the
   library's problem and not the application's;
3. **migrations**: versioned schema changes, applied the same way on both
   engines, refused when they would lose data unannounced;
4. **a data mover** for the edition upgrade: SQLite to PostgreSQL (and
   back, for tests only), run once, verified row by row;
5. **connections that know about high availability**: several hosts, the
   writable one found, a failover survived with a retry policy the
   application declares;
6. **safety rules enforced by the API**: no SQL text built from strings, no
   statement stepped while its own table is written, explicit
   transactions;
7. **a conformance suite** that runs every test against both engines.

---

## 2. The problem

### 2.1 Why one engine is not enough

| need | SQLite | PostgreSQL |
|---|---|---|
| zero administration, one file | yes | no |
| an appliance with no network dependency | yes | no |
| several concurrent writers | one writer at a time **[to verify: WAL mode allows readers during a write]** | yes, MVCC |
| a hot standby and failover | no | yes, with streaming replication and a manager |
| access by other tools (BI, a DBA) | awkward | natural |

A product sold to small sites and to enterprises needs both, one per
deployment, and should not be written twice. The engine is chosen by the
edition; a customer who moves up an edition moves their data once.

### 2.2 Where a migration from SQLite to PostgreSQL goes wrong

- **Dialect**: auto-increment keys, `INSERT ... ON CONFLICT`, `RETURNING`,
  type affinity against strict types, date and time storage, boolean
  representation, `LIKE` and case, full-text search (FTS5 against
  `tsvector`) **[each to verify against both engines' documentation]**.
- **Concurrency**: code that relied on SQLite's single writer (a
  read-modify-write without a transaction) races on PostgreSQL.
- **Hidden shapes**: a SELECT still being stepped while the same table is
  updated works on one engine and misbehaves or deadlocks on the other.
- **Data**: SQLite accepts a text in an integer column; PostgreSQL
  refuses it at the migration, a year of data later.
- **Tests**: a suite that ran on SQLite says nothing about PostgreSQL.

### 2.3 What already exists (measured in the Alire index)

| crate | version | licence | what it is |
|---|---|---|---|
| `gnatcoll_sql` | 26.0.0 | GPL-3.0-or-later WITH GCC-exception-3.1 | AdaCore's SQL layer: query construction in Ada, sessions **[features to verify]** |
| `gnatcoll_postgres` | 26.0.0 | same | its PostgreSQL backend (over libpq) |
| `gnatcoll_sqlite` | 26.0.0 | same | its SQLite backend |
| `ado` | 2.5.0 | Apache-2.0 | Ada Database Objects (core): an object-relational layer **[features to verify]** |
| `ado_postgresql` | 2.5.0 | Apache-2.0 | ADO's PostgreSQL driver |
| `ado_sqlite` | 2.5.0 | Apache-2.0 | ADO's SQLite driver |
| `ada_sqlite3` | 0.1.1 | MIT OR Apache-2.0 WITH LLVM-exception | a SQLite binding with the engine embedded |
| `libpq`, `libsqlite3` | — | — | the C client libraries, as system crates |

**The drivers exist; the layer an application needs above them does not.**
Neither project, as listed, offers a dual-engine migration tool, a data
mover, HA-aware connections or a conformance suite **[to confirm in M1:
if one already does, Strata adopts it rather than rewriting]**.

---

## 3. Goals and non-goals

### Goals

1. **Build on one existing library**: no new driver, no new SQL builder
   if an adequate one exists.
2. **One schema, two engines**: a schema description (in Ada or a small
   declarative file) generates the DDL for each engine and typed Ada
   accessors.
3. **Typed queries only** in application code; raw SQL confined to
   reviewed, named statements.
4. **Migrations**: ordered, versioned, checksummed, applied in a
   transaction where the engine allows it; a destructive change (a dropped
   column, a narrowed type) refused unless the migration declares it.
5. **The data mover**: copy every table from SQLite to PostgreSQL, in
   batches, with types checked, counts and checksums compared, resumable,
   and a report.
6. **HA-aware connections** for PostgreSQL: several hosts, the writable
   one chosen, a lost connection or a failover handled by a declared
   policy (retry the transaction, or fail it).
7. **The conformance suite**: every Strata test, and every application
   test written against Strata, runs on both engines in CI.
8. **Safety rules enforced** (§6).

### Non-goals

- A new SQL driver for either engine.
- Other engines (MySQL, Oracle) in the first release. The design leaves
  room for them.
- An ORM with lazy loading and identity maps. Strata stays close to SQL.
- Running the database or its HA (that is the operations team's, and of
  tools such as Patroni or pg_auto_failover **[to verify]**). Strata only
  connects well.

---

## 4. Choosing the base: GNATCOLL.SQL or ADO

M1 reads both and decides on these criteria:

| criterion | why it matters |
|---|---|
| **licence** | GNATCOLL is GPL-3.0-or-later with the GCC runtime exception, which allows linking into a proprietary program compiled with GCC **[to verify the exception's conditions for this use]**; ADO is Apache-2.0. Strata's own licence follows from the base. |
| typed query construction | queries checked by the Ada compiler: column types, joins |
| code generation from a schema | GNATCOLL has a schema-to-Ada generator **[to verify: `gnatcoll_db2ada`; no crate of that name is listed]**; ADO has its own generator **[to verify]** |
| both engines behind one session type | the dual-engine promise rests on it |
| prepared statements and parameter binding | no SQL text built from values |
| transactions and savepoints | migrations and the retry policy need them |
| connection pooling | services with several tasks |
| maintenance | release cadence, open issues, who maintains |
| tasking safety | which objects may be shared across tasks |

The decision is written in `docs/decisions.md` with the reading behind
it. **The recommendation is not made here**: it would rest on features
not yet read.

---

## 5. Design

### 5.1 The schema

```ada
--  schema/devices.ads -- a declarative description (form to fix in M2)
Table ("devices",
  (Column ("id",         Big_Serial,  Primary_Key),
   Column ("mac",        Text (17),   Unique, Not_Null),
   Column ("first_seen", Timestamp_UTC, Not_Null),
   Column ("vendor",     Text (128))),
  Indexes => (Index ("devices_by_vendor", ("vendor"))));
```

From it, Strata generates:

- the DDL for SQLite and for PostgreSQL, each in its dialect;
- typed Ada accessors (records, query helpers) through the base library's
  generator, or its own if the base has none;
- a schema fingerprint (a hash of the description) stored in the
  database, so that a program refuses to run against a schema it was not
  built for.

### 5.2 Types across engines

| Strata type | SQLite | PostgreSQL | rule |
|---|---|---|---|
| `Big_Serial` | `INTEGER PRIMARY KEY` | `BIGINT GENERATED ... AS IDENTITY` **[to verify]** | keys never reused |
| `Text (N)` | `TEXT` with a `CHECK (length(x) <= N)` | `VARCHAR(N)` | the limit enforced on both |
| `Timestamp_UTC` | `INTEGER` (Unix microseconds) or ISO-8601 text, one chosen | `TIMESTAMPTZ` | UTC always; the mover converts |
| `Boolean` | `INTEGER CHECK (x IN (0,1))` | `BOOLEAN` | |
| `Bytes` | `BLOB` | `BYTEA` | |
| `Decimal (P, S)` | `TEXT` | `NUMERIC(P,S)` | no float for money or counters |

SQLite tables are created `STRICT` where the SQLite version allows it
**[to verify the version]**, so that SQLite refuses a wrong type as
PostgreSQL would: the data mover then never meets a bad row.

### 5.3 Queries

Application code uses the base library's typed builder. Strata adds:

- **dialect switches** for the few statements that differ (upsert,
  returning the generated key), behind one Ada call;
- **named raw statements**: when SQL text is unavoidable, it lives in one
  package per engine, under a name, reviewed, with its parameters bound.

### 5.4 Migrations

```
migrations/
  0001_create_devices.strata
  0002_add_vendor_index.strata
  0003_split_first_seen.strata     -- declares: Destructive => False
```

- Each migration is applied once; its checksum is stored; a changed
  migration file is refused.
- Applied inside a transaction on PostgreSQL, and on SQLite where the
  statement allows it **[to verify which DDL SQLite can run in a
  transaction]**.
- **A destructive change** (drop, narrowing, a NOT NULL without a
  default on a non-empty table) is refused unless the migration declares
  it and provides the data step.
- `strata migrate --plan` prints what would run, on which engine, before
  anything runs.

### 5.5 The data mover

```
strata move --from sqlite:/var/lib/app/app.db --to postgres://... --batch 10000
```

1. Checks that both schemas carry the same fingerprint.
2. Copies table by table in dependency order (foreign keys), in batches,
   inside transactions, resumable from the last committed batch.
3. Converts the types of §5.2.
4. **Verifies**: row counts per table, and a checksum per table computed
   the same way on both sides.
5. Reports, and refuses to call the move done until every table
   verifies.

### 5.6 HA-aware connections (PostgreSQL)

- A connection descriptor lists several hosts; the writable primary is
  chosen at connect time. libpq supports several hosts and a
  `target_session_attrs` choice **[to verify the option names and values
  in libpq's documentation]**.
- **A failover policy declared per transaction**: `Retry (Times => 3)` for
  idempotent work, `Fail` for the rest. A transaction lost in a failover
  is never silently replayed if it was not declared idempotent.
- Read-only queries may go to a standby when the application allows it.

### 5.7 Concurrency

- Each engine's model is stated in the API: on SQLite, one writer at a
  time (Strata serialises writers inside the process and sets a busy
  timeout); on PostgreSQL, a pool of connections.
- `Transaction` is a controlled object: commit is explicit, and leaving
  its scope without a commit rolls it back.

---

## 6. The safety rules, enforced

Each rule is held by the API's shape or by a check, never by a sentence
in the documentation alone:

| rule | how it is enforced |
|---|---|
| no SQL text built from values | the API takes parameters, not strings; a lint refuses `&` on SQL text in application code |
| no statement stepped while its own table is written in the same connection | a cursor records its tables; a write to one of them while the cursor is open raises a refusal (a result, not an exception) |
| every write inside a transaction | write calls take a `Transaction` parameter |
| a schema mismatch stops the program at start | the fingerprint check (§5.1) |
| no unbounded result set loaded into memory | iteration by cursor; `Fetch_All` takes a maximum |
| time in UTC | the only timestamp type is UTC |

---

## 7. Testing

- **The conformance suite**: every test runs twice, on SQLite (a file in
  a temporary directory) and on PostgreSQL (a throwaway server in a
  container). CI fails if either fails.
- **Dialect tests** per switch of §5.3: the same Ada call, both engines,
  the same result.
- **Migration tests**: every migration applied from empty and from the
  previous release's real data (anonymised), on both engines.
- **Mover tests**: round trip SQLite → PostgreSQL → SQLite, checksums
  equal; interrupted moves resumed.
- **Failover tests**: a primary killed during a transaction, with a
  standby promoted by the HA manager in the test environment; the declared
  policy observed.
- **Scale tests**: a million rows per table, timing published; the shapes
  that are fine on ten rows and quadratic on a million are found here.

---

## 8. Roadmap

| milestone | content | done when |
|---|---|---|
| **M0** reading | SQLite's and PostgreSQL's documentation for every [to verify] in §2.2, §5.2, §5.4, §5.6; libpq's connection options | each answered and the document amended |
| **M1** the base | GNATCOLL.SQL and ADO read on §4's criteria; a spike: the same small schema and three queries on both engines with each library | the choice recorded with its reasons; if one already offers part of Strata, the scope shrinks |
| **M2** schema and types | the schema description, the DDL generators, the type table of §5.2, the fingerprint | both engines created from one description; the conformance suite's first tests green on both |
| **M3** queries and safety | dialect switches, named statements, the rules of §6 | each rule has a test that shows it refusing |
| **M4** migrations | §5.4 | migrations applied on both engines; a destructive one refused without its declaration |
| **M5** the mover | §5.5 | a million-row round trip verified; an interrupted move resumed |
| **M6** HA connections | §5.6 | the failover tests pass with a real standby |
| **M7** publication | crate, documentation, an example service | a stranger runs the conformance suite on both engines |

---

## 9. Risks and open questions

| risk | answer |
|---|---|
| The base library's licence constrains users | §4's first criterion; Apache-2.0 if ADO is chosen; the GCC exception read carefully if GNATCOLL is |
| A dialect difference cannot be hidden | expose it as an engine-specific named statement, documented |
| SQLite `STRICT` not available on old systems | the version checked at start; a CHECK-based fallback |
| Scope creep into an ORM | §3's non-goals |
| HA behaviour depends on the manager | Strata connects and retries; failover itself is the manager's, and the tests use a real one |

Open questions for the maintainers:

1. GNATCOLL.SQL or ADO: settled by M1's reading, not before.
2. The schema description: an Ada package (checked by the compiler) or a
   small file format (readable by non-Ada tools)?
3. Strata's own licence: it follows the base (Apache-2.0 with ADO; with
   GNATCOLL, GPL-compatible with the same exception, or permissive if the
   exception allows it).

---

## 10. Repository layout

```
strata/
  alire.toml                     -- depends on the chosen base and its two drivers
  strata.gpr
  src/
    strata.ads                   -- results, errors
    strata-schema.ad[sb]         -- description, fingerprint
    strata-ddl-sqlite.ad[sb]  strata-ddl-postgres.ad[sb]
    strata-types.ad[sb]          -- §5.2's mapping
    strata-query.ad[sb]          -- dialect switches over the base builder
    strata-statements.ad[sb]     -- named raw statements, per engine
    strata-transactions.ad[sb]   -- controlled transactions, failover policy
    strata-connections.ad[sb]    -- HA-aware descriptors, pooling
    strata-migrate.ad[sb]
    strata-move.ad[sb]
    strata-safety.ad[sb]         -- cursor/table tracking, limits
  tools/
    strata_cli.adb               -- migrate, move, plan, check
  tests/
    conformance/                 -- run on both engines
    migrations/  mover/  failover/  scale/
  examples/
    inventory-service/
  docs/
    decisions.md  schema.md  types.md  migrations.md  mover.md  ha.md  safety.md
```

---

## Appendix A — measurements behind this document

21 October 2026:

```bash
alr show gnatcoll_sql gnatcoll_postgres gnatcoll_sqlite
#   26.0.0, GPL-3.0-or-later WITH GCC-exception-3.1, from AdaCore/gnatcoll-db v26.0.0
alr show ado ado_postgresql ado_sqlite
#   2.5.0, Apache-2.0, from gitlab.com/stcarrez/ada-ado
alr show ada_sqlite3       # 0.1.1, MIT OR Apache-2.0 WITH LLVM-exception
command -v psql postgres   # nothing: PostgreSQL not installed here
```
