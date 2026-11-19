# Routing

Matching requests to endpoints: route templates, constraints, groups, precedence and URL generation.

Author

Benedict Thekkel

**Routing** selects an endpoint from the request path and HTTP method. The route table is built when the app starts. Templates use `{name}` placeholders, and constraints control which values match.

> **NOTE:**
>
> The code on this page has not been compiled or run yet. It targets .NET 11 and C# 15.

------------------------------------------------------------------------

## 1. Basic Routes

``` csharp
app.MapGet("/items", ListItems);
app.MapGet("/items/{id:int}", GetItem);
app.MapPost("/items", CreateItem);
app.MapPut("/items/{id:int}", ReplaceItem);
app.MapPatch("/items/{id:int}", UpdateItem);
app.MapDelete("/items/{id:int}", DeleteItem);
app.MapMethods("/items", ["HEAD"], HeadItems);
```

A route parameter binds to a handler parameter with the same name:

``` csharp
static IResult GetItem(int id) => TypedResults.Ok(new { id });
```

Matching is not case-sensitive, so `/Items/1` matches `/items/{id:int}`.

------------------------------------------------------------------------

## 2. Template Syntax

| Template | Matches | Value |
|----|----|----|
| `/items/{id}` | `/items/42` | `id = "42"` |
| `/items/{id?}` | `/items` and `/items/42` | `id` is optional |
| `/items/{page=1}` | `/items` | `page` defaults to `1` |
| `/files/{*path}` | `/files/a/b.txt` | `path = "a/b.txt"` |
| `/files/{**path}` | `/files/a/b.txt` | Same match, and generated links keep `/` unescaped |
| `/{slug}.html` | `/about.html` | `slug = "about"` |

------------------------------------------------------------------------

## 3. Route Constraints

Constraints select a route. They do not validate input. A value that fails a constraint produces a `404`, because no route matches.

| Constraint | Example | Matches |
|----|----|----|
| `int` | `{id:int}` | `42` |
| `long` | `{id:long}` | Large integers |
| `guid` | `{id:guid}` | A GUID |
| `bool` | `{on:bool}` | `true` or `false` |
| `datetime` | `{day:datetime}` | A date |
| `min` and `max` | `{n:int:min(1)}` | `1` or greater |
| `range` | `{n:range(1,100)}` | `1` to `100` |
| `minlength` and `maxlength` | `{code:maxlength(8)}` | Up to 8 characters |
| `alpha` | `{name:alpha}` | Letters only |
| `regex` | `{sku:regex(^[[A-Z]]{{3}}$)}` | A regular expression |

Escape `[`, `]`, `{` and `}` in a regex constraint by doubling them. Prefer handler validation over complex regex constraints, because validation can return a clear `400` message.

------------------------------------------------------------------------

## 4. Route Groups

A **route group** shares a prefix and metadata across endpoints.

``` csharp
var api = app.MapGroup("/api/v1")
    .RequireAuthorization();

var items = api.MapGroup("/items").WithTags("Items");

items.MapGet("/", ListItems);
items.MapGet("/{id:int}", GetItem);
items.MapPost("/", CreateItem).RequireAuthorization("items:write");

api.MapGroup("/health").AllowAnonymous().MapGet("/", () => "ok");
```

Groups can nest. Metadata on the outer group applies to each inner endpoint.

------------------------------------------------------------------------

## 5. Precedence

Route order in the code does not decide the winner. The router selects the most specific template.

| Priority | Segment type | Example |
|----|----|----|
| Highest | Literal | `/items/featured` |
| High | Parameter with a constraint, or a complex segment | `/items/{id:int}`, `/items/{name}.json` |
| Medium | Parameter | `/items/{slug}` |
| Lowest | Catch-all | `/items/{**rest}` |

Two equally specific routes raise an `AmbiguousMatchException` at request time, which produces a `500`.

``` csharp
// Ambiguous: both match /items/42
app.MapGet("/items/{id}", ById);
app.MapGet("/items/{name}", ByName);
```

------------------------------------------------------------------------

## 6. Generate URLs

Name a route, then generate its URL. A route change then cannot break the links.

``` csharp
items.MapGet("/{id:int}", GetItem).WithName("GetItem");

items.MapPost("/", (CreateItem body, LinkGenerator links, HttpContext http) =>
{
    var item = new ItemDto(7, body.Name);
    var url = links.GetPathByName(http, "GetItem", new { id = item.Id });
    return TypedResults.Created(url, item);
});
```

`TypedResults.CreatedAtRoute(item, "GetItem", new { id = item.Id })` is a shorter form for this common case.

------------------------------------------------------------------------

## 7. Inspect the Route Table

Inject `EndpointDataSource` to list every endpoint in Development:

``` csharp
if (app.Environment.IsDevelopment())
{
    app.MapGet("/debug/routes", (IEnumerable<EndpointDataSource> sources) =>
        sources.SelectMany(s => s.Endpoints)
               .OfType<RouteEndpoint>()
               .Select(e => e.RoutePattern.RawText));
}
```

This is the equivalent of `manage.py show_urls` from `django-extensions`.

------------------------------------------------------------------------

## 8. Compared With Django

| Django | ASP.NET Core |
|----|----|
| `path("items/<int:pk>/", view)` | `app.MapGet("/items/{pk:int}", handler)` |
| `include("items.urls")` | `MapGroup("/items")` |
| `reverse("item-detail", args=[1])` | `LinkGenerator.GetPathByName` |
| DRF `DefaultRouter` | Explicit `Map` calls or controllers |
| First matching pattern wins | Most specific pattern wins |
| Trailing slash is significant | Trailing slash is optional |

------------------------------------------------------------------------

## 9. Common Mistakes

| Mistake | What goes wrong | Do this instead |
|----|----|----|
| Using constraints for validation | Bad input gets a confusing `404` | Validate in the handler and return `400` |
| Two routes with the same shape | `500` at request time | Add a literal segment or a constraint |
| Generating URLs by string concatenation | Links break when routes change | Use `LinkGenerator` |
| Parameter name differs from the template | The value binds from the query string instead | Keep the names identical |
| Catch-all route placed under a group by accident | It hides expected `404` responses | Limit it to one known prefix |

------------------------------------------------------------------------

Back to top
