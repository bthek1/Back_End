# Web APIs

Building HTTP APIs with Minimal APIs and controllers: typed results, endpoint organisation, OpenAPI, pagination and versioning.

Author

Benedict Thekkel

ASP.NET Core offers two ways to write an API. **Minimal APIs** map a route straight to a function. **Controllers** group actions into classes, like DRF views. Use Minimal APIs for new services, and keep controllers in codebases that already use them.

> **NOTE:**
>
> The code on this page has not been compiled or run yet. It targets .NET 11 and C# 15.

------------------------------------------------------------------------

## 1. Minimal APIs or Controllers

| Topic | Minimal APIs | Controllers |
|----|----|----|
| Code style | Functions and route groups | Classes with attributes |
| Startup cost | Lower | Higher |
| Native AOT support | Supported | Not supported |
| Filters | Endpoint filters | Action filters |
| Validation | `AddValidation()` | Automatic with `[ApiController]` |
| Best fit | New services | Large existing MVC codebases |

------------------------------------------------------------------------

## 2. A Complete Resource

This feature keeps routes in one static class. The handlers are thin and delegate to a service.

``` csharp
using Microsoft.AspNetCore.Http.HttpResults;

public record ItemResponse(int Id, string Name, decimal Price);
public record CreateItemRequest(string Name, decimal Price);

public static class ItemEndpoints
{
    public static RouteGroupBuilder MapItems(this IEndpointRouteBuilder routes)
    {
        var group = routes.MapGroup("/api/items").WithTags("Items");

        group.MapGet("/", List);
        group.MapGet("/{id:int}", Get).WithName("GetItem");
        group.MapPost("/", Create);
        group.MapDelete("/{id:int}", Delete);

        return group;
    }

    private static async Task<Ok<IReadOnlyList<ItemResponse>>> List(
        ItemService service, CancellationToken ct) =>
        TypedResults.Ok(await service.ListAsync(ct));

    private static async Task<Results<Ok<ItemResponse>, NotFound>> Get(
        int id, ItemService service, CancellationToken ct) =>
        await service.FindAsync(id, ct) is { } item
            ? TypedResults.Ok(item)
            : TypedResults.NotFound();

    private static async Task<CreatedAtRoute<ItemResponse>> Create(
        CreateItemRequest body, ItemService service, CancellationToken ct)
    {
        var item = await service.CreateAsync(body, ct);
        return TypedResults.CreatedAtRoute(item, "GetItem", new { id = item.Id });
    }

    private static async Task<Results<NoContent, NotFound>> Delete(
        int id, ItemService service, CancellationToken ct) =>
        await service.DeleteAsync(id, ct)
            ? TypedResults.NoContent()
            : TypedResults.NotFound();
}
```

------------------------------------------------------------------------

## 3. Typed Results

`TypedResults` returns a concrete type. `Results<A, B>` lists each possible response. The compiler checks each return statement, and OpenAPI reads the same types to document the responses.

| Status | Helper                                   |
|--------|------------------------------------------|
| 200    | `TypedResults.Ok(value)`                 |
| 201    | `TypedResults.Created(url, value)`       |
| 202    | `TypedResults.Accepted(url, value)`      |
| 204    | `TypedResults.NoContent()`               |
| 400    | `TypedResults.ValidationProblem(errors)` |
| 401    | `TypedResults.Unauthorized()`            |
| 403    | `TypedResults.Forbid()`                  |
| 404    | `TypedResults.NotFound()`                |
| 409    | `TypedResults.Conflict(value)`           |

------------------------------------------------------------------------

## 4. The Same API as a Controller

Learn to read this form, because many existing .NET codebases use it.

``` csharp
using Microsoft.AspNetCore.Mvc;

[ApiController]
[Route("api/items")]
public sealed class ItemsController(ItemService service) : ControllerBase
{
    [HttpGet("{id:int}", Name = "GetItem")]
    [ProducesResponseType<ItemResponse>(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<ItemResponse>> Get(int id, CancellationToken ct)
    {
        var item = await service.FindAsync(id, ct);
        return item is null ? NotFound() : Ok(item);
    }
}
```

``` csharp
builder.Services.AddControllers();

var app = builder.Build();
app.MapControllers();
```

------------------------------------------------------------------------

## 5. OpenAPI

ASP.NET Core generates an OpenAPI document. Add a viewer separately, because the framework does not include one.

``` bash
dotnet add package Microsoft.AspNetCore.OpenApi
dotnet add package Scalar.AspNetCore
```

``` csharp
using Scalar.AspNetCore;

builder.Services.AddOpenApi();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();                // /openapi/v1.json
    app.MapScalarApiReference();     // /scalar
}
```

