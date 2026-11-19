# Model Binding and Validation

How ASP.NET Core turns requests into typed parameters, and how to validate them with data annotations or FluentValidation.

Author

Benedict Thekkel

**Model binding** reads route values, query strings, headers and JSON into typed handler parameters. **Validation** then checks that the values follow your rules. DRF serializers do both jobs, while ASP.NET Core separates them.

> **NOTE:**
>
> The code on this page has not been compiled or run yet. It targets .NET 11 and C# 15.

------------------------------------------------------------------------

## 1. Binding Sources

Minimal APIs infer the source of each parameter from its type and name:

| Parameter                                             | Inferred source |
|-------------------------------------------------------|-----------------|
| Name matches a route value                            | Route           |
| Simple type, such as `int` or `string`                | Query string    |
| Registered in DI                                      | Services        |
| `HttpContext`, `CancellationToken`, `ClaimsPrincipal` | Framework       |
| `IFormFile`                                           | Form            |
| Other complex type on `POST`, `PUT` or `PATCH`        | JSON body       |

``` csharp
app.MapPut("/items/{id:int}", (
    int id,                          // route
    bool notify,                     // query string
    UpdateItem body,                 // JSON body
    ItemService service,             // DI
    ClaimsPrincipal user,            // framework
    CancellationToken ct) => TypedResults.NoContent());
```

------------------------------------------------------------------------

## 2. Explicit Attributes

State the source when the inference is unclear.

``` csharp
using Microsoft.AspNetCore.Mvc;

app.MapGet("/search", (
    [FromQuery(Name = "q")] string term,
    [FromHeader(Name = "X-Tenant")] string tenant,
    [FromServices] SearchService service) => service.Find(tenant, term));
```

| Attribute        | Source                                     |
|------------------|--------------------------------------------|
| `[FromRoute]`    | Route value                                |
| `[FromQuery]`    | Query string                               |
| `[FromHeader]`   | Request header                             |
| `[FromBody]`     | JSON body                                  |
| `[FromForm]`     | Form field                                 |
| `[FromServices]` | DI container                               |
| `[AsParameters]` | Each property of a record binds separately |

A handler can read only one body parameter.

------------------------------------------------------------------------

## 3. Optional and Repeated Values

Nullability decides whether a parameter is required. A missing required value returns `400`.

``` csharp
app.MapGet("/items", (
    string? search,                // optional
    int page = 1,                  // optional with a default
    int[]? ids = null) =>          // ?ids=1&ids=2
    TypedResults.Ok(new { search, page, ids }));
```

------------------------------------------------------------------------

## 4. Request Records

Use a dedicated record for each write operation. A client then cannot set a field that the record does not contain, such as `IsAdmin` or `Id`.

``` csharp
using System.ComponentModel.DataAnnotations;

public record CreateItemRequest(
    [Required, StringLength(100, MinimumLength = 2)] string Name,
    [Range(0.01, 100000)] decimal Price,
    [EmailAddress] string? ContactEmail);
```

This protection replaces the `fields = [...]` list on a DRF serializer.

------------------------------------------------------------------------

## 5. Built-in Validation

Minimal APIs validate data annotations after you add one service:

``` csharp
builder.Services.AddProblemDetails();
builder.Services.AddValidation();

var app = builder.Build();

app.MapPost("/items", (CreateItemRequest body) => TypedResults.Created("/items/1", body));
```

Invalid input returns `400` with a problem details body:

``` json
{
  "title": "One or more validation errors occurred.",
  "status": 400,
  "errors": {
    "name": ["The field Name must be a string with a minimum length of 2 and a maximum length of 100."]
  }
}
```

Use `.DisableValidation()` on an endpoint that must skip this step. Controllers with `[ApiController]` validate automatically.

------------------------------------------------------------------------

## 6. Rules Across Several Fields

Implement `IValidatableObject` for a rule that needs more than one field:

``` csharp
public record DateRange(DateOnly From, DateOnly To) : IValidatableObject
{
    public IEnumerable<ValidationResult> Validate(ValidationContext context)
    {
        if (To < From)
        {
            yield return new ValidationResult("To must not be before From.", [nameof(To)]);
        }
    }
}
```

