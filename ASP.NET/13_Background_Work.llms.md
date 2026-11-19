# Background Work

Running work outside the request: hosted services, in-process queues, Hangfire, Quartz.NET and the outbox pattern.

Author

Benedict Thekkel

Some work must not hold up an HTTP response, such as sending email or generating a report. Django projects normally use Celery for this. ASP.NET Core can run background work inside the web process, and it can use a durable job library when jobs must survive a restart.

> **NOTE:**
>
> The code on this page has not been compiled or run yet. It targets .NET 11 and C# 15.

------------------------------------------------------------------------

## 1. Choose a Tool

| Need | Use | Survives a restart? |
|----|----|----|
| A loop or timer inside the app | `BackgroundService` | No |
| Short fire-and-forget work | `Channel<T>` with a `BackgroundService` | No |
| Durable jobs with retries and a dashboard | Hangfire | Yes |
| Complex schedules and calendars | Quartz.NET | Yes, with a persistent store |
| Messages between services | RabbitMQ or SQS with a worker | Yes |

Start with the simplest row that meets the requirement. If losing a job is not acceptable, use a durable option.

------------------------------------------------------------------------

## 2. BackgroundService

A **hosted service** starts and stops with the application.

``` csharp
public sealed class CleanupWorker(
    IServiceScopeFactory scopes,
    ILogger<CleanupWorker> logger) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromMinutes(10));

        while (await timer.WaitForNextTickAsync(stoppingToken))
        {
            try
            {
                await using var scope = scopes.CreateAsyncScope();
                var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
                var cutoff = DateTimeOffset.UtcNow.AddDays(-30);

                var deleted = await db.Sessions
                    .Where(s => s.ExpiresAt < cutoff)
                    .ExecuteDeleteAsync(stoppingToken);

                logger.LogInformation("Deleted {Count} expired sessions", deleted);
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                logger.LogError(ex, "Cleanup failed");
            }
        }
    }
}
```

``` csharp
builder.Services.AddHostedService<CleanupWorker>();
```

Three rules apply:

1.  Create a new DI scope for each run, because a hosted service is a singleton.
2.  Catch exceptions inside the loop. By default, an unhandled exception in a `BackgroundService` stops the whole host.
3.  Pass `stoppingToken` to each async call, so that shutdown is quick.

------------------------------------------------------------------------

## 3. An In-Process Queue

`Channel<T>` is a fast, thread-safe queue. A bounded channel applies backpressure when the worker falls behind.

``` csharp
using System.Threading.Channels;

public record EmailJob(string To, string Subject);

public sealed class EmailQueue
{
    private readonly Channel<EmailJob> _channel = Channel.CreateBounded<EmailJob>(
        new BoundedChannelOptions(500) { FullMode = BoundedChannelFullMode.Wait });

    public ValueTask EnqueueAsync(EmailJob job, CancellationToken ct) =>
        _channel.Writer.WriteAsync(job, ct);

    public IAsyncEnumerable<EmailJob> ReadAllAsync(CancellationToken ct) =>
        _channel.Reader.ReadAllAsync(ct);
}

public sealed class EmailWorker(EmailQueue queue, IEmailSender sender, ILogger<EmailWorker> logger)
    : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await foreach (var job in queue.ReadAllAsync(stoppingToken))
        {
            try
            {
                await sender.SendAsync(job.To, job.Subject, stoppingToken);
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                logger.LogError(ex, "Email to {To} failed", job.To);
            }
        }
    }
}
```

``` csharp
builder.Services.AddSingleton<EmailQueue>();
builder.Services.AddHostedService<EmailWorker>();

app.MapPost("/invite", async (EmailJob job, EmailQueue queue, CancellationToken ct) =>
{
    await queue.EnqueueAsync(job, ct);
    return TypedResults.Accepted("/invite");
});
```

Jobs in this queue exist only in memory. A crash or deployment loses them.

------------------------------------------------------------------------

## 4. Do Not Use Task.Run in a Handler

``` csharp
// Wrong: the scoped DbContext is disposed while the task still runs
app.MapPost("/report", (AppDbContext db) =>
{
    _ = Task.Run(() => BuildReportAsync(db));
    return TypedResults.Accepted("/report");
});
```

The request scope ends when the response finishes. Nothing tracks the task, and any exception disappears. Use a queue or a durable job.

------------------------------------------------------------------------

## 5. Hangfire

**Hangfire** stores jobs in your database and adds retries, scheduling and a dashboard. It is the closest match to Celery, and it does not need a separate broker.

``` bash
dotnet add package Hangfire.AspNetCore
dotnet add package Hangfire.PostgreSql
```