Add descriptions to each endpoint:

``` csharp
group.MapGet("/{id:int}", Get)
    .WithName("GetItem")
    .WithSummary("Get one item")
    .WithDescription("Returns 404 when the item does not exist.");
```

Generate your React client from this document with a tool such as `openapi-typescript`.

------------------------------------------------------------------------

## 6. JSON

`System.Text.Json` is the default serializer. It uses camelCase property names on the web.

``` csharp
using System.Text.Json.Serialization;

builder.Services.ConfigureHttpJsonOptions(options =>
{
    options.SerializerOptions.Converters.Add(new JsonStringEnumConverter());
    options.SerializerOptions.DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull;
});
```

------------------------------------------------------------------------

## 7. Pagination and Filtering

Bind query parameters into one record with `[AsParameters]`. Clamp the page size on the server.

``` csharp
public record ItemQuery(string? Search, int Page = 1, int PageSize = 20)
{
    public int Skip => (Math.Max(Page, 1) - 1) * Take;
    public int Take => Math.Clamp(PageSize, 1, 100);
}

public record PagedResponse<T>(IReadOnlyList<T> Items, int Total, int Page, int PageSize);
```

``` csharp
group.MapGet("/", async ([AsParameters] ItemQuery query, AppDbContext db, CancellationToken ct) =>
{
    var items = db.Items.AsNoTracking();

    if (!string.IsNullOrWhiteSpace(query.Search))
    {
        items = items.Where(i => i.Name.Contains(query.Search));
    }

    var total = await items.CountAsync(ct);
    var page = await items
        .OrderBy(i => i.Id)
        .Skip(query.Skip)
        .Take(query.Take)
        .Select(i => new ItemResponse(i.Id, i.Name, i.Price))
        .ToListAsync(ct);

    return TypedResults.Ok(new PagedResponse<ItemResponse>(page, total, query.Page, query.Take));
});
```

Always sort before `Skip` and `Take`. Otherwise, rows can repeat or disappear between pages.

------------------------------------------------------------------------

## 8. CORS for a React Front End

``` csharp
builder.Services.AddCors(options =>
    options.AddDefaultPolicy(policy => policy
        .WithOrigins("http://localhost:5173")
        .AllowAnyHeader()
        .AllowAnyMethod()
        .AllowCredentials()));

var app = builder.Build();
app.UseCors();
```

You cannot combine `AllowCredentials` with `AllowAnyOrigin`. List each trusted origin.

------------------------------------------------------------------------

## 9. API Versioning

Version in the URL. It is visible in logs and simple for clients. `Asp.Versioning.Http` adds version metadata, reporting and deprecation headers.

``` bash
dotnet add package Asp.Versioning.Http
```

``` csharp
using Asp.Versioning;

builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1);
    options.ReportApiVersions = true;
    options.ApiVersionReader = new UrlSegmentApiVersionReader();
});

var versions = app.NewApiVersionSet()
    .HasDeprecatedApiVersion(new ApiVersion(1))
    .HasApiVersion(new ApiVersion(2))
    .Build();

var v = app.MapGroup("/api/v{version:apiVersion}").WithApiVersionSet(versions);

v.MapGet("/items", ListV1).MapToApiVersion(new ApiVersion(1));
v.MapGet("/items", ListV2).MapToApiVersion(new ApiVersion(2));
```

------------------------------------------------------------------------

## 10. Compared With DRF

| DRF                          | ASP.NET Core                                |
|------------------------------|---------------------------------------------|
| `APIView` or function view   | Minimal API handler                         |
| `ModelViewSet`               | No built-in equivalent, write each endpoint |
| `DefaultRouter`              | `MapGroup`                                  |
| `Serializer`                 | Request and response records                |
| `Response(data, status=201)` | `TypedResults.Created(url, data)`           |
| `drf-spectacular`            | `Microsoft.AspNetCore.OpenApi`              |
| `PageNumberPagination`       | A small query record                        |

------------------------------------------------------------------------

## 11. Common Mistakes

| Mistake | What goes wrong | Do this instead |
|----|----|----|
| Returning EF Core entities | Internal fields leak and JSON can loop | Return response records |
| Using `Results` instead of `TypedResults` | OpenAPI loses the response types | Use `TypedResults` |
| Not passing `CancellationToken` | Abandoned requests keep running | Add it to each async handler |
| Unbounded list endpoints | One request can load the table | Clamp the page size |
| `Skip` and `Take` with no sort | Pages are unstable | Add `OrderBy` |

------------------------------------------------------------------------

Back to top
