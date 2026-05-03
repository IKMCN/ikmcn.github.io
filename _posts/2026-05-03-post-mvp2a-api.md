# MVP 2a: Building the API Layer

With the `PostgreSqlCarRepository` in place, the next step was building the Web API project that exposes the car management system over HTTP. This post covers the API project setup, dependency injection configuration, and the `CarsController`.

## Project Structure

MVP 1 was a class library — no API, no HTTP, just the domain model, interfaces, and in-memory repository. For MVP 2a I created a new ASP.NET Core Web API project and added a reference to the class library. The solution now has two projects:

- `CMS_ClassLibrary` — the domain, interfaces, and data access
- `CMS_WebAPI` — the API layer

## Wiring Up Dependency Injection

`Program.cs` is where ASP.NET Core's dependency injection container is configured. This is where you tell the framework: when something asks for `ICarRepository`, give it a `PostgreSqlCarRepository`. When something asks for `ICarService`, give it a `CarService`.

```csharp
builder.Services.AddScoped<ICarRepository, PostgreSqlCarRepository>();
builder.Services.AddScoped<ICarService, CarService>();
```

`AddScoped` means a new instance is created per HTTP request. That's the right choice for database-backed repositories.

The Dapper snake_case mapping is also configured here, before the app is built:

```csharp
DefaultTypeMap.MatchNamesWithUnderscores = true;
```

The full `Program.cs` also registers Swagger for API exploration during development, and a CORS policy that allows all origins — appropriate for local development, not for production.

## The Connection String

The connection string lives in `appsettings.json`, not in code. This keeps sensitive configuration out of the codebase:

```json
"ConnectionStrings": {
  "DefaultConnection": "Host=localhost;Port=5432;Database=CarManagementSystem;Username=cms_user;Password=test123"
}
```

`PostgreSqlCarRepository` reads this at startup via `IConfiguration`. When running in Docker later, `Host=localhost` will become `Host=db` — the Docker service name.

## CarsController

The controller handles incoming HTTP requests and delegates to `ICarService`. It does not talk to `ICarRepository` directly — that's the service layer's responsibility.

The flow is:

**HTTP Request → CarsController → ICarService → ICarRepository → PostgreSQL**

The controller injects `ICarService` through its constructor:

```csharp
[Route("api/[controller]")]
[ApiController]
public class CarsController : ControllerBase
{
    private readonly ICarService _carService;

    public CarsController(ICarService carService)
    {
        _carService = carService;
    }
}
```

The `[Route("api/[controller]")]` attribute means the base route is `api/Cars` — ASP.NET Core replaces `[controller]` with the class name minus the `Controller` suffix automatically.

## The Five Endpoints

**GET api/Cars** — returns all cars:

```csharp
[HttpGet]
public async Task<ActionResult<IEnumerable<Car>>> GetCars()
{
    try { return Ok(await _carService.GetAllCars()); }
    catch (Exception ex) { return StatusCode(500, "An error occurred"); }
}
```

**GET api/Cars/{id}** — returns a single car by id:

```csharp
[HttpGet("{id}")]
public async Task<ActionResult<Car>> GetCarsById(Guid id)
{
    try { return Ok(await _carService.GetCarById(id)); }
    catch (Exception ex) { return StatusCode(500, "An error occurred"); }
}
```

**POST api/Cars** — creates a new car. The `[FromBody]` attribute tells ASP.NET Core to deserialise the request body into a `Car` object:

```csharp
[HttpPost]
public async Task<ActionResult<bool>> Post([FromBody] Car car)
{
    var output = await _carService.CreateCar(car);
    return Ok(output);
}
```

**PUT api/Cars/{id}** — updates an existing car. The id comes from the URL, the updated data from the request body:

```csharp
[HttpPut("{id}")]
public async Task<ActionResult<bool>> Put(Guid id, [FromBody] Car car)
{
    var output = await _carService.UpdateCar(car);
    return Ok(output);
}
```

**DELETE api/Cars/{id}** — deletes a car by id:

```csharp
[HttpDelete("{id}")]
public async Task<ActionResult<bool>> Delete(Guid id)
{
    var output = await _carService.DeleteCar(id);
    return Ok(output);
}
```

## Error Handling

The GET endpoints use try/catch blocks to return a 500 status code if something goes wrong. Importantly, the error message returned to the caller is a generic `"An error occurred"` rather than `ex.Message` — the real exception details stay on the server and won't leak database structure or server information to the outside world. A proper logging pipeline will be added in a later MVP.

## What's Next

The code is complete. The next step is creating the PostgreSQL database and `cars` table, then testing all five endpoints through Swagger.
