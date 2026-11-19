# Deployment

Shipping ASP.NET Core to production: publishing, Docker images, health checks, reverse proxies, migrations and CI/CD with GitHub Actions.

Author

Benedict Thekkel

A .NET app publishes to a self-contained folder or a container image. It needs no virtual environment on the server. This page takes one service from `dotnet publish` to a production deployment on Docker and AWS.

> **NOTE:**
>
> The code on this page has not been compiled or run yet. It targets .NET 11 and C# 15.

------------------------------------------------------------------------

## 1. Publish Options

| Option | Command | Result | Use it for |
|----|----|----|----|
| Framework-dependent | `dotnet publish -c Release` | Small output, needs the runtime | Containers using the ASP.NET image |
| Self-contained | `dotnet publish -c Release -r linux-x64 --self-contained` | Includes the runtime | Servers with no .NET installed |
| ReadyToRun | Add `-p:PublishReadyToRun=true` | Faster startup, larger files | Services that scale often |
| Native AOT | Add `<PublishAot>true</PublishAot>` | One native binary | Small services with compatible libraries |

Use framework-dependent output in a container for most APIs. EF Core and reflection-heavy libraries may not work fully with Native AOT, so test before you adopt it.

------------------------------------------------------------------------

## 2. Dockerfile

Use a multi-stage build. The final image then has no SDK or source code.

``` dockerfile
FROM mcr.microsoft.com/dotnet/sdk:11.0 AS build
WORKDIR /src

COPY ItemsApi/ItemsApi.csproj ItemsApi/
RUN dotnet restore ItemsApi/ItemsApi.csproj

COPY . .
RUN dotnet publish ItemsApi/ItemsApi.csproj -c Release -o /app --no-restore

FROM mcr.microsoft.com/dotnet/aspnet:11.0
WORKDIR /app
COPY --from=build /app .
USER $APP_UID
EXPOSE 8080
ENTRYPOINT ["dotnet", "ItemsApi.dll"]
```

Add a `.dockerignore`, so that local build output does not enter the image:

    **/bin
    **/obj
    **/.git
    **/appsettings.Development.json

| Practice | Reason |
|----|----|
| Copy the project file first | Docker reuses the restore layer until dependencies change |
| Use `USER $APP_UID` | The app does not run as root |
| Listen on port 8080 | The default for ASP.NET images, and it needs no root |
| Consider a chiseled image | Ubuntu Chiseled images have no shell or package manager, which reduces the attack surface |

The SDK can also build an image with no Dockerfile:

``` bash
dotnet publish -c Release /t:PublishContainer -p:ContainerRepository=items-api
```

------------------------------------------------------------------------

## 3. Health Checks

Separate **liveness** from **readiness**. Liveness asks whether the process works. Readiness asks whether it can serve traffic now.

``` bash
dotnet add package Microsoft.Extensions.Diagnostics.HealthChecks.EntityFrameworkCore
```

``` csharp
using Microsoft.AspNetCore.Diagnostics.HealthChecks;

builder.Services.AddHealthChecks()
    .AddDbContextCheck<AppDbContext>(tags: ["ready"]);

var app = builder.Build();

app.MapHealthChecks("/health/live", new HealthCheckOptions { Predicate = _ => false })
    .AllowAnonymous();

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready"),
}).AllowAnonymous();
```

The liveness endpoint runs no dependency checks. A database outage must not make the orchestrator restart healthy containers.

------------------------------------------------------------------------

## 4. Reverse Proxy

Run Nginx, Caddy or a load balancer in front of Kestrel for TLS. The app must then trust the forwarded headers, or it will see the proxy address and `http` as the scheme.

``` csharp
using Microsoft.AspNetCore.HttpOverrides;

builder.Services.Configure<ForwardedHeadersOptions>(options =>
{
    options.ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto;
    options.KnownIPNetworks.Add(System.Net.IPNetwork.Parse("10.0.0.0/8"));
});

var app = builder.Build();

app.UseForwardedHeaders();
```

List only your proxy network. Trusting every source allows a client to forge its IP address.

``` nginx
server {
    listen 443 ssl;
    server_name api.example.com;

    location / {
        proxy_pass http://items-api:8080;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
    }
}
```

The `Upgrade` headers allow WebSocket connections. Define `$connection_upgrade` with a `map` block in the `http` section.

------------------------------------------------------------------------

## 5. Docker Compose

``` yaml
services:
  api:
    build: .
    ports:
      - "8080:8080"
    environment:
      ASPNETCORE_ENVIRONMENT: Production
      ConnectionStrings__Default: Host=db;Database=items;Username=items;Password=${DB_PASSWORD}
      OTEL_EXPORTER_OTLP_ENDPOINT: http://otel-collector:4317
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped

  db:
    image: postgres:17-alpine
    environment:
      POSTGRES_DB: items
      POSTGRES_USER: items
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U items"]
      interval: 5s
      retries: 10

volumes:
  pgdata:
```

