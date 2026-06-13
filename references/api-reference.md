# NiORM API Reference

## Namespaces

| Namespace | Purpose |
|---|---|
| `NiORM.Attributes` | `[TableName]`, `[PrimaryKey]`, `[CollectionName]` |
| `NiORM.SQLServer` | `DataCore` — SQL Server entry point |
| `NiORM.SQLServer.Interfaces` | `ITable`, `IView`, `IUpdatable`, `IReportable`, `IEntities<T>`, `IWhereEntities<T>` |
| `NiORM.SQLServer.Core` | `NiORMLogger`, `NiORMException`, `SqlParameterHelper`, `SqlInjectionValidator` |
| `NiORM.Mongo` | MongoDB `DataCore` |
| `NiORM.Mongo.Core` | `MongoCollection`, `Entities<T>` |

## Attributes

### `[TableName("TableName")]` (required on SQL Server entities)

Maps class to SQL table. Without it, `ObjectDescriber` throws at runtime.

### `[PrimaryKey(isAutoIncremental: bool, isGUID: bool)]`

Place on PK property/field. Defaults: `isAutoIncremental: true`, `isGUID: false`.

| Configuration | Insert behavior |
|---|---|
| `isAutoIncremental: true`, int | Column excluded from INSERT; DB generates value |
| `isAutoIncremental: true`, `isGUID: true` | GUID auto-assigned before INSERT |
| `isAutoIncremental: false` | PK value must be set manually |

Multiple `[PrimaryKey]` properties = composite key.

### `[CollectionName("name")]` (required on MongoDB entities)

Maps class to MongoDB collection.

## SQL Server — DataCore

```csharp
public class DataCore : IDataCore
{
    public DataCore(string connectionString);
    [Obsolete] public DataCore();

    public IEntities<T> CreateEntity<T>() where T : ITable, new();
    public List<T> SqlRaw<T>(string query);
}
```

## SQL Server — IEntities&lt;T&gt;

All methods require `T : ITable`.

### Read

| Method | Description |
|---|---|
| `ToList()` | All rows |
| `ToList(string whereQuery)` | Filtered by raw WHERE clause (no `WHERE` keyword) |
| `List()` | Alias for `ToList()` |
| `List(string Query)` | Alias for `ToList(whereQuery)` |
| `Find(int id)` | Single PK lookup (parameterized) |
| `Find(string id)` | Single PK lookup (parameterized) |
| `Find(object id)` | Delegates to string |
| `Find(int firstId, string secondId)` | Composite PK (exactly 2 keys) |
| `FirstOrDefault()` | TOP 1 |
| `FirstOrDefault(string Query)` | TOP 1 with raw WHERE |
| `FirstOrDefault(Expression<Func<T, bool>> predicate)` | TOP 1 via LINQ |
| `Where(Expression<Func<T, bool>> predicate)` | Returns `IWhereEntities<T>` |
| `Query(string Query)` | Custom SELECT, mapped to T |
| `SqlRaw` on DataCore | Custom query, any return type |

### Write

| Method | Description |
|---|---|
| `Add(T entity)` | INSERT (parameterized) |
| `AddReturn(T entity)` | INSERT with `OUTPUT inserted.*`, returns entity |
| `Edit(T entity)` | UPDATE by PK (parameterized) |
| `Remove(T entity)` | DELETE by PK (parameterized) |
| `Remove(Expression<Func<T, bool>> predicate)` | DELETE with LINQ WHERE |
| `Set(Expression<Func<T, bool>> predicate, object Value, string? whereConditions)` | Bulk UPDATE |
| `Execute(string Query)` | Non-query SQL, returns rows affected |

### IWhereEntities&lt;T&gt;

Returned by `Where()`:

| Method | Description |
|---|---|
| `ToList()` | Execute filtered SELECT |
| `Set(Expression<Func<T, bool>> predicate, object value)` | UPDATE rows matching Where + Set |

## LINQ Expression Translation

`Where()` and `FirstOrDefault(predicate)` compile expressions to SQL WHERE fragments.

**Supported:**
- `==`, `!=`
- `&&`, `||`
- Property access: `p.Name`, `[ColumnName]`
- Constants (strings quoted as `N'...'`, dates formatted, bools as 0/1, enums as int)
- `DateTime.Now` → `GETDATE()`
- `nullableDateTime.Value.Date` or `.Date` on DateTime → `CAST(column AS DATE)`

**Not supported (throws `NotSupportedException`):**
- Comparison: `>`, `<`, `>=`, `<=`
- `Contains`, `StartsWith`, `EndsWith`
- Arithmetic, method calls, subqueries

**Workaround for range queries:** use `SqlParameterHelper` with `SqlRaw` or trusted static WHERE in `List()`.

## SqlParameterHelper

```csharp
var helper = new SqlParameterHelper();

// Anonymous params: @param1, @param2, ...
var p1 = helper.AddParameter("value");
var p2 = helper.AddParameter(42);

// Named params
var p3 = helper.AddNamedParameter("Name", userInput);

// Use in query string
var sql = $"SELECT * FROM People WHERE Name = {p1} AND Age = {p2}";
// Pass helper.Parameters to SqlMaster internally via DataCore.SqlRaw pattern
```

Also provides `BuildInsertValuesClause`, `BuildUpdateSetClause`, `BuildPrimaryKeyWhereClause` (used internally by CRUD).

## SqlInjectionValidator

Static utility for detecting injection patterns in raw SQL strings:

```csharp
var result = SqlInjectionValidator.ValidateSql(sql);
// result.IsValid, result.RiskLevel, result.Warnings
```

Use when accepting raw SQL from external sources.

## Exception Types

| Type | When |
|---|---|
| `NiORMException` | General errors; has `SqlQuery`, `OperationType` |
| `NiORMConnectionException` | Connection failures |
| `NiORMValidationException` | Invalid entity ops; has `Entity` |
| `NiORMMappingException` | Entity ↔ DB mapping errors |

## NiORMLogger

Static configuration:

```csharp
NiORMLogger.IsEnabled = true;
NiORMLogger.MinimumLogLevel = LogLevel.Warning; // Info, Warning, Error, Debug
NiORMLogger.LogFilePath = null; // null = console
```

## MongoDB — DataCore

```csharp
public class DataCore
{
    public DataCore(string connectionString, string databaseName);
    public Entities<T> CreateEntity<T>() where T : MongoCollection, new();
}
```

## MongoDB — Entities&lt;T&gt;

Base class: `MongoCollection` with `[BsonId] string ID`.

| Method | Returns |
|---|---|
| `Add(T entity)` | `bool` |
| `Edit(T item)` | `bool` (ReplaceOne by ID) |
| `Remove(T entity)` | `bool` |
| `Remove(string ID)` | `bool` |
| `List()` | `List<T>` |
| `Get(string Query)` | `List<T>` (MongoDB filter string) |
| `Find(string ID)` | `T` |

## Type Mapping Notes

- Property names must match SQL column names (case-sensitive in SQL Server brackets)
- Only writable properties are included in INSERT/UPDATE
- Enums stored and read as integers
- Nullable value types supported
- `SqlRaw<T>` can map to classes **without** `[TableName]` or `ITable` (property names must match result columns)

## Package Info

- Target: `net6.0`
- Dependencies: `System.Data.SqlClient` 4.8.6, `MongoDB.Driver` 2.19.1
- Install: `dotnet add package NiORM`