``` csharp
using Hangfire;
using Hangfire.PostgreSql;

var connection = builder.Configuration.GetConnectionString("Default");

builder.Services.AddHangfire(config => config
    .UsePostgreSqlStorage(options => options.UseNpgsqlConnection(connection)));

builder.Services.AddHangfireServer(options => options.WorkerCount = 4);
builder.Services.AddScoped<ReportJobs>();

var app = builder.Build();
```

Write each job as a public method on a DI service:

``` csharp
public sealed class ReportJobs(AppDbContext db, ILogger<ReportJobs> logger)
{
    [AutomaticRetry(Attempts = 5)]
    public async Task BuildAsync(int reportId, CancellationToken ct)
    {
        logger.LogInformation("Building report {ReportId}", reportId);
        await db.SaveChangesAsync(ct);
    }
}
```

Enqueue, delay and schedule jobs:

``` csharp
app.MapPost("/reports/{id:int}", (int id, IBackgroundJobClient jobs) =>
{
    var jobId = jobs.Enqueue<ReportJobs>(j => j.BuildAsync(id, CancellationToken.None));
    return TypedResults.Accepted($"/jobs/{jobId}");
});

BackgroundJob.Schedule<ReportJobs>(
    j => j.BuildAsync(1, CancellationToken.None),
    TimeSpan.FromHours(1));

RecurringJob.AddOrUpdate<ReportJobs>(
    "nightly-report",
    j => j.BuildAsync(1, CancellationToken.None),
    Cron.Daily(2));
```

Hangfire replaces the `CancellationToken.None` placeholder with its own shutdown token when the job runs.

The dashboard allows only local requests by default. Add an authorization filter before you expose it.

``` csharp
app.UseHangfireDashboard("/hangfire");
```

------------------------------------------------------------------------

## 6. Write Idempotent Jobs

Durable job systems run a job **at least once**. A worker can crash after it finishes the work and before it records success. An **idempotent** job gives the same result when it runs twice.

``` csharp
public async Task ChargeAsync(int orderId, CancellationToken ct)
{
    var order = await db.Orders.SingleAsync(o => o.Id == orderId, ct);
    if (order.PaidAt is not null)
    {
        return;
    }

    await payments.ChargeAsync(order.Total, idempotencyKey: $"order-{order.Id}", ct);
    order.PaidAt = DateTimeOffset.UtcNow;
    await db.SaveChangesAsync(ct);
}
```

Pass IDs to a job, not whole objects. The job then reads the current data when it runs.

------------------------------------------------------------------------

## 7. The Outbox Pattern

A request that saves a row and then enqueues a message can fail between the two steps. The **outbox pattern** saves the message in the same database transaction. A worker publishes it afterwards.

``` csharp
db.Orders.Add(order);
db.OutboxMessages.Add(new OutboxMessage
{
    Type = "OrderPlaced",
    Payload = JsonSerializer.Serialize(new { order.Id }),
});

await db.SaveChangesAsync(ct);
```

Hangfire with the same PostgreSQL database gives a similar guarantee only if the enqueue shares the transaction. An outbox table makes the guarantee explicit.

------------------------------------------------------------------------

## 8. Several Replicas

A `BackgroundService` runs in **every** replica. Three replicas run the cleanup loop three times.

| Option | How it works |
|----|----|
| Hangfire recurring job | One server runs each occurrence |
| Separate worker deployment | Run one replica of a worker project |
| PostgreSQL advisory lock | Only the replica that holds the lock runs the work |

Use the Worker template for a separate process:

``` bash
dotnet new worker -n ItemsWorker
```

------------------------------------------------------------------------

## 9. Compared With Celery

| Celery | ASP.NET Core |
|----|----|
| `@shared_task` | Public method on a DI service |
| `task.delay(id)` | `jobs.Enqueue<T>(j => j.Run(id, CancellationToken.None))` |
| `apply_async(countdown=60)` | `BackgroundJob.Schedule` |
| Celery beat | `RecurringJob.AddOrUpdate` |
| Flower | Hangfire dashboard |
| Redis or RabbitMQ broker | PostgreSQL storage |
| `autoretry_for` | `[AutomaticRetry]` |

------------------------------------------------------------------------

## 10. Common Mistakes

| Mistake | What goes wrong | Do this instead |
|----|----|----|
| Injecting `DbContext` into a hosted service | Captive dependency | Create a scope per run |
| Unhandled exception in `ExecuteAsync` | The host stops | Catch inside the loop |
| `Task.Run` from a handler | Lost work and disposed services | Use a queue or Hangfire |
| Unbounded queue | Memory grows under load | Use a bounded channel |
| Non-idempotent durable jobs | Retries repeat side effects | Add an idempotency check |
| Timer loop on every replica | Work runs several times | Use one worker or a lock |

------------------------------------------------------------------------

Back to top
