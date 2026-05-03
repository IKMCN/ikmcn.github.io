# MVP 2a: Swapping to PostgreSQL with Dapper

In MVP 1 I built the car management system with an in-memory repository. It was enough to get the architecture right and the tests passing, but it was never meant to be permanent. Data disappears the moment the application stops. MVP 2a is where I replace that with a real database — PostgreSQL — using Dapper as the data access layer.

## Why a New Repository, Not a Modified One

The first question I had to answer was: do I change the existing `CarRepository`, or do I create something new?

The answer came from looking at `ICarRepository`. The interface defines the contract — what a repository must be able to do. The in-memory `CarRepository` is one implementation of that contract. What I needed was a second implementation: one that talks to PostgreSQL instead of a list in memory.

So I created `PostgreSqlCarRepository`, which implements `ICarRepository` exactly. The `CarService` depends on `ICarRepository`, not on any specific implementation. That means I can swap the underlying data store without touching the service layer at all. This is the payoff of the interface-driven design I put in place from the start.

## Choosing Dapper

I chose Dapper over Entity Framework deliberately. Dapper is a micro-ORM — it sits just above raw SQL and handles mapping query results to C# objects, but it doesn't abstract away the SQL itself. I want to understand what's happening at the database level, and Dapper keeps that visible.

Two NuGet packages are needed:

- `Npgsql` — the PostgreSQL driver for .NET
- `Dapper` — the micro-ORM for mapping results

Worth noting: if you've seen Dapper examples using `System.Data.SqlClient`, that's for SQL Server. For PostgreSQL you use `NpgsqlConnection` instead.

## Getting the Connection String

`PostgreSqlCarRepository` needs to know how to connect to the database. In ASP.NET Core, connection strings live in `appsettings.json`:

```json
"ConnectionStrings": {
  "DefaultConnection": "Host=localhost;Port=5432;Database=CarManagementSystem;Username=cms_user;Password=test123"
}
```

The repository gets access to this by injecting `IConfiguration` through its constructor:

```csharp
public PostgreSqlCarRepository(IConfiguration configuration)
{
    _connectionString = configuration.GetConnectionString("DefaultConnection")!;
}
```

## The snake_case Problem

PostgreSQL conventionally uses `snake_case` for column names — `body_type`, `engine_size`, `boot_space`. My C# `Car` model uses `PascalCase` — `BodyType`, `EngineSize`, `BootSpace`. Without any configuration, Dapper wouldn't know how to map one to the other.

The fix is a single line in `Program.cs`:

```csharp
DefaultTypeMap.MatchNamesWithUnderscores = true;
```

This tells Dapper to treat underscores as non-existent when matching column names to property names, so `body_type` maps correctly to `BodyType`.

## The Repository Methods

Each method follows the same pattern: open a connection, execute SQL with Dapper, return a result.

**GetAllAsync** uses `QueryAsync<Car>` to fetch all rows and map them to `Car` objects:

```csharp
public async Task<IEnumerable<Car>> GetAllAsync()
{
    using var connection = new NpgsqlConnection(_connectionString);
    var cars = await connection.QueryAsync<Car>("SELECT * FROM cars");
    return cars;
}
```

**GetByIdAsync** uses `QueryFirstOrDefaultAsync<Car>` to fetch a single row, returning null if not found. Parameters are passed as an anonymous object — Dapper matches the property name `Id` to the `@Id` placeholder in the SQL:

```csharp
public async Task<Car?> GetByIdAsync(Guid id)
{
    using var connection = new NpgsqlConnection(_connectionString);
    var car = await connection.QueryFirstOrDefaultAsync<Car>(
        "SELECT * FROM cars WHERE id = @Id",
        new { Id = id });
    return car;
}
```

**CreateAsync** and **DeleteByIdAsync** use `ExecuteAsync` — they don't return rows, just confirmation that something happened. The `rowsAffected > 0` check converts that into a meaningful bool:

```csharp
public async Task<bool> DeleteByIdAsync(Guid id)
{
    using var connection = new NpgsqlConnection(_connectionString);
    var rowsAffected = await connection.ExecuteAsync(
        "DELETE FROM cars WHERE id = @Id",
        new { Id = id });
    return rowsAffected > 0;
}
```

**UpdateAsync** passes the full `Car` object directly to Dapper. Rather than building a manual anonymous object, Dapper maps the `Car` properties to the `@` parameters by name automatically:

```csharp
public async Task<bool> UpdateAsync(Car car)
{
    using var connection = new NpgsqlConnection(_connectionString);
    var sql = @"UPDATE cars SET 
                make = @Make, model = @Model, colour = @Colour,
                year = @Year, price = @Price, mileage = @Mileage,
                gearbox = @Gearbox, body_type = @BodyType, doors = @Doors,
                seats = @Seats, engine_size = @EngineSize,
                boot_space = @BootSpace, fuel_type = @FuelType
                WHERE id = @Id";
    var rowsAffected = await connection.ExecuteAsync(sql, car);
    return rowsAffected > 0;
}
```

## What's Next

The repository is wired up. The next step is creating the `cars` table in PostgreSQL and running the API to test everything end to end.
