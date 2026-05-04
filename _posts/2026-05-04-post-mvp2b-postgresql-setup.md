---
layout: post
title: "Car Management API — MVP 2B: PostgreSQL Setup and Connection"
description: "Creating the database, mapping C# enums to VARCHAR columns, solving Dapper's type handling, and swapping the DI registration from in-memory to real PostgreSQL."
date: 2026-05-04
category: Architecture
series: Car Management API
series_part: 6
---

*This is part 6 of the Car Management API series. [Part 5](/writing/2026-05-03-post-mvp2a-api) covers the API layer and CarsController.*

---

With `PostgreSqlCarRepository` written and the API layer in place, the missing piece was an actual database to talk to. MVP 2B is where the system stops using a list in memory and starts writing to real PostgreSQL tables — and where a few decisions made earlier in the series pay off.

## Designing the schema

Before writing any SQL, the schema needed thinking through. The `Car` model from MVP 1 has fifteen properties, three of which are C# enums: `CarBodyType`, `CarGearboxType`, and `CarFuelType`.

PostgreSQL has a native `ENUM` type, which seemed like the obvious match. The problem is that Dapper does not automatically map a PostgreSQL enum column back to a C# enum — it requires custom type handlers registered with Npgsql. A `VARCHAR` column storing the string value (`'Saloon'`, `'Automatic'`, `'Diesel'`) has no such problem. Dapper's own type handler mechanism can take care of the conversion cleanly.

The other decision worth noting: PostgreSQL has a native `UUID` type, which maps directly to C#'s `Guid`. Using it means no casting, no string comparisons, and the database enforcing valid UUID format at the storage level.

Here is the complete setup script:

```sql
-- Create the database
CREATE DATABASE car_management_system;

-- Connect to it
\c car_management_system;

-- Create the cars table
CREATE TABLE cars
(
    id          UUID PRIMARY KEY,
    make        VARCHAR(100) NOT NULL,
    model       VARCHAR(100) NOT NULL,
    colour      VARCHAR(50)  NOT NULL,
    year        INT          NOT NULL CHECK (year >= 1900 AND year <= EXTRACT(YEAR FROM CURRENT_DATE) + 1),
    price       NUMERIC(12,2) NOT NULL CHECK (price >= 0),
    mileage     INT          NOT NULL CHECK (mileage >= 0),
    gearbox     VARCHAR(20)  NOT NULL,
    body_type   VARCHAR(20)  NOT NULL,
    doors       INT          NOT NULL CHECK (doors >= 2 AND doors <= 7),
    seats       INT          NOT NULL CHECK (seats >= 1 AND seats <= 9),
    engine_size NUMERIC(4,1) NOT NULL CHECK (engine_size >= 0),
    boot_space  INT          NOT NULL CHECK (boot_space >= 0),
    fuel_type   VARCHAR(20)  NOT NULL
);

-- Indexes for the most common query patterns
CREATE INDEX idx_cars_make  ON cars(make);
CREATE INDEX idx_cars_model ON cars(model);
CREATE INDEX idx_cars_price ON cars(price);
CREATE INDEX idx_cars_year  ON cars(year);

-- Sample row to verify the setup
INSERT INTO cars (id, make, model, colour, year, price, mileage, gearbox, body_type, doors, seats, engine_size, boot_space, fuel_type)
VALUES (gen_random_uuid(), 'BMW', 'X5', 'Black', 2022, 45999.99, 18000, 'Automatic', 'SUV', 5, 5, 3.0, 650, 'Diesel');

-- Verify
SELECT * FROM cars;
```

The CHECK constraints are worth the small overhead. `year` cannot be in the future by more than one model year, `price` and `mileage` cannot go negative, `doors` is bounded between 2 and 7. These are invariants that should hold regardless of what the application layer does — enforcing them at the database level means they hold even if something bypasses the API.

`created_at` and `updated_at` were considered and deliberately left out. There is no audit requirement yet and they are not on the `Car` model. Adding columns to a table that have no corresponding property in the domain model creates a gap that will need explaining later. They can be added in a future migration when there is a real reason for them.

## Creating a dedicated database user

Running the application as the `postgres` superuser is not something you want in any environment. A dedicated user with only the permissions it needs is the right approach even for local development — it builds the habit.

```sql
CREATE USER cms_user WITH PASSWORD 'test123';
GRANT ALL PRIVILEGES ON DATABASE car_management_system TO cms_user;

\c car_management_system

GRANT ALL ON SCHEMA public TO cms_user;
GRANT ALL ON ALL TABLES IN SCHEMA public TO cms_user;
```

PostgreSQL 15 and later require the schema-level grants separately from the database-level grant. Without them, `cms_user` can connect to the database but cannot query any tables.

## The connection string

The connection string lives in `appsettings.json`. Nothing sensitive goes in code:

```json
"ConnectionStrings": {
  "DefaultConnection": "Host=localhost;Port=5432;Database=car_management_system;Username=cms_user;Password=test123"
}
```

When the project moves to Docker later, `Host=localhost` becomes `Host=db` — the Docker Compose service name. That is the only change needed because the connection string is configuration, not code.

## Solving the enum mapping problem

