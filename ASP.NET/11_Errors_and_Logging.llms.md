# Error Handling and Logging

Consistent API errors with ProblemDetails, global exception handlers, structured logging, and traces and metrics through OpenTelemetry.

Author

Benedict Thekkel

A production API needs two things when a request fails. The client needs a consistent error body with no internal details. The operator needs logs, traces and metrics that explain what happened. ASP.NET Core provides the foundations for both.

> **NOTE:**
>
> The code on this page has not been compiled or run yet. It targets .NET 11 and C# 15.

------------------------------------------------------------------------

## 1. Problem Details

**Problem Details** (RFC 9457) is a standard JSON format for HTTP errors. Enable it with these lines:

``` csharp
builder.Services.AddProblemDetails();

var app = builder.Build();

app.UseExceptionHandler();
app.UseStatusCodePages();
```

A failed request then returns this body:

``` json
{
  "type": "https://tools.ietf.org/html/rfc9110#section-15.6.1",
  "title": "An error occurred while processing your request.",
  "status": 500,
  "traceId": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01"
}
```

`UseStatusCodePages` adds the same format to empty error responses, such as a `404` with no body.

------------------------------------------------------------------------

## 2. Handle Domain Exceptions

Implement `IExceptionHandler` to map known exceptions to HTTP responses.

``` csharp
using Microsoft.AspNetCore.Diagnostics;
using Microsoft.AspNetCore.Mvc;

public sealed class NotFoundException(string message) : Exception(message);
public sealed class ConflictException(string message) : Exception(message);

public sealed class DomainExceptionHandler(IProblemDetailsService problems) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext context, Exception exception, CancellationToken ct)
    {
        var (status, title) = exception switch
        {
            NotFoundException => (StatusCodes.Status404NotFound, "Resource not found"),
            ConflictException => (StatusCodes.Status409Conflict, "Conflict"),
            _ => (0, ""),
        };

        if (status == 0)
        {
            return false;
        }

        context.Response.StatusCode = status;

        return await problems.TryWriteAsync(new ProblemDetailsContext
        {
            HttpContext = context,
            Exception = exception,
            ProblemDetails = new ProblemDetails
            {
                Status = status,
                Title = title,
                Detail = exception.Message,
            },
        });
    }
}
```

``` csharp
builder.Services.AddExceptionHandler<DomainExceptionHandler>();
builder.Services.AddProblemDetails();
```

Handlers run in registration order. Returning `false` passes the exception to the next handler, and the default response is a `500`.

------------------------------------------------------------------------

## 3. Add Fields to Every Error

``` csharp
builder.Services.AddProblemDetails(options =>
    options.CustomizeProblemDetails = context =>
    {
        context.ProblemDetails.Instance =
            $"{context.HttpContext.Request.Method} {context.HttpContext.Request.Path}";
    });
```

Do not include a stack trace in a production response. Log it on the server instead.

------------------------------------------------------------------------

## 4. Exceptions or Result Values

| Situation | Use |
|----|----|
| Expected outcome, such as “not found” in a handler | Return a typed result |
| A failure deep in the call stack | Throw a domain exception |
| An unexpected bug or outage | Let it reach the global handler |

Exceptions are slower than normal returns. Avoid them on a hot path that fails often.

------------------------------------------------------------------------

## 5. Structured Logging

`ILogger<T>` writes **structured** logs. Named placeholders become searchable fields in Loki or another log store.

``` csharp
public sealed class OrderService(ILogger<OrderService> logger)
{
    public void Place(int orderId, decimal total)
    {
        logger.LogInformation("Placed order {OrderId} with total {Total}", orderId, total);
    }
}
```

| Level            | Use it for                     |
|------------------|--------------------------------|
| `Trace`, `Debug` | Development detail             |
| `Information`    | Normal business events         |
| `Warning`        | Unusual but handled conditions |
| `Error`          | A failed operation             |
| `Critical`       | The service cannot continue    |

Set levels per namespace in `appsettings.json`:

``` json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  }
}
```

Do not use string interpolation in a log call. `$"Order {id}"` creates one unique message and loses the `OrderId` field.

------------------------------------------------------------------------

## 6. High-Performance Logging

On a hot path, generate the logging code at compile time:

``` csharp
public static partial class Log
{
    [LoggerMessage(Level = LogLevel.Information, Message = "Cache miss for {Key}")]
    public static partial void CacheMiss(ILogger logger, string key);
}

Log.CacheMiss(logger, key);
```

------------------------------------------------------------------------

## 7. JSON Logs and Request Logs

The built-in JSON console formatter is enough for containers:

``` csharp
using Microsoft.AspNetCore.HttpLogging;

builder.Logging.ClearProviders();
builder.Logging.AddJsonConsole();

builder.Services.AddHttpLogging(options =>
{
    options.LoggingFields = HttpLoggingFields.RequestPropertiesAndHeaders
        | HttpLoggingFields.ResponseStatusCode
        | HttpLoggingFields.Duration;
});

var app = builder.Build();
app.UseHttpLogging();
```

**Serilog** is a popular alternative with many output targets. Add it when you need its sinks or enrichers.

------------------------------------------------------------------------

## 8. OpenTelemetry

**OpenTelemetry** is the vendor-neutral standard for logs, traces and metrics. It connects to Grafana, Prometheus and Loki through an OpenTelemetry Collector.

``` bash
dotnet add package OpenTelemetry.Extensions.Hosting
dotnet add package OpenTelemetry.Instrumentation.AspNetCore
dotnet add package OpenTelemetry.Instrumentation.Http
dotnet add package OpenTelemetry.Exporter.OpenTelemetryProtocol
```

``` csharp
using OpenTelemetry;
using OpenTelemetry.Metrics;
using OpenTelemetry.Resources;
using OpenTelemetry.Trace;

builder.Logging.AddOpenTelemetry(logging =>
{
    logging.IncludeFormattedMessage = true;
    logging.IncludeScopes = true;
});

builder.Services.AddOpenTelemetry()
    .ConfigureResource(resource => resource.AddService("items-api"))
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddSource("ItemsApi"))
    .WithMetrics(metrics => metrics
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddRuntimeInstrumentation()
        .AddMeter("ItemsApi"))
    .UseOtlpExporter();
```

Set the collector address with the standard environment variable:

``` bash
export OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
```

`AddRuntimeInstrumentation` needs the `OpenTelemetry.Instrumentation.Runtime` package.

------------------------------------------------------------------------

## 9. Custom Traces and Metrics

``` csharp
using System.Diagnostics;
using System.Diagnostics.Metrics;

public sealed class CheckoutTelemetry
{
    public static readonly ActivitySource Source = new("ItemsApi");
    private static readonly Meter Meter = new("ItemsApi");

    public static readonly Counter<long> Orders =
        Meter.CreateCounter<long>("orders.placed", description: "Orders placed");
}
```

Record a span and a metric where the work happens:

``` csharp
using var activity = CheckoutTelemetry.Source.StartActivity("checkout");
activity?.SetTag("order.id", orderId);

CheckoutTelemetry.Orders.Add(1);
```

An **Activity** is the .NET name for a trace span.

------------------------------------------------------------------------

## 10. Compared With Django

| Django | ASP.NET Core |
|----|----|
| DRF `EXCEPTION_HANDLER` | `IExceptionHandler` |
| `Http404` | A domain exception or `TypedResults.NotFound()` |
| `logging.getLogger(__name__)` | `ILogger<T>` |
| `LOGGING` setting | `Logging` section in `appsettings.json` |
| `structlog` | Built-in structured logging |
| `django-prometheus` | OpenTelemetry metrics |

------------------------------------------------------------------------

## 11. Common Mistakes

| Mistake | What goes wrong | Do this instead |
|----|----|----|
| Logging with `$"..."` | Fields are lost and cardinality grows | Use message templates |
| Logging and rethrowing at each layer | One failure appears many times | Log once at the boundary |
| Logging passwords or tokens | Secrets reach the log store | Filter sensitive fields |
| Returning `exception.Message` for unknown errors | Internal details leak | Expose messages only for domain exceptions |
| Using user IDs as metric labels | Metric storage grows without limit | Keep labels low-cardinality |
| Empty `catch` blocks | Failures disappear | Handle, log or rethrow |

------------------------------------------------------------------------

Back to top