------------------------------------------------------------------------

## 7. FluentValidation

Use **FluentValidation** when rules are complex or need a database call. The rules live in a class of their own.

``` bash
dotnet add package FluentValidation.DependencyInjectionExtensions
```

``` csharp
using FluentValidation;

public sealed class CreateItemValidator : AbstractValidator<CreateItemRequest>
{
    public CreateItemValidator(AppDbContext db)
    {
        RuleFor(x => x.Name)
            .NotEmpty()
            .MaximumLength(100)
            .MustAsync(async (name, ct) => !await db.Items.AnyAsync(i => i.Name == name, ct))
            .WithMessage("An item with this name already exists.");

        RuleFor(x => x.Price).GreaterThan(0);
    }
}
```

Apply it with one reusable endpoint filter:

``` csharp
public sealed class ValidationFilter<T>(IValidator<T> validator) : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context, EndpointFilterDelegate next)
    {
        var body = context.Arguments.OfType<T>().First();
        var result = await validator.ValidateAsync(body, context.HttpContext.RequestAborted);

        return result.IsValid
            ? await next(context)
            : TypedResults.ValidationProblem(result.ToDictionary());
    }
}
```

``` csharp
builder.Services.AddValidatorsFromAssemblyContaining<CreateItemValidator>(ServiceLifetime.Scoped);

items.MapPost("/", CreateItem).AddEndpointFilter<ValidationFilter<CreateItemRequest>>();
```

A uniqueness check in a validator can still race with another request. Keep a unique index in the database as the final guarantee.

------------------------------------------------------------------------

## 8. Which Validator?

| Need | Use |
|----|----|
| Required, length, range, format | Data annotations |
| Conditional or multi-field rules | FluentValidation |
| Rules that query the database | FluentValidation, plus a database constraint |
| A business rule tied to current state | The service layer |

------------------------------------------------------------------------

## 9. Custom Binding

Add a static `TryParse` method to bind a custom type from a route or query value:

``` csharp
public readonly record struct Sku(string Value)
{
    public static bool TryParse(string? input, out Sku sku)
    {
        sku = new Sku(input?.Trim().ToUpperInvariant() ?? "");
        return sku.Value.Length == 8;
    }
}

app.MapGet("/products/{sku}", (Sku sku) => TypedResults.Ok(sku.Value));
```

------------------------------------------------------------------------

## 10. File Uploads

``` csharp
app.MapPost("/upload", async (IFormFile file, CancellationToken ct) =>
{
    if (file.Length is 0 or > 5_000_000)
    {
        return Results.BadRequest("File must be between 1 byte and 5 MB.");
    }

    var path = Path.Combine(Path.GetTempPath(), Path.GetRandomFileName());
    await using var stream = File.Create(path);
    await file.CopyToAsync(stream, ct);
    return Results.Ok(new { file.Length });
}).DisableAntiforgery();
```

Disable antiforgery only for endpoints that use bearer tokens. Keep it for cookie-authenticated forms. Never use the client’s file name as a storage path.

------------------------------------------------------------------------

## 11. Compared With DRF

| DRF | ASP.NET Core |
|----|----|
| `Serializer` fields | Request record |
| `serializer.is_valid(raise_exception=True)` | `AddValidation()` or a filter |
| `validate_name` | `RuleFor(x => x.Name)` |
| `validate` | `IValidatableObject` or FluentValidation |
| `UniqueValidator` | `MustAsync` and a unique index |
| `read_only_fields` | Omit the field from the request record |

------------------------------------------------------------------------

## 12. Common Mistakes

| Mistake | What goes wrong | Do this instead |
|----|----|----|
| Binding directly to an EF Core entity | Over-posting changes protected fields | Use a request record |
| Adding `[property: Required]` to a record parameter | MVC rejects validation metadata on the property | Put the attribute on the parameter |
| Two body parameters | The app throws at startup | Wrap them in one record |
| Trusting validation alone for uniqueness | Concurrent requests create duplicates | Add a unique index |
| Using `string` for every ID and date | Bad values pass through | Use `int`, `Guid` and `DateOnly` |

------------------------------------------------------------------------

Back to top