------------------------------------------------------------------------

## 6. Database Migrations

Run migrations once, as a separate step before the new version starts.

``` bash
dotnet ef migrations bundle --self-contained -r linux-x64 -o efbundle
./efbundle --connection "$CONNECTION_STRING"
```

A **migration bundle** is one executable that needs neither the SDK nor the source code.

Use the **expand and contract** pattern for zero-downtime changes:

1.  Add the new column as nullable, and deploy code that writes both columns.
2.  Backfill the existing rows.
3.  Deploy code that reads the new column.
4.  Remove the old column in a later release.

------------------------------------------------------------------------

## 7. Multiple Replicas

| Concern | What to do |
|----|----|
| Data Protection keys | Share them, or cookies fail between replicas |
| Sessions | Keep them in Redis or use stateless tokens |
| In-memory cache | Accept per-replica data or use `HybridCache` with Redis |
| SignalR | Add the Redis backplane and sticky sessions |
| Background loops | Run them in one worker or use a lock |
| Uploaded files | Store them in S3, not on the container disk |

``` bash
dotnet add package Microsoft.AspNetCore.DataProtection.StackExchangeRedis
```

``` csharp
using Microsoft.AspNetCore.DataProtection;
using StackExchange.Redis;

var redis = ConnectionMultiplexer.Connect(builder.Configuration.GetConnectionString("Redis")!);

builder.Services.AddDataProtection()
    .SetApplicationName("items-api")
    .PersistKeysToStackExchangeRedis(redis, "dataprotection-keys");
```

------------------------------------------------------------------------

## 8. Resource Limits

The .NET garbage collector reads container limits. By default, the heap can use 75% of the memory limit.

| Variable | Effect |
|----|----|
| `DOTNET_GCHeapHardLimitPercent` | Sets the maximum heap as a percentage, in hexadecimal |
| `DOTNET_gcServer` | `1` selects server GC, and `0` selects workstation GC |
| `DOTNET_TieredPGO` | `0` disables profile-guided optimisation |

Server GC is the default for ASP.NET Core apps with more than one CPU. It favours throughput and uses more memory. Try workstation GC for a small service with a tight memory limit, and measure the result.

------------------------------------------------------------------------

## 9. CI/CD With GitHub Actions

This workflow tests the code, then pushes an image to Amazon ECR with OIDC. It stores no long-lived AWS keys.

``` yaml
name: deploy

on:
  push:
    branches: [main]

permissions:
  contents: read
  id-token: write

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          global-json-file: global.json
      - run: dotnet test -c Release

  image:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_DEPLOY_ROLE_ARN }}
          aws-region: ap-southeast-2
      - id: ecr
        uses: aws-actions/amazon-ecr-login@v2
      - run: |
          IMAGE="${{ steps.ecr.outputs.registry }}/items-api:${{ github.sha }}"
          docker build -t "$IMAGE" .
          docker push "$IMAGE"
```

Tag images with the commit SHA, so that each deployment can be traced and rolled back.

------------------------------------------------------------------------

## 10. Production Checklist

| Check                    | Target                                        |
|--------------------------|-----------------------------------------------|
| `ASPNETCORE_ENVIRONMENT` | `Production`                                  |
| Secrets                  | Injected at deploy time, none in the image    |
| Container user           | Non-root                                      |
| Health checks            | Liveness and readiness are separate           |
| Migrations               | Run once per release                          |
| Forwarded headers        | Limited to the known proxy network            |
| Logs, traces and metrics | Exported to your observability stack          |
| Shutdown timeout         | Below the orchestrator grace period           |
| Security patches         | Rebuild images on each .NET servicing release |

------------------------------------------------------------------------

## 11. Compared With Django

| Django                          | ASP.NET Core                            |
|---------------------------------|-----------------------------------------|
| Gunicorn workers                | One Kestrel process                     |
| `collectstatic`                 | `dotnet publish` includes static assets |
| `manage.py migrate`             | Migration bundle                        |
| `SECRET_KEY`                    | Data Protection keys                    |
| `SECURE_PROXY_SSL_HEADER`       | `UseForwardedHeaders`                   |
| `requirements.txt` in the image | Compiled output in the image            |

------------------------------------------------------------------------

## 12. Common Mistakes

| Mistake | What goes wrong | Do this instead |
|----|----|----|
| Running as root | A breakout has more privileges | Use `USER $APP_UID` |
| Migrating at startup on each replica | Migrations race | Use a single deployment step |
| Missing forwarded headers | Wrong client IP and redirect loops | Configure them first in the pipeline |
| Data Protection keys in the container | Users are signed out on each deploy | Persist the keys |
| Liveness check that tests the database | A database outage restarts all containers | Keep liveness dependency-free |
| Using a floating image tag in production | Builds change without notice | Pin versions or digests |

------------------------------------------------------------------------

Back to top
