# Caching and Performance

Making ASP.NET Core faster: measurement, HybridCache, output caching, Redis, compression, rate limiting and common bottlenecks.

Author

Benedict Thekkel

Measure first. Most slow APIs spend their time in the database or in a remote call, so a framework setting rarely fixes the real bottleneck. This page starts with measurement, then covers each cache layer and the controls that protect a service under load.

> **NOTE:**
>
> The code on this page has not been compiled or run yet. It targets .NET 11 and C# 15.

------------------------------------------------------------------------

## 1. Measure Before You Optimise

| Question                        | Tool                                    |
|---------------------------------|-----------------------------------------|
| How fast is the API under load? | `k6` or `wrk`                           |
| Which request is slow?          | OpenTelemetry traces in Grafana         |
| Which SQL is slow?              | EF Core command logs, `EXPLAIN ANALYZE` |
| Where does CPU time go?         | `dotnet-trace`                          |
| What fills memory?              | `dotnet-gcdump`                         |
| Live runtime counters           | `dotnet-counters`                       |

``` bash
dotnet tool install --global dotnet-counters
dotnet-counters monitor --name ItemsApi --counters System.Runtime,Microsoft.AspNetCore.Hosting
```

Set a target such as “p95 under 50 ms at 500 requests per second”. Compare the numbers before and after each change.

------------------------------------------------------------------------

## 2. Choose a Cache Layer

| Layer | Stores | Shared across replicas? | Best for |
|----|----|----|----|
| `IMemoryCache` | Objects in process | No | Small data on one instance |
| `IDistributedCache` | Bytes in Redis | Yes | Basic shared cache |
| `HybridCache` | Objects in process and bytes in Redis | Yes | Default choice for application data |
| Output cache | Whole HTTP responses | Optional, with Redis | Public or repeated GET responses |
| Response cache headers | Responses in browsers and CDNs | Not applicable | Static or public content |

------------------------------------------------------------------------

## 3. HybridCache

`HybridCache` combines a fast local cache with an optional shared cache. It also prevents a **cache stampede**: when many requests miss the same key, only one rebuilds the value.

``` bash
dotnet add package Microsoft.Extensions.Caching.Hybrid
dotnet add package Microsoft.Extensions.Caching.StackExchangeRedis
```

``` csharp
using Microsoft.Extensions.Caching.Hybrid;

builder.Services.AddStackExchangeRedisCache(options =>
    options.Configuration = builder.Configuration.GetConnectionString("Redis"));

builder.Services.AddHybridCache(options =>
{
    options.DefaultEntryOptions = new HybridCacheEntryOptions
    {
        Expiration = TimeSpan.FromMinutes(5),
        LocalCacheExpiration = TimeSpan.FromSeconds(30),
    };
});
```

``` csharp
public sealed class ItemReader(AppDbContext db, HybridCache cache)
{
    public ValueTask<ItemResponse?> GetAsync(int id, CancellationToken ct) =>
        cache.GetOrCreateAsync(
            $"item:{id}",
            async token => await db.Items
                .AsNoTracking()
                .Where(i => i.Id == id)
                .Select(i => new ItemResponse(i.Id, i.Name, i.Price))
                .FirstOrDefaultAsync(token),
            tags: ["items"],
            cancellationToken: ct);

    public async Task InvalidateAsync(int id, CancellationToken ct)
    {
        await cache.RemoveAsync($"item:{id}", ct);
    }
}
```

Remove all entries with one tag through `cache.RemoveByTagAsync("items", ct)`.

A short local expiry limits how long a replica can serve stale data after another replica changes it.

------------------------------------------------------------------------

## 4. Output Caching

Output caching stores complete responses on the server.

``` csharp
builder.Services.AddOutputCache(options =>
{
    options.AddPolicy("items", policy => policy
        .Expire(TimeSpan.FromMinutes(1))
        .SetVaryByQuery("page", "search")
        .Tag("items"));
});

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();
app.UseOutputCache();

app.MapGet("/items", ListItems).CacheOutput("items");

app.MapPost("/items", async (CreateItemRequest body, IOutputCacheStore store, CancellationToken ct) =>
{
    await store.EvictByTagAsync("items", ct);
    return TypedResults.NoContent();
});
```

By default, the output cache only stores `200` responses to `GET` and `HEAD` requests with no `Authorization` header or cookie. Keep this rule unless you vary the cache by user.