This is the most interesting problem in MVP 2B. The database stores `'Automatic'` as a `VARCHAR`. The `Car` model has `CarGearboxType Gearbox`. Dapper reads a `string` from the database and has to set it on a property typed as `CarGearboxType`. Without any help, that fails.

The solution is Dapper's `TypeHandler` mechanism. A type handler is a class that tells Dapper exactly how to convert a type when reading from or writing to the database. Two methods: `SetValue` (C# → database) and `Parse` (database → C#).

I created a `DapperTypeHandlers.cs` file in the `DataAccess` folder of the class library. All three enum handlers live in the same file:

```csharp
using CMS_ClassLibrary.Models;
using Dapper;
using System.Data;

namespace CMS_ClassLibrary.DataAccess;

public class DapperTypeHandlers
{
    public class CarGearboxTypeHandler : SqlMapper.TypeHandler<CarGearboxType>
    {
        public override void SetValue(IDbDataParameter parameter, CarGearboxType value)
        {
            parameter.Value = value.ToString();
        }

        public override CarGearboxType Parse(object value)
        {
            return Enum.Parse<CarGearboxType>(value.ToString()!);
        }
    }

    public class CarBodyTypeHandler : SqlMapper.TypeHandler<CarBodyType>
    {
        public override void SetValue(IDbDataParameter parameter, CarBodyType value)
        {
            parameter.Value = value.ToString();
        }

        public override CarBodyType Parse(object value)
        {
            return Enum.Parse<CarBodyType>(value.ToString()!);
        }
    }

    public class CarFuelTypeHandler : SqlMapper.TypeHandler<CarFuelType>
    {
        public override void SetValue(IDbDataParameter parameter, CarFuelType value)
        {
            parameter.Value = value.ToString();
        }

        public override CarFuelType Parse(object value)
        {
            return Enum.Parse<CarFuelType>(value.ToString()!);
        }
    }
}
```

`Enum.Parse<T>` is the .NET method for converting a string to an enum value. `'Automatic'.ToString()` becomes `CarGearboxType.Automatic`. The conversion logic lives in one place, the `Car` model stays clean, and nothing else in the stack needs to know how enums are stored.

## Wiring it up in Program.cs

Three things need to happen before the app starts: register the Dapper type handlers, tell Dapper to match snake_case column names to PascalCase properties, and swap the DI registration from the in-memory repository to the PostgreSQL one.

```csharp
using CMS_ClassLibrary.DataAccess;
using CMS_ClassLibrary.Repository;
using CMS_ClassLibrary.Services;
using Dapper;

// Dapper configuration
DefaultTypeMap.MatchNamesWithUnderscores = true;
SqlMapper.AddTypeHandler(new DapperTypeHandlers.CarGearboxTypeHandler());
SqlMapper.AddTypeHandler(new DapperTypeHandlers.CarBodyTypeHandler());
SqlMapper.AddTypeHandler(new DapperTypeHandlers.CarFuelTypeHandler());

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// Swap to the real repository
builder.Services.AddScoped<ICarRepository, PostgreSqlCarRepository>();
builder.Services.AddScoped<ICarService, CarService>();

builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowAll", policy =>
        policy.AllowAnyOrigin().AllowAnyHeader().AllowAnyMethod());
});

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI(c => c.RoutePrefix = string.Empty);
}

app.UseCors("AllowAll");
app.UseAuthorization();
app.MapControllers();
app.Run();
```

`DefaultTypeMap.MatchNamesWithUnderscores = true` handles the column name mismatch — `body_type` maps to `BodyType`, `engine_size` maps to `EngineSize`, and so on. Without it, Dapper returns null for any property whose database column name contains underscores.

The DI registration change is a single line — `InMemoryCarRepository` becomes `PostgreSqlCarRepository`. That is the payoff of the interface-driven design established in MVP 1. `CarService` does not change. `CarsController` does not change. The only change in the application layer is which concrete class is registered against `ICarRepository`.

## Making Swagger the default page

A small quality-of-life change: rather than navigating to `/swagger` manually every time the application starts, Swagger can be served from the root. Two changes are needed.

In `Program.cs`:

```csharp
app.UseSwaggerUI(c => c.RoutePrefix = string.Empty);
```

In `launchSettings.json`, update `launchUrl` in all three profiles from `"cars"` to `"swagger"`:

```json
"launchUrl": "swagger"
```

The browser now opens directly to the Swagger UI when the application starts in development.

## Verifying it works

With the database created, the user configured, the connection string in place, and the application running — a POST to `api/Cars` through Swagger with a valid car body should return `true` and write a row to the database.

The real verification is querying the database directly afterwards and seeing the data. At this point the system is no longer pretending — it is writing to and reading from a real PostgreSQL instance, with proper type mapping, a dedicated database user, and a clean separation between configuration and code.

## What this MVP proves

The in-memory repository was always a placeholder. MVP 2B is where the architecture stops being theoretical. The interface-driven design, the async signatures, the separation of concerns — all of it was built with this moment in mind. Swapping the underlying data store required changing one line in `Program.cs`.

The next step is a vanilla JavaScript frontend — a simple UI to drive the API without needing Swagger for every operation.

---

*Next: building a vanilla JS frontend to drive the car management API.*
