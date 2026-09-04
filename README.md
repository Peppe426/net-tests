# net-tests (.NET 10)

A boilerplate and reference architecture demonstrating automated testing best practices in **.NET 10** using **NUnit**, **FluentAssertions**, and **ASP.NET Core WebApplicationFactory**.

---

## When to Use What Test

| Test Type | When to Use | Execution Speed | Dependencies / Scope |
| :--- | :--- | :--- | :--- |
| **Unit Test** | Pure business logic, algorithmic methods, calculations, domain entities, and isolated utility classes. | ⚡ Ultra fast (milliseconds) | No external I/O, no database, no HTTP. Dependencies are mocked or stubbed. |
| **Integration Test** | Verifying multiple components working together: ASP.NET Core request pipeline, middleware, controller/minimal API endpoints, DI registration, and real database persistence. | 🚀 Fast (seconds) | Uses in-memory test host (`WebApplicationFactory`) and in-memory SQLite database. |
| **Acceptance / E2E Test** | End-to-end user workflows, validating business requirements across the complete deployed application or microservices. | 🐢 Slowest | Full environment with external services, real networks, or UI drivers. |

---

## Test Examples

### 1. Unit Test Example

Unit tests verify small, isolated units of logic in complete isolation without starting the web server or database:

```csharp
using NUnit.Framework;
using FluentAssertions;
using ApiSample;

namespace UnitTests;

[TestFixture]
public class WeatherForecastTests
{
    [Test]
    public void TemperatureF_ShouldConvertCorrectlyFromCelsius()
    {
        // Arrange
        var forecast = new WeatherForecast(1, DateOnly.FromDateTime(DateTime.Now), 25, "Warm");

        // Act & Assert
        forecast.TemperatureF.Should().Be(76); // 32 + (int)(25 / 0.5556)
    }
}
```

### 2. Integration Test Example (Endpoint Call)

Integration tests can invoke the full HTTP pipeline via the test host's `HttpClient`:

```csharp
using System.Net;
using NUnit.Framework;
using IntegrationTests;
using Microsoft.EntityFrameworkCore;

namespace ApiSampleNunitTests;

[TestFixture]
public class EndpointTests : IntegrationTest<Program, DbContext>
{
    [Test]
    public async Task ShouldReturnSuccessStatusCodeOnGetWeatherForecast()
    {
        // When: invoking the endpoint through in-memory ASP.NET Core test host
        var response = await Client.GetAsync("/weatherforecast");

        // Then: verify HTTP response
        response.StatusCode.Should().Be(HttpStatusCode.OK);
    }
}
```

---

## Highlighting Integration Testing: DI & In-Memory Database

The core strength of the integration test harness in `Base` and `IntegrationTests` is its seamless handling of **Dependency Injection (DI)** and **relational in-memory SQLite database isolation**.

### Why SQLite In-Memory instead of EF Core InMemory?
The EF Core `InMemory` provider is not a relational database—it does not enforce foreign keys, relational constraints, or transaction semantics. This solution uses SQLite in-memory (`DataSource=:memory:`):
- **True Relational Engine**: Validates SQL constraints, migrations, and relational behavior.
- **Persistent Open Connection**: SQLite in-memory databases exist only while their connection remains open. `WebHostFactory` creates and maintains an open `SqliteConnection` for the test fixture lifecycle.
- **State Cleanliness**: Each fixture recreates a clean database schema using `context.Database.EnsureDeleted()` and `context.Database.EnsureCreated()`.

### Resolving Services via Dependency Injection (`ResolveService<T>`)
The test base class exposes `ResolveService<T>()`, which retrieves registered application services from the test server's DI container scope. This allows tests to:
1. Seed initial data via repositories or DbContext.
2. Directly invoke business services within the ASP.NET Core container context.
3. Assert state changes directly in the database.

### Full Example: DI + In-Memory Database + HTTP Client

From `ApiSampleNunitTests/PersistentDataTests.cs`:

```csharp
using ApiSample.Models;
using ApiSample.Repository;
using FluentAssertions;
using IntegrationTests;
using System.Text.Json;

namespace ApiSampleNunitTests;

public class PersistentDataTests : IntegrationTest<Program, TestDbContext>
{
    [Test]
    public async Task ShouldSaveDataToDatabase()
    {
        // 1. Arrange & Act: Call API endpoint via test HttpClient
        var response = await Client.GetAsync("/weatherforecast");
        response.EnsureSuccessStatusCode();

        var content = await response.Content.ReadAsStringAsync();
        var forecasts = JsonSerializer.Deserialize<List<WeatherForecast>>(content);

        // 2. Dependency Injection: Resolve service directly from the DI container
        var provider = ResolveService<WeatherProvider>();

        // 3. In-Memory Database: Verify initial state is empty
        DbContext.Set<WeatherForecast>().Count().Should().Be(0);

        // 4. Act: Execute repository logic using resolved DI provider
        var outcome = await provider.SaveWeatherToDatabase(forecasts!.First());
        outcome.Should().Be(1);

        // 5. Assert: Verify the entity was persisted in the in-memory SQLite database
        DbContext.Set<WeatherForecast>().Count().Should().Be(1);
    }
}
```

---

## Repository Structure

- **`Base`**: Core test infrastructure including `WebHostFactory<TEntryPoint, TContext>`, SQLite in-memory connection management, and DI service resolution.
- **`ApiSample`**: ASP.NET Core Web API sample targeting `net10.0` with SQLite and Entity Framework Core.
- **`IntegrationTests`**: Abstract generic base class `IntegrationTest<TEntryPoint, TContext>` managing test lifecycle, `HttpClient`, and database cleanup.
- **`ApiSampleNunitTests`**: Integration tests testing `ApiSample` endpoints and database persistence.
- **`UnitTests`**: Unit test suite scaffolding.

---

## Getting Started

### Prerequisites
- [.NET 10 SDK](https://dotnet.microsoft.com/)

### Build and Run Tests
```bash
# Clone the repository
git clone https://github.com/Peppe426/net-tests.git
cd net-tests

# Restore dependencies
dotnet restore Base/Base.sln

# Build the solution
dotnet build Base/Base.sln

# Run all tests
dotnet test Base/Base.sln
```

## License
This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.
