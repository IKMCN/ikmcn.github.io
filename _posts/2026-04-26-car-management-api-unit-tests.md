---
layout: post
title: "Car Management API — MVP 1: Unit Testing the Service Layer"
description: "Writing the first unit tests for the car management system. What to test, what not to test, and why the interface pattern makes testing clean."
date: 2026-04-26
category: Testing
series: Car Management API
series_part: 3
---

*This is part 3 of the Car Management API series. [Part 2](/writing/2026-04-26-car-management-api-mvp1) covers the class library structure.*

---

MVP 1 has no database, no API, and no frontend. What it does have is a service layer sitting on top of a repository — and that service layer can be tested right now without any of those things existing.

This post covers the first unit tests for the system: what to test, what to skip for now, and how the interface pattern built in MVP 1 makes testing clean.

## What to test at this stage

The class library has three layers: the `Car` model, `CarRepository`, and `CarService`. Not all of them are worth testing right now.

**`Car` model** — it is a data class with properties. There is no logic to test. Skipped for now.

**`CarRepository`** — the in-memory implementation is a `List<Car>` with five methods wrapped around it. Testing that `List.Add()` works is testing the .NET framework, not our code. The repository tests will come in MVP 2a when there is a real PostgreSQL implementation worth verifying.

**`CarService`** — this is where the tests live. The service has real behaviour: it calls the repository, handles the results, and returns them to the caller. That behaviour is worth proving.

## Why interfaces make testing clean

The whole point of `ICarRepository` existing is this moment. Instead of spinning up a real database to test the service, a fake repository can be injected that returns predictable data. The service has no idea it is talking to a fake — it just sees something that implements `ICarRepository`.

This is the value of dependency injection in practice. Not just for swapping databases in production, but for making tests fast, isolated, and reliable.

## The fake repository

The fake lives in a `Fakes` folder inside the test project. It implements `ICarRepository` but instead of a database it uses a pre-populated list with two known cars:

```csharp
using CMS_ClassLibrary.Models;
using CMS_ClassLibrary.Repository;

namespace CMS_Testing.Fakes;

public class FakeCarRepository : ICarRepository
{
    private readonly Guid _knownId = Guid.Parse("a1b2c3d4-e5f6-7890-abcd-ef1234567890");
    private readonly List<Car> _cars;

    public FakeCarRepository()
    {
        _cars = new List<Car>
        {
            new Car { Id = _knownId, Make = "Honda", Model = "Civic" },
            new Car { Make = "Ford", Model = "Escort" }
        };
    }

    public Guid GetKnownId() => _knownId;

    public Task<bool> CreateAsync(Car car)
    {
        _cars.Add(car);
        return Task.FromResult(true);
    }

    public Task<Car?> GetByIdAsync(Guid id)
    {
        var car = _cars.SingleOrDefault(x => x.Id == id);
        return Task.FromResult(car);
    }

    public Task<IEnumerable<Car>> GetAllAsync()
    {
        return Task.FromResult(_cars.AsEnumerable());
    }

    public Task<bool> UpdateAsync(Car car)
    {
        var carIndex = _cars.FindIndex(x => x.Id == car.Id);
        if (carIndex == -1) return Task.FromResult(false);
        _cars[carIndex] = car;
        return Task.FromResult(true);
    }

    public Task<bool> DeleteByIdAsync(Guid id)
    {
        var removedCount = _cars.RemoveAll(x => x.Id == id);
        return Task.FromResult(removedCount > 0);
    }
}
```

`GetKnownId()` is the key addition. Because `Car.Id` is generated automatically with `Guid.NewGuid()`, there is no way to know the Honda's Id from outside the fake. This method exposes it so tests can ask for that specific car by Id.

## The tests

Every test follows the same pattern — Arrange, Act, Assert. Set up what you need, call the method, check the result.

```csharp
using CMS_ClassLibrary.Models;
using CMS_ClassLibrary.Services;
using CMS_Testing.Fakes;

namespace CMS_Testing.Services;

public class CarServiceTests
{
    [Fact]
    public async Task GetAllCars_ReturnsAllCars()
    {
        // Arrange
        var fakeRepository = new FakeCarRepository();
        var carService = new CarService(fakeRepository);

        // Act
        var result = await carService.GetAllCars();

        // Assert
        Assert.Equal(2, result.Count());
    }

    [Fact]
    public async Task GetCarById_ReturnsCorrectCar()
    {
        // Arrange
        var fakeRepository = new FakeCarRepository();
        var carService = new CarService(fakeRepository);
        var knownId = fakeRepository.GetKnownId();

        // Act
        var result = await carService.GetCarById(knownId);

        // Assert
        Assert.NotNull(result);
        Assert.Equal("Honda", result.Make);
    }

    [Fact]
    public async Task CreateCar_ReturnsTrueAndIncreasesCount()
    {
        // Arrange
        var fakeRepository = new FakeCarRepository();
        var carService = new CarService(fakeRepository);
        var newCar = new Car { Make = "Toyota", Model = "Corolla" };

        // Act
        var result = await carService.CreateCar(newCar);
        var carTotal = await carService.GetAllCars();

        // Assert
        Assert.True(result);
        Assert.Equal(3, carTotal.Count());
    }

    [Fact]
    public async Task DeleteCar_ReturnsTrueWhenCarDeleted()
    {
        // Arrange
        var fakeRepository = new FakeCarRepository();
        var carService = new CarService(fakeRepository);
        var knownId = fakeRepository.GetKnownId();

        // Act
        var result = await carService.DeleteCar(knownId);

        // Assert
        Assert.True(result);
    }

    [Fact]
    public async Task UpdateCar_ReturnsTrueAndUpdatesCorrectly()
    {
        // Arrange
        var fakeRepository = new FakeCarRepository();
        var carService = new CarService(fakeRepository);
        var newCar = new Car { Make = "Toyota", Model = "Corolla" };

        // Act
        var result = await carService.CreateCar(newCar);
        var carTotal = await carService.GetAllCars();
        newCar.Make = "Ford";
        var updatedCar = await carService.UpdateCar(newCar);
        var carUpdatedTotal = await carService.GetAllCars();

        // Assert
        Assert.True(result);
        Assert.Equal(3, carTotal.Count());
        Assert.True(updatedCar);
        Assert.Equal("Ford", carUpdatedTotal.ElementAt(2).Make);
    }
}
```

## What each test proves

**GetAllCars** — the service returns everything the repository holds. Two cars in, two cars back.

**GetCarById** — the service returns the right car when asked for a specific Id. Not just any car — the Honda specifically.

**CreateCar** — the service reports success and the count increases. Both things need to be true — returning `true` is not enough if the car was not actually added.

**DeleteCar** — the service reports success when the car exists. A later MVP will add a test for deleting a car that does not exist.

**UpdateCar** — the service creates a car, changes its Make, updates it, then verifies the change is reflected in the list. This tests the full round trip.

## What is deliberately missing

These are unit tests — fast, isolated, no dependencies. Integration tests that run against a real PostgreSQL database come in MVP 2a. End-to-end tests that drive the full stack come later still.

The test suite will grow with each MVP. The habit of writing tests alongside the code starts here.

## Running the tests

From the terminal in your solution folder:

```bash
dotnet test
```

All five should pass. If one fails the output tells you exactly which assertion failed and what the actual value was versus the expected value.

---

*Next: wiring up GitHub Actions so these tests run automatically on every push.*
