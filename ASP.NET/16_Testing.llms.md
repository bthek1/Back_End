# Testing

Testing ASP.NET Core with xUnit, WebApplicationFactory, Testcontainers and a real PostgreSQL database.

Author

Benedict Thekkel

The most valuable test for an API sends a real HTTP request through the real pipeline to a real database. `WebApplicationFactory` runs the whole app in memory, and Testcontainers supplies PostgreSQL. Add unit tests for logic with many branches.

> **NOTE:**
>
> The code on this page has not been compiled or run yet. It targets .NET 11 and C# 15.

------------------------------------------------------------------------

## 1. The Test Pyramid

| Level | Scope | Speed | Use it for |
|----|----|----|----|
| Unit | One class | Milliseconds | Calculations and business rules |
| Integration | HTTP to database | Seconds | Routing, auth, validation and SQL |
| End-to-end | Browser to database | Slow | A few critical user journeys, with Playwright |

For a typical CRUD API, put most effort into integration tests. They catch the wiring errors that unit tests cannot see.

------------------------------------------------------------------------

## 2. Project Setup

``` bash
dotnet new install xunit.v3.templates
dotnet new xunit3 -n ItemsApi.Tests
cd ItemsApi.Tests
dotnet add reference ../ItemsApi/ItemsApi.csproj
dotnet add package Microsoft.AspNetCore.Mvc.Testing
dotnet add package Testcontainers.PostgreSql
dotnet add package AwesomeAssertions
```

``` bash
dotnet test
dotnet test --filter "FullyQualifiedName~ItemsApiTests"
dotnet test --collect:"XPlat Code Coverage"
```

Add `using AwesomeAssertions;` to use the `Should()` assertions. Current .NET versions generate a public `Program` class, so the test project can reference it.

------------------------------------------------------------------------

## 3. Unit Tests

``` csharp
public sealed class PriceCalculatorTests
{
    [Fact]
    public void Total_adds_tax()
    {
        var total = PriceCalculator.Total(100m, taxRate: 0.1m);

        total.Should().Be(110m);
    }

    [Theory]
    [InlineData(0, 0)]
    [InlineData(10, 11)]
    [InlineData(99.99, 109.99)]
    public void Total_rounds_to_cents(decimal price, decimal expected)
    {
        PriceCalculator.Total(price, 0.1m).Should().Be(expected);
    }
}
```

| xUnit                          | pytest                     |
|--------------------------------|----------------------------|
| `[Fact]`                       | `def test_x():`            |
| `[Theory]` with `[InlineData]` | `@pytest.mark.parametrize` |
| Constructor                    | Function-scoped fixture    |
| `IClassFixture<T>`             | Class-scoped fixture       |
| `ICollectionFixture<T>`        | Session-scoped fixture     |

------------------------------------------------------------------------

## 4. Control Time

Inject `TimeProvider` instead of reading the clock directly.

``` csharp
public sealed class TrialService(TimeProvider time)
{
    public bool IsExpired(DateTimeOffset endsAt) => time.GetUtcNow() > endsAt;
}
```

``` bash
dotnet add package Microsoft.Extensions.TimeProvider.Testing
```

``` csharp
using Microsoft.Extensions.Time.Testing;

[Fact]
public void Trial_expires_after_the_end_date()
{
    var time = new FakeTimeProvider(new DateTimeOffset(2026, 1, 1, 0, 0, 0, TimeSpan.Zero));
    var service = new TrialService(time);
    var endsAt = time.GetUtcNow().AddDays(7);

    time.Advance(TimeSpan.FromDays(8));

    service.IsExpired(endsAt).Should().BeTrue();
}
```

------------------------------------------------------------------------

## 5. The Application Fixture

This fixture starts one PostgreSQL container, applies migrations and runs the app against it.

``` csharp
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Mvc.Testing;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;
using Testcontainers.PostgreSql;

public sealed class ApiFactory : WebApplicationFactory<Program>, IAsyncLifetime
{
    private readonly PostgreSqlContainer _postgres = new PostgreSqlBuilder()
        .WithImage("postgres:17-alpine")
        .Build();

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.UseEnvironment("Testing");
        builder.UseSetting("ConnectionStrings:Default", _postgres.GetConnectionString());
    }

    public async ValueTask InitializeAsync()
    {
        await _postgres.StartAsync();

        await using var scope = Services.CreateAsyncScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        await db.Database.MigrateAsync();
    }

    public override async ValueTask DisposeAsync()
    {
        await _postgres.DisposeAsync();
        await base.DisposeAsync();
    }
}
```

Applying real migrations also tests that they work. Avoid the EF Core in-memory provider, because it does not behave like PostgreSQL.

