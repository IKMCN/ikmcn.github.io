---
layout: post
title: "Designing an agnostic API from the ground up"
description: "How interfaces and dependency injection let you swap your frontend, authentication, and database without touching your core logic."
date: 2026-04-25
category: Architecture
---

When I started thinking about how to structure a car dealership API as a learning project, I kept coming back to one question: what happens when the requirements change? What if I want to swap JWT for Auth0? What if I want to switch from PostgreSQL to SQL Server for testing? What if the JavaScript frontend gets replaced by a mobile app?

The answer, it turns out, is the same in every case: **interfaces and dependency injection**.

## The problem with tight coupling

The naive approach looks like this. Your controller creates a `CarService` directly:

```csharp
public class CarsController : ControllerBase
{
    private CarService _carService = new CarService();
}
```

Now your controller is locked to `CarService`. You can't test it without hitting the real database. You can't swap the implementation without changing the controller. Every time something changes downstream, you're opening files you shouldn't need to touch.

## The interface as a contract

An interface defines what something must be able to do, without caring how it does it:

```csharp
public interface ICarService
{
    List<Car> GetAllCars();
    Car GetCarById(int id);
    void AddCar(Car car);
}
```

Now `CarService` implements that contract:

```csharp
public class CarService : ICarService
{
    public List<Car> GetAllCars() { /* hits the real database */ }
}
```

And for testing, a fake version implements the same contract:

```csharp
public class FakeCarService : ICarService
{
    public List<Car> GetAllCars() { /* returns hardcoded data */ }
}
```

## Dependency injection wires it together

The controller no longer creates its dependencies — they get handed in:

```csharp
public class CarsController : ControllerBase
{
    private readonly ICarService _carService;

    public CarsController(ICarService carService)
    {
        _carService = carService;
    }
}
```

The controller has no idea what `_carService` actually is. It just knows it implements `ICarService`. In your DI setup you decide:

```csharp
services.AddTransient<ICarService, CarService>();        // production
// or
services.AddTransient<ICarService, FakeCarService>();   // testing
```

One line. Nothing else changes.

## Applying the same pattern to authentication

The same principle applies to authentication. Define the contract:

```csharp
public interface IAuthService
{
    string GenerateToken(User user);
    bool ValidateToken(string token);
}
```

Implement it for JWT today:

```csharp
public class JwtAuthService : IAuthService { ... }
```

When you want Auth0 later:

```csharp
public class Auth0AuthService : IAuthService { ... }
```

Your controllers never change. You swap one line in your DI registration.

## The full architecture

This is what the design looks like when you apply the pattern consistently:

```
JavaScript Frontend
        ↓ HTTP
ASP.NET Core API
        ↓ IAuthService     → JwtAuthService / Auth0AuthService
        ↓ ICarService      → CarService / FakeCarService
        ↓ ICarRepository   → PostgresCarRepository / FakeCarRepository
PostgreSQL in Docker
```

Every arrow is an interface. Every implementation is swappable without touching the layer above it.

## Why this matters

I arrived at this design not from a textbook but from working through the problem step by step. The moment you understand interfaces and DI, the architecture follows naturally.

The frontend is agnostic because your API just returns JSON — anything that can make an HTTP request can consume it.

The authentication is agnostic because your controllers depend on `IAuthService`, not a specific provider.

The database is agnostic because your services depend on `ICarRepository`, not a specific database technology.

That's clean architecture in practice. Not as an abstract concept — as a direct consequence of the two patterns you learned first.