Add `Microsoft.AspNetCore.OutputCaching.StackExchangeRedis` and call `AddStackExchangeRedisOutputCache` to share the output cache across replicas.

------------------------------------------------------------------------

## 5. Response Compression

Let the reverse proxy compress responses when one exists. Otherwise, enable compression in the app:

``` csharp
builder.Services.AddResponseCompression(options => options.EnableForHttps = true);

var app = builder.Build();
app.UseResponseCompression();
```

Compression over HTTPS can expose secrets through attacks such as BREACH. Do not compress responses that mix secrets with user-controlled input.

------------------------------------------------------------------------

## 6. Rate Limiting

The rate limiter is built in. It protects the service from abuse and overload.

``` csharp
using System.Threading.RateLimiting;
using Microsoft.AspNetCore.RateLimiting;

builder.Services.AddRateLimiter(options =>
{
    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;

    options.AddPolicy("per-user", context =>
        RateLimitPartition.GetTokenBucketLimiter(
            context.User.FindFirst("sub")?.Value
                ?? context.Connection.RemoteIpAddress?.ToString()
                ?? "anonymous",
            _ => new TokenBucketRateLimiterOptions
            {
                TokenLimit = 100,
                TokensPerPeriod = 50,
                ReplenishmentPeriod = TimeSpan.FromSeconds(10),
                QueueLimit = 0,
            }));
});

var app = builder.Build();
app.UseRateLimiter();

app.MapGet("/items", ListItems).RequireRateLimiting("per-user");
```

| Algorithm | Behaviour |
|----|----|
| Fixed window | N requests per window. Simple, but allows bursts at the edge |
| Sliding window | Smooths the window edge |
| Token bucket | Allows controlled bursts |
| Concurrency | Limits requests in progress |

The limiter counts per process. Each replica keeps its own counters, so enforce strict global limits at the gateway or in Redis.

------------------------------------------------------------------------

## 7. Common Bottlenecks

| Symptom | Likely cause | Fix |
|----|----|----|
| Latency rises and CPU stays low | Thread pool starvation from blocking calls | Remove `.Result`, `.Wait()` and sync IO |
| Timeouts on database connections | Pool exhausted by long transactions | Shorten transactions and use `AddDbContextPool` |
| Memory grows steadily | Unbounded cache or static collection | Set a size limit and an expiry |
| Slow list endpoints | Too many columns or N+1 queries | Project with `Select` and add indexes |
| Slow first request | JIT compilation and cold caches | Warm up, or consider ReadyToRun |

``` csharp
builder.Services.AddDbContextPool<AppDbContext>(options =>
    options.UseNpgsql(connectionString));
```

`AddDbContextPool` reuses `DbContext` objects, which lowers allocations on busy services.

------------------------------------------------------------------------

## 8. Stream Large Results

Return `IAsyncEnumerable<T>` so that the response does not build the whole list in memory:

``` csharp
app.MapGet("/export", (AppDbContext db) =>
    db.Items.AsNoTracking()
        .Select(i => new ItemResponse(i.Id, i.Name, i.Price))
        .AsAsyncEnumerable());
```

------------------------------------------------------------------------

## 9. Compared With Django

| Django                 | ASP.NET Core                            |
|------------------------|-----------------------------------------|
| `cache.get_or_set`     | `HybridCache.GetOrCreateAsync`          |
| `cache_page`           | `CacheOutput`                           |
| `django-redis`         | `AddStackExchangeRedisCache`            |
| DRF throttling         | `AddRateLimiter`                        |
| `GZipMiddleware`       | `UseResponseCompression`                |
| `django-debug-toolbar` | Traces, EF Core logs, `dotnet-counters` |

------------------------------------------------------------------------

## 10. Common Mistakes

| Mistake | What goes wrong | Do this instead |
|----|----|----|
| Optimising without a measurement | Effort goes to the wrong code | Profile first |
| Caching per-user data under a shared key | Users see each other’s data | Include the user in the key |
| `IMemoryCache` with no size or expiry | Memory grows until the process dies | Set limits |
| Output cache before authorization | Protected data can be served | Add it after authorization |
| Rate limiting behind a proxy without forwarded headers | All users share one limit | Configure forwarded headers |

------------------------------------------------------------------------

Back to top