------------------------------------------------------------------------

## 6. Integration Tests

``` csharp
using System.Net;
using System.Net.Http.Json;

public sealed class ItemsApiTests(ApiFactory factory) : IClassFixture<ApiFactory>
{
    private readonly HttpClient _client = factory.CreateClient();
    private static CancellationToken Ct => TestContext.Current.CancellationToken;

    [Fact]
    public async Task Create_then_get_returns_the_item()
    {
        var created = await _client.PostAsJsonAsync(
            "/api/items", new { name = "Pen", price = 2.50m }, Ct);

        created.StatusCode.Should().Be(HttpStatusCode.Created);

        var item = await _client.GetFromJsonAsync<ItemResponse>(created.Headers.Location, Ct);

        item!.Name.Should().Be("Pen");
    }

    [Fact]
    public async Task Create_rejects_an_empty_name()
    {
        var response = await _client.PostAsJsonAsync(
            "/api/items", new { name = "", price = 2.50m }, Ct);

        response.StatusCode.Should().Be(HttpStatusCode.BadRequest);
    }
}
```

------------------------------------------------------------------------

## 7. Fake Authentication

Replace token validation with a handler that signs in a test user.

``` csharp
using System.Security.Claims;
using System.Text.Encodings.Web;
using Microsoft.AspNetCore.Authentication;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;

public sealed class TestAuthHandler(
    IOptionsMonitor<AuthenticationSchemeOptions> options,
    ILoggerFactory logger,
    UrlEncoder encoder) : AuthenticationHandler<AuthenticationSchemeOptions>(options, logger, encoder)
{
    public const string Scheme = "Test";

    protected override Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        if (!Request.Headers.TryGetValue("X-Test-User", out var user))
        {
            return Task.FromResult(AuthenticateResult.NoResult());
        }

        var identity = new ClaimsIdentity([new Claim("sub", user.ToString())], Scheme);
        var ticket = new AuthenticationTicket(new ClaimsPrincipal(identity), Scheme);
        return Task.FromResult(AuthenticateResult.Success(ticket));
    }
}
```

``` csharp
builder.ConfigureTestServices(services =>
    services.AddAuthentication(TestAuthHandler.Scheme)
        .AddScheme<AuthenticationSchemeOptions, TestAuthHandler>(TestAuthHandler.Scheme, _ => { }));
```

Send the `X-Test-User` header to act as a user. Omit it to test the `401` response. Test the real authorization policies, and replace only the token check.

------------------------------------------------------------------------

## 8. Replace External Services

Fake services that you do not own, such as email or payment providers.

``` csharp
var client = factory.WithWebHostBuilder(builder =>
    builder.ConfigureTestServices(services =>
    {
        services.RemoveAll<IEmailSender>();
        services.AddSingleton<IEmailSender, FakeEmailSender>();
    })).CreateClient();
```

Do not fake your own database. It is the part most likely to differ from your assumptions.

------------------------------------------------------------------------

## 9. Keep Tests Isolated

| Strategy                      | Speed   | Note                        |
|-------------------------------|---------|-----------------------------|
| Unique data per test          | Fastest | Use a new GUID in each name |
| Reset tables with **Respawn** | Fast    | Deletes rows between tests  |
| One container per test class  | Slower  | Strong isolation            |

Tests in one xUnit class run in sequence, and test classes run in parallel by default. Classes that share a fixture through a collection do not run in parallel with each other.

------------------------------------------------------------------------

## 10. Compared With pytest-django

| pytest-django            | .NET                               |
|--------------------------|------------------------------------|
| `pytest`                 | `dotnet test`                      |
| `APIClient`              | `factory.CreateClient()`           |
| `force_authenticate`     | Test authentication handler        |
| `@pytest.mark.django_db` | Testcontainers fixture             |
| `freezegun`              | `FakeTimeProvider`                 |
| `override_settings`      | `UseSetting`                       |
| `unittest.mock.patch`    | `ConfigureTestServices`            |
| `factory_boy`            | Test data builder classes or Bogus |

------------------------------------------------------------------------

## 11. Common Mistakes

| Mistake | What goes wrong | Do this instead |
|----|----|----|
| Using the EF Core in-memory provider | SQL behaviour is not tested | Use Testcontainers |
| Asserting on the exact JSON text | Tests fail on harmless changes | Deserialize and check fields |
| Tests that depend on run order | Failures appear at random | Isolate the data |
| `Task.Delay` to wait for background work | Slow, flaky tests | Poll a condition with a timeout |
| Mocking everything | Tests pass while wiring is broken | Prefer integration tests |
| Removing authorization in tests | Permission bugs go unnoticed | Fake only authentication |

------------------------------------------------------------------------

Back to top
