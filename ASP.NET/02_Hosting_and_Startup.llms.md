# Hosting and Startup

How an ASP.NET Core process starts: Program.cs, WebApplicationBuilder, the Kestrel server, environments and graceful shutdown.

Author

Benedict Thekkel

Every ASP.NET Core app starts from one file, `Program.cs`. It has two phases. The first phase registers services, and the second phase builds the app and declares how requests are handled. Most startup bugs come from putting code in the wrong phase.

> **NOTE:**
>
> The code on this page has not been compiled or run yet. It targets .NET 11 and C# 15.

------------------------------------------------------------------------

## 1. The Smallest App

``` bash
dotnet new web -n HostingDemo
cd HostingDemo
dotnet run
```

``` csharp
var builder = WebApplication.CreateBuilder(args);

// Phase 1: register services
builder.Services.AddProblemDetails();

var app = builder.Build();

// Phase 2: configure the request pipeline and endpoints
app.UseExceptionHandler();
app.MapGet("/", () => "Hello");

app.Run();
```

| Line | What it does |
|----|----|
| `CreateBuilder(args)` | Loads configuration, logging and the Kestrel defaults |
| `builder.Services` | The dependency injection registrations |
| `builder.Build()` | Freezes the registrations and creates the app |
| `app.Use...` | Adds middleware |
| `app.Map...` | Adds endpoints |
| `app.Run()` | Starts the server and blocks until shutdown |

You cannot add services after `Build()`. The service collection is read-only from that point.

------------------------------------------------------------------------

## 2. What the Builder Sets Up

`WebApplication.CreateBuilder` applies these defaults:

| Area | Default |
|----|----|
| Configuration | `appsettings.json`, the environment file, user secrets in Development, environment variables, command-line arguments |
| Logging | Console, debug and event source providers |
| Server | Kestrel |
| Service validation | Scope and build validation in Development |
| Middleware | Routing, authentication and authorization are added automatically when needed |

`WebApplication.CreateSlimBuilder` starts with fewer features. Use it for Native AOT (ahead-of-time compilation) apps.

------------------------------------------------------------------------

## 3. Kestrel

**Kestrel** is the cross-platform HTTP server inside your process. It supports HTTP/1.1, HTTP/2 and HTTP/3. There is no separate application server, so Kestrel replaces Gunicorn or Uvicorn.

Set the listening address with one of these options:

| Option               | Example                                   |
|----------------------|-------------------------------------------|
| Environment variable | `ASPNETCORE_URLS=http://+:8080`           |
| HTTP ports only      | `ASPNETCORE_HTTP_PORTS=8080`              |
| Command line         | `dotnet run --urls http://localhost:5050` |
| Code                 | `app.Run("http://localhost:5050")`        |

Change request limits in code when the defaults do not fit:

``` csharp
builder.WebHost.ConfigureKestrel(options =>
{
    options.Limits.MaxRequestBodySize = 10 * 1024 * 1024;
    options.Limits.RequestHeadersTimeout = TimeSpan.FromSeconds(15);
});
```

The default maximum request body size is about 30 MB.

------------------------------------------------------------------------

## 4. Environments

The `ASPNETCORE_ENVIRONMENT` variable selects the environment. The value is `Production` when the variable is not set.

``` csharp
if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();
}
```

`Properties/launchSettings.json` sets the environment and ports for `dotnet run` on your machine. It is a development file only, and it has no effect on a published app.

``` json
{
  "profiles": {
    "http": {
      "commandName": "Project",
      "applicationUrl": "http://localhost:5050",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    }
  }
}
```

------------------------------------------------------------------------

## 5. One Process, Many Threads

Django normally runs several worker processes. ASP.NET Core runs one process and schedules requests on a shared **thread pool**.

| Topic             | Gunicorn and Django | Kestrel and ASP.NET Core   |
|-------------------|---------------------|----------------------------|
| Workers           | Several processes   | One process                |
| CPU cores         | One worker per core | All cores from one process |
| In-process memory | Separate per worker | Shared by all requests     |
| Blocking call     | Holds a worker      | Holds a pool thread        |

Shared memory makes an in-process cache useful. It also means that shared mutable state must be thread-safe.

------------------------------------------------------------------------

## 6. Keep Program.cs Readable

Move each feature into an extension method when the file grows. The startup file then reads like a table of contents.

``` csharp
public static class ItemsFeature
{
    public static IServiceCollection AddItems(this IServiceCollection services)
    {
        services.AddScoped<ItemService>();
        return services;
    }

    public static IEndpointRouteBuilder MapItems(this IEndpointRouteBuilder routes)
    {
        var group = routes.MapGroup("/items");
        group.MapGet("/", (ItemService service, CancellationToken ct) => service.ListAsync(ct));
        return routes;
    }
}
```

``` csharp
builder.Services.AddItems();

var app = builder.Build();
app.MapItems();
app.Run();
```

------------------------------------------------------------------------

## 7. Startup and Shutdown Hooks

`IHostApplicationLifetime` exposes three events:

``` csharp
var lifetime = app.Services.GetRequiredService<IHostApplicationLifetime>();

lifetime.ApplicationStarted.Register(() => app.Logger.LogInformation("Started"));
lifetime.ApplicationStopping.Register(() => app.Logger.LogInformation("Draining requests"));
lifetime.ApplicationStopped.Register(() => app.Logger.LogInformation("Stopped"));
```

On `SIGTERM`, the host follows these steps:

1.  It stops accepting new connections.
2.  It lets running requests finish.
3.  It stops hosted services.
4.  It exits when the work completes or the shutdown timeout expires. The default timeout is 30 seconds.

Set the timeout below the grace period of your orchestrator:

``` csharp
builder.Services.Configure<HostOptions>(options =>
    options.ShutdownTimeout = TimeSpan.FromSeconds(20));
```

------------------------------------------------------------------------

## 8. Compared With Django

| Django                   | ASP.NET Core                              |
|--------------------------|-------------------------------------------|
| `settings.py`            | `appsettings.json` and `builder.Services` |
| `wsgi.py` or `asgi.py`   | `Program.cs`                              |
| `manage.py runserver`    | `dotnet watch`                            |
| Gunicorn or Uvicorn      | Kestrel                                   |
| `DJANGO_SETTINGS_MODULE` | `ASPNETCORE_ENVIRONMENT`                  |
| `AppConfig.ready()`      | `ApplicationStarted` or a hosted service  |

------------------------------------------------------------------------

## 9. Common Mistakes

| Mistake | What goes wrong | Do this instead |
|----|----|----|
| Registering a service after `Build()` | The app throws at startup | Register all services first |
| Relying on `launchSettings.json` in production | The file is not used | Set environment variables |
| Blocking startup on a slow remote call | The app fails health checks | Use a hosted service with retries |
| Exposing development tools in every environment | Internal data becomes public | Guard them with `IsDevelopment()` |
| Keeping unsynchronised static state | Requests corrupt each other | Use thread-safe types or a scoped service |

------------------------------------------------------------------------

Back to top
