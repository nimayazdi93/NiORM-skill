# NiORM Examples

Patterns derived from `NiORM.Test` and the project README.

## Project Structure

```
MyApp/
├── Models/
│   └── Person.cs          # ITable + attributes
├── Services/
│   └── DataService.cs     # extends DataCore
└── Program.cs             # usage
```

Reference project:

```
NiORM.Test/
├── Models/Person.cs, Cat.cs
├── Service/DataService.cs
└── Program.cs
```

## Complete SQL Server Example

### Person.cs

```csharp
using NiORM.Attributes;
using NiORM.SQLServer.Interfaces;

namespace MyApp.Models;

[TableName("People")]
public class Person : ITable
{
    [PrimaryKey(isAutoIncremental: true)]
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public int Age { get; set; }
    public Marriage? Marriage { get; set; }
}

public enum Marriage { Single, Married }
```

### DataService.cs

```csharp
using NiORM.SQLServer;
using NiORM.SQLServer.Interfaces;
using MyApp.Models;

namespace MyApp.Services;

public class DataService : DataCore
{
    public DataService(string connectionString) : base(connectionString) { }

    private IEntities<Person>? _people;
    public IEntities<Person> People =>
        _people ??= CreateEntity<Person>();
}
```

### Program.cs (from NiORM.Test)

```csharp
using MyApp.Models;
using MyApp.Services;

var conn = "Data Source=(localdb)\\MSSQLLocalDB;Initial Catalog=StoreDb;Integrated Security=True;";
var db = new DataService(conn);

// Fetch all
var people = db.People.ToList();

// Add
var person = new Person { Age = 29, Name = "Nima", Marriage = Marriage.Single };
db.People.Add(person);

// Find by PK
person = db.People.Find(1)!;

// LINQ filter
people = db.People.Where(c => c.Name == "Nima" && c.Age == 30).ToList();

// Raw SQL (any type — Cat has no ITable)
var cats = db.SqlRaw<Cat>("SELECT * FROM Cats");
var names = db.SqlRaw<string>("SELECT [Name] FROM Cats");

// Edit
person.Age = 30;
db.People.Edit(person);

// Remove
db.People.Remove(person);
```

### Cat.cs (raw SQL mapping only)

```csharp
namespace MyApp.Models;

public class Cat
{
    public int CatID { get; set; }
    public string Name { get; set; } = "";
}
```

## Multiple Entities in One Service

```csharp
public class DataService : DataCore
{
    public DataService(string cs) : base(cs) { }

    private IEntities<Person>? _people;
    private IEntities<Order>? _orders;

    public IEntities<Person> People => _people ??= CreateEntity<Person>();
    public IEntities<Order> Orders => _orders ??= CreateEntity<Order>();
}
```

## IUpdatable Entity

```csharp
[TableName("Articles")]
public class Article : ITable, IUpdatable
{
    [PrimaryKey(isAutoIncremental: true)]
    public int Id { get; set; }
    public string Title { get; set; } = "";
    public DateTime CreatedDateTime { get; set; }
    public DateTime UpdatedDateTime { get; set; }
}
// Add/Edit auto-set timestamps
```

## GUID Primary Key

```csharp
[TableName("Sessions")]
public class Session : ITable
{
    [PrimaryKey(isAutoIncremental: true, isGUID: true)]
    public string SessionId { get; set; } = "";
    public DateTime ExpiresAt { get; set; }
}
```

## Safe Parameterized Raw SQL

```csharp
using NiORM.SQLServer.Core;

var helper = new SqlParameterHelper();
var nameParam = helper.AddParameter(userInput);
var results = db.SqlRaw<Person>(
    $"SELECT * FROM People WHERE Name = {nameParam}");
```

## Error Handling Pattern

```csharp
using NiORM.SQLServer.Core;

try
{
    var person = db.People.Find(123);
}
catch (NiORMValidationException ex)
{
    // Invalid operation (e.g. Add on IView)
}
catch (NiORMConnectionException ex)
{
    // Connection string / network issue
}
catch (NiORMException ex)
{
    Console.WriteLine(ex.Message);
    if (ex.SqlQuery != null)
        Console.WriteLine($"Query: {ex.SqlQuery}");
}
```

## Try-Add Wrapper

```csharp
public bool TryAddPerson(Person person, out string error)
{
    error = "";
    try
    {
        People.Add(person);
        return true;
    }
    catch (NiORMException ex)
    {
        error = ex.Message;
        return false;
    }
}
```

## MongoDB Example

```csharp
using NiORM.Attributes;
using NiORM.Mongo;
using NiORM.Mongo.Core;

[CollectionName("Users")]
public class User : MongoCollection
{
    public string Username { get; set; } = "";
    public string Email { get; set; } = "";
}

public class MongoDataService : DataCore
{
    public MongoDataService(string conn, string db) : base(conn, db) { }

    public Entities<User> Users => CreateEntity<User>();
}

// Usage
var mongo = new MongoDataService("mongodb://localhost:27017", "MyDb");
mongo.Users.Add(new User { Username = "nima", Email = "n@example.com" });
var all = mongo.Users.List();
var one = mongo.Users.Find("507f1f77bcf86cd799439011");
```

## Common Mistakes

| Mistake | Fix |
|---|---|
| Missing `[TableName]` | Add attribute or runtime throws |
| Class doesn't implement `ITable` | Add `: ITable` |
| Using `FindByProperty` / `WhereMultiple` | Use `Where(p => p.X == value)` |
| `Where(p => p.Age > 18)` | Not supported; use raw SQL with parameters |
| Empty connection string | `ArgumentNullException` at construction |
| Editing `IView` entity | Use read-only queries only |
| String concat in SQL with user input | Use `SqlParameterHelper` |

## Connection String Examples

**LocalDB (from NiORM.Test):**
```
Data Source=(localdb)\MSSQLLocalDB;Initial Catalog=StoreDb;Integrated Security=True;Connect Timeout=30;Encrypt=False;TrustServerCertificate=False;
```

**SQL Server:**
```
Server=localhost;Database=MyDb;User Id=sa;Password=***;TrustServerCertificate=True;
```

**MongoDB:**
```
mongodb://localhost:27017
```
