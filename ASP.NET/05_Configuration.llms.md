# Configuration

Layered configuration in ASP.NET Core: appsettings files, environment variables, user secrets and validated strongly typed options.

Author

Benedict Thekkel

ASP.NET Core reads settings from several sources and merges them into one key-value store. A later source overrides an earlier one. The options pattern then turns those values into validated, typed objects.

> **NOTE:**
>
> The code on this page has not been compiled or run yet. It targets .NET 11 and C# 15.

------------------------------------------------------------------------

## 1. Sources and Precedence

`WebApplication.CreateBuilder` loads these sources. The last source wins.

| Order | Source                           | Use it for                      |
|-------|----------------------------------|---------------------------------|
| 1     | `appsettings.json`               | Safe defaults, committed to git |
| 2     | `appsettings.{Environment}.json` | Per-environment defaults        |
| 3     | User secrets, Development only   | Local passwords and API keys    |
| 4     | Environment variables            | Deployment values and secrets   |
| 5     | Command-line arguments           | One-off overrides               |

``` json
{
  "Payments": {
    "BaseUrl": "https://sandbox.example.com",
    "TimeoutSeconds": 10
  },
  "ConnectionStrings": {
    "Default": "Host=localhost;Database=items;Username=items"
  }
}
```

------------------------------------------------------------------------

## 2. Keys and Environment Variables

Configuration keys are flat strings that use `:` for nesting. They are not case-sensitive.

| JSON path | Configuration key | Environment variable |
|----|----|----|
| `Payments.BaseUrl` | `Payments:BaseUrl` | `Payments__BaseUrl` |
| `ConnectionStrings.Default` | `ConnectionStrings:Default` | `ConnectionStrings__Default` |
| `Hosts[0]` | `Hosts:0` | `Hosts__0` |

The double underscore works on every operating system.

``` bash
export Payments__TimeoutSeconds=30
dotnet run
```

------------------------------------------------------------------------

## 3. Read a Value

``` csharp
var baseUrl = builder.Configuration["Payments:BaseUrl"];
var timeout = builder.Configuration.GetValue<int>("Payments:TimeoutSeconds");
var connection = builder.Configuration.GetConnectionString("Default");
```

Direct reads suit startup code. Use the options pattern everywhere else, because a mistyped key here silently returns `null` or a default value.

------------------------------------------------------------------------

## 4. The Options Pattern

Define a class for one group of settings, then bind and validate it at startup.

``` csharp
using System.ComponentModel.DataAnnotations;

public sealed class PaymentsOptions
{
    public const string Section = "Payments";

    [Required, Url]
    public required string BaseUrl { get; init; }

    [Range(1, 60)]
    public int TimeoutSeconds { get; init; } = 10;

    [Required]
    public required string ApiKey { get; init; }
}
```

``` csharp
builder.Services
    .AddOptions<PaymentsOptions>()
    .BindConfiguration(PaymentsOptions.Section)
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

`ValidateOnStart` stops the app at launch when a value is wrong. Without it, the error appears only on the first use.

Consume the options through DI:

``` csharp
using Microsoft.Extensions.Options;

public sealed class PaymentsClient(HttpClient http, IOptions<PaymentsOptions> options)
{
    private readonly PaymentsOptions _settings = options.Value;

    public Task<string> StatusAsync(CancellationToken ct) =>
        http.GetStringAsync($"{_settings.BaseUrl}/status", ct);
}
```

------------------------------------------------------------------------

## 5. Which Options Interface?

| Interface | Lifetime | Sees changes after startup? | Use it for |
|----|----|----|----|
| `IOptions<T>` | Singleton | No | Most settings |
| `IOptionsSnapshot<T>` | Scoped | Yes, on each request | Per-request reload |
| `IOptionsMonitor<T>` | Singleton | Yes, with change callbacks | Singletons and workers that need updates |

Start with `IOptions<T>`. A restart then applies any configuration change, which is easy to reason about.

------------------------------------------------------------------------

## 6. Secrets

Never commit a secret to `appsettings.json`.

For local development, use **user secrets**. The tool stores values in your home directory, outside the repository.

``` bash
dotnet user-secrets init
dotnet user-secrets set "Payments:ApiKey" "local-test-key"
dotnet user-secrets list
```

| Environment | Where the secret lives                           |
|-------------|--------------------------------------------------|
| Local       | `dotnet user-secrets`                            |
| CI          | GitHub Actions secrets, or OIDC for cloud access |
| Container   | Environment variables injected at deploy time    |
| AWS         | Secrets Manager or SSM Parameter Store           |

User secrets are not encrypted. They only keep values out of the source tree.

------------------------------------------------------------------------

## 7. Add a Custom Source

Add a source after the defaults when values come from another system. This key-per-file provider reads mounted Docker or Kubernetes secrets. Each file name is a key, and the file content is the value.

``` csharp
builder.Configuration.AddKeyPerFile("/run/secrets", optional: true);
```

Sources added later have higher precedence than the defaults.

------------------------------------------------------------------------

## 8. Compared With Django

| Django                  | ASP.NET Core                           |
|-------------------------|----------------------------------------|
| `settings.py`           | `appsettings.json` and options classes |
| `os.environ["KEY"]`     | `builder.Configuration["KEY"]`         |
| `django-environ`        | Built in                               |
| `settings.PAYMENTS_URL` | `IOptions<PaymentsOptions>`            |
| `manage.py check`       | `ValidateOnStart`                      |

------------------------------------------------------------------------

## 9. Common Mistakes

| Mistake | What goes wrong | Do this instead |
|----|----|----|
| Single underscore in an environment variable | The value is ignored | Use `__` for each nesting level |
| Missing `ValidateOnStart` | The app fails later, under traffic | Validate at startup |
| Secrets in `appsettings.Production.json` | Secrets enter git history | Use environment variables or a secret manager |
| Injecting `IConfiguration` everywhere | Keys are untyped and scattered | Use the options pattern |
| Injecting `IOptionsSnapshot` into a singleton | Captive dependency error | Use `IOptionsMonitor` |

------------------------------------------------------------------------

Back to top
