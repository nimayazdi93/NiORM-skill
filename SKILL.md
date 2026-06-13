---
name: niorm-skill
description: Build and maintain .NET apps with NiORM — attribute-mapped SQL Server and MongoDB data access, CRUD, LINQ-style Where, raw SQL, logging, and security patterns. Use when working with NiORM, NiORM NuGet package, DataCore, IEntities, SQL Server ORM mapping, or MongoDB via NiORM.
license: MIT
compatibility: Requires .NET 6+, NiORM NuGet package, SQL Server or MongoDB connection string
metadata:
  author: NiORM
  version: "1.0.0"
  package: NiORM
  package-version: "1.6.0-beta.2"
---

# NiORM

NiORM is a lightweight .NET ORM (SQL Server + MongoDB) using attribute-based mapping and convention-over-configuration. Load only the references needed for the current task.

## Workflow

1. Confirm target: **SQL Server** (`NiORM.SQLServer`) or **MongoDB** (`NiORM.Mongo`).
2. Install: `dotnet add package NiORM`
3. Define entity models with required attributes.
4. Create a `DataCore` subclass exposing `IEntities<T>` properties.
5. Use CRUD and query APIs; prefer safe methods over raw SQL.
6. For full API details, see [references/api-reference.md](references/api-reference.md).
7. For copy-paste patterns, see [references/examples.md](references/examples.md).

## Quick Start (SQL Server)

### 1. Entity model

Every SQL Server entity **must** implement `ITable` and use `[TableName]`:

```csharp
using NiORM.Attributes;
using NiORM.SQLServer.Interfaces;

[TableName("People")]
public class Person : ITable
{
    [PrimaryKey(isAutoIncremental: true)]
    public int Id { get; set; }
    public string Name { get; set; }
    public int Age { get; set; }
    public Marriage? Marriage { get; set; } // enums stored as int
}

public enum Marriage { Single, Married }
```

Optional interfaces:
- `IUpdatable` — auto-sets `CreatedDateTime` / `UpdatedDateTime` on Add/Edit
- `IView` — read-only; Add/Edit throw `NiORMValidationException`
- `IReportable` — marker for report entities

Primary key rules:
- Single PK: use `Find(int id)` or `Find(string id)`
- Composite PK (exactly 2): use `Find(int firstId, string secondId)`
- Auto-increment int PK: omitted on insert
- GUID PK: `[PrimaryKey(isAutoIncremental: true, isGUID: true)]` — auto-generated on insert

### 2. Data service

```csharp
using NiORM.SQLServer;
using NiORM.SQLServer.Interfaces;

public class DataService : DataCore
{
    public DataService(string connectionString) : base(connectionString) { }

    private IEntities<Person>? _people;
    public IEntities<Person> People =>
        _people ??= CreateEntity<Person>();
}
```

Always pass a non-empty connection string. The parameterless `DataCore()` constructor is **obsolete** — do not use.

### 3. CRUD

```csharp
var db = new DataService(connectionString);

// Read
var all = db.People.ToList();
var one = db.People.Find(1);
var first = db.People.FirstOrDefault(p => p.Name == "Nima");

// Create
db.People.Add(new Person { Name = "Nima", Age = 29 });
var inserted = db.People.AddReturn(new Person { Name = "Ali", Age = 25 });

// Update
one!.Age = 30;
db.People.Edit(one);

// Delete
db.People.Remove(one);
db.People.Remove(p => p.Age == 18); // only ==, !=, &&, || in expression predicates
```

## Querying

### LINQ Where (preferred for filters)

```csharp
var results = db.People
    .Where(p => p.Name == "Nima" && p.Age == 30)
    .ToList();

// DateTime (v1.6.0-beta+)
var today = db.People
    .Where(p => p.CreatedAt.Value.Date == DateTime.Now.Date)
    .ToList();
```

**Supported expression operators:** `==`, `!=`, `&&`, `||`, member access, constants, `DateTime.Now`, `DateTime?.Value.Date`.

**Not supported in Where expressions:** `>`, `<`, `>=`, `<=`, method calls, string methods. For those, use parameterized raw SQL via `SqlParameterHelper` or static WHERE with trusted literals only.

### Raw SQL

```csharp
// On DataCore — maps to any type (no ITable required)
var cats = db.SqlRaw<Cat>("SELECT * FROM Cats");
var names = db.SqlRaw<string>("SELECT [Name] FROM Cats");

// On entity — returns ITable-mapped entities
var custom = db.People.Query("SELECT * FROM People WHERE Age > 18");
```

For user-supplied values in raw SQL, **always** use `SqlParameterHelper`:

```csharp
using NiORM.SQLServer.Core;

var helper = new SqlParameterHelper();
var nameParam = helper.AddParameter(userInput);
var safe = db.SqlRaw<Person>($"SELECT * FROM People WHERE Name = {nameParam}");
```

## Security Rules

| Safe (parameterized) | Use with caution (logs warning) |
|---|---|
| `Add`, `AddReturn`, `Edit`, `Remove` | `Query`, `Execute` |
| `Find(id)` | `List(whereClause)`, `ToList(whereClause)` |
| `Where(Expression)` for filters | `FirstOrDefault(string whereClause)` |
| `SqlRaw` + `SqlParameterHelper` | `SqlRaw` with string concatenation |

**Do not use** `FindByProperty()` or `WhereMultiple()` — they appear in older README/docs but are **not implemented** in v1.6.0-beta.2. Use `Where(p => p.Property == value)` instead.

Never concatenate user input into SQL strings.

## Logging & Errors

```csharp
using NiORM.SQLServer.Core;

NiORMLogger.IsEnabled = true;
NiORMLogger.MinimumLogLevel = LogLevel.Info;
NiORMLogger.LogFilePath = @"C:\logs\niorm.log"; // optional; default is console
```

Catch hierarchy:
- `NiORMValidationException` — mapping/validation (e.g. editing a view)
- `NiORMConnectionException` — connection failures
- `NiORMException` — general DB errors; check `ex.SqlQuery` and `ex.OperationType`

## MongoDB (brief)

Namespace: `NiORM.Mongo`. Requires connection string **and** database name.

```csharp
using NiORM.Attributes;
using NiORM.Mongo;
using NiORM.Mongo.Core;

[CollectionName("Products")]
public class Product : MongoCollection
{
    public string Title { get; set; }
}

public class MongoService : DataCore
{
    public MongoService(string conn, string db) : base(conn, db) { }
    public Entities<Product> Products => CreateEntity<Product>();
}
```

Mongo API: `Add`, `Edit`, `Remove`, `Remove(id)`, `List`, `Get(query)`, `Find(id)`.

## Decision Checklist

When implementing NiORM code, verify:

- [ ] Entity has `[TableName]` and implements `ITable` (SQL Server)
- [ ] Primary key properties marked with `[PrimaryKey]`
- [ ] `DataService` inherits `NiORM.SQLServer.DataCore` with valid connection string
- [ ] User input filtered via `Where()` expressions or parameterized SQL
- [ ] No reliance on undocumented methods (`FindByProperty`, `WhereMultiple`)
- [ ] Views use `IView`; timestamped entities use `IUpdatable`
- [ ] Exceptions handled with appropriate `NiORM*` types

## Additional Resources

- [API Reference](references/api-reference.md) — namespaces, interfaces, method signatures, LINQ limits
- [Examples](references/examples.md) — patterns from NiORM.Test and README
