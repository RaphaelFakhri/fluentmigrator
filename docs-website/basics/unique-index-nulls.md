# NULL handling in unique indexes

Database engines disagree on whether two NULLs conflict in a unique index. SQL Server and Db2 treat NULLs as equal, so a unique index allows at most one row with NULL in the indexed columns. PostgreSQL, SQLite, MySQL and Firebird treat NULLs as distinct, so any number of such rows is allowed. A migration that relies on the engine default therefore means different things on different engines.

FluentMigrator provides two provider-agnostic options that state the intent instead. Each provider maps the intent to the native syntax that expresses it, or reports that it cannot.

```csharp
Create.Index("UX_Prices_Product_Tier")
    .OnTable("Prices")
    .OnColumn("ProductId").Ascending()
    .OnColumn("TierPriceId").Ascending()
    .WithOptions().UniqueTreatNullsAsEqual();
```

| Method | Meaning |
|---|---|
| `UniqueTreatNullsAsDistinct()` | Index values must be unique. Rows with NULL in an indexed column never conflict, so any number of them may exist. |
| `UniqueTreatNullsAsEqual()` | Index values must be unique. NULLs compare equal, so at most one row may hold a given combination that includes NULL. |

Both methods also mark the index as unique, so you don't need to call `Unique()` as well.

## Semantics per provider

The table describes the SQL that each generator emits for the two options.

| Provider | Native behavior | `UniqueTreatNullsAsDistinct()` | `UniqueTreatNullsAsEqual()` |
|---|---|---|---|
| SQL Server 2008 and later | Equal | Filtered index: `WHERE [c1] IS NOT NULL AND [c2] IS NOT NULL` | No change (native) |
| SQL Server 2005 | Equal | Compatibility mode | No change (native) |
| SQL Server 2000 | Equal | Compatibility mode | Compatibility mode |
| PostgreSQL 15 and later | Distinct | No change (native) | `NULLS NOT DISTINCT` |
| PostgreSQL 14 and earlier | Distinct | No change (native) | Compatibility mode |
| SQLite | Distinct | No change (native) | Compatibility mode |
| MySQL 4, 5 and 8 | Distinct | No change (native) | Compatibility mode |
| Firebird | Distinct | No change (native) | Compatibility mode |
| Db2 and Db2 for i | Equal | Compatibility mode | No change (native) |
| Oracle and Oracle 12c | Mixed (see [Oracle](#oracle)) | Compatibility mode | Compatibility mode |
| SAP HANA | Not declared | Compatibility mode | Compatibility mode |
| Redshift and Snowflake | Indexes not supported | Compatibility mode (the whole index) | Compatibility mode (the whole index) |

In the table:

- **No change (native)** means the engine already behaves as requested, so the generator emits the same `CREATE UNIQUE INDEX` statement as `Unique()`.
- **Compatibility mode** means the generator can't express the request. The generator's compatibility mode decides what happens:
  - `CompatibilityMode.STRICT` throws a `DatabaseOperationNotSupportedException`.
  - `CompatibilityMode.LOOSE`, the default, ignores the request and emits a plain unique index with the engine's native behavior.

To make unsupported requests fail instead of silently falling back to the engine default, set the compatibility mode to `STRICT`:

```csharp
services.Configure<GeneratorOptions>(opt => opt.CompatibilityMode = CompatibilityMode.STRICT);
```

## Provider notes

### SQL Server

SQL Server has no clause that makes NULLs distinct in a unique index. On SQL Server 2008 and later, `UniqueTreatNullsAsDistinct()` emits a filtered index that excludes rows where any indexed column is NULL. Filtered indexes don't exist in SQL Server 2005 and earlier.

### PostgreSQL

PostgreSQL 15 added `NULLS NOT DISTINCT`, which `UniqueTreatNullsAsEqual()` emits. Earlier versions have no equivalent, and a partial index can't express it.

### Db2

A Db2 unique index treats NULLs as equal, so `UniqueTreatNullsAsEqual()` needs no extra syntax. The Db2 generator doesn't emit any NULL-related index clause, so `UniqueTreatNullsAsDistinct()` goes through compatibility mode. Db2 for i uses the same generator and behaves the same way.

Db2 for Linux, UNIX and Windows has `EXCLUDE NULL KEYS`, but that clause only skips keys in which every column is NULL. A composite key with only some NULL columns still compares equal, so the clause doesn't provide distinct semantics for multi-column indexes.

### Oracle

Oracle doesn't index a key in which every column is NULL, so any number of such rows may exist. A composite key with only some NULL columns is indexed, and those NULLs compare equal. Neither option describes this behavior for every index, so both go through compatibility mode.

## Provider-specific options

The provider-specific options in `FluentMigrator.SqlServer` and `FluentMigrator.Postgres` are unchanged. When you set both a provider-specific option and one of these options on the same index, the provider-specific option takes precedence.

## See also

- [Indexes](./indexes.md)
- [SQL Server provider](../providers/sql-server.md)
- [PostgreSQL provider](../providers/postgresql.md)
