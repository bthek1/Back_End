# Server-Rendered UI

Razor syntax, MVC, Razor Pages and Blazor render modes, and how to host a React single-page app from ASP.NET Core.

Author

Benedict Thekkel

ASP.NET Core can render HTML on the server, and it can also serve a separate React build. Choose one UI model for each app. If you already use React and TypeScript, keep them and read this page mainly to understand existing .NET codebases.

> **NOTE:**
>
> The code on this page has not been compiled or run yet. It targets .NET 11 and C# 15.

------------------------------------------------------------------------

## 1. Choose a UI Model

| Option | How it works | Best fit |
|----|----|----|
| React with an API | The browser renders, and ASP.NET Core serves JSON | Your default stack |
| Razor Pages | One page file with one handler class | Forms and content pages |
| MVC | Controllers select views | Large existing server-rendered apps |
| Blazor | C# components, rendered on the server or in WebAssembly | Internal tools from a C# team |

------------------------------------------------------------------------

## 2. Razor Syntax

**Razor** mixes C# into HTML. The `@` character switches to C#.

``` cshtml
@{
    var title = "Items";
}

<h1>@title</h1>

@if (Model.Items.Count == 0)
{
    <p>No items yet.</p>
}
else
{
    <ul>
        @foreach (var item in Model.Items)
        {
            <li>@item.Name costs @item.Price.ToString("C")</li>
        }
    </ul>
}
```

Razor encodes output by default, which blocks most XSS (cross-site scripting). `@Html.Raw(value)` turns that protection off, so use it only for trusted content.

| Django template             | Razor                                      |
|-----------------------------|--------------------------------------------|
| `{ item.name }`             | `@item.Name`                               |
| `{% if x %}`                | `@if (x) { }`                              |
| `{% for i in items %}`      | `@foreach (var i in items) { }`            |
| `{% extends "base.html" %}` | `Layout = "_Layout";`                      |
| `{% block content %}`       | `@RenderBody()` and `@section`             |
| `{% include "row.html" %}`  | `<partial name="_Row" />`                  |
| `{% url "item" 1 %}`        | `asp-page` or `asp-action` tag helpers     |
| `{% csrf_token %}`          | Added automatically by the form tag helper |

------------------------------------------------------------------------

## 3. Razor Pages

A **Razor Page** keeps the markup and its handlers together. Use it for a new server-rendered form.

``` csharp
builder.Services.AddRazorPages();

var app = builder.Build();

app.UseStaticFiles();
app.MapRazorPages();
```

`Pages/Items/Create.cshtml.cs`:

``` csharp
using System.ComponentModel.DataAnnotations;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;

public sealed class CreateModel(AppDbContext db) : PageModel
{
    [BindProperty]
    public InputModel Input { get; set; } = new();

    public sealed class InputModel
    {
        [Required, StringLength(100)]
        public string Name { get; set; } = "";
    }

    public async Task<IActionResult> OnPostAsync(CancellationToken ct)
    {
        if (!ModelState.IsValid)
        {
            return Page();
        }

        db.Items.Add(new Item { Name = Input.Name });
        await db.SaveChangesAsync(ct);
        return RedirectToPage("Index");
    }
}
```

`Pages/Items/Create.cshtml`:

``` cshtml
@page
@model CreateModel

<form method="post">
    <label asp-for="Input.Name"></label>
    <input asp-for="Input.Name" />
    <span asp-validation-for="Input.Name"></span>
    <button type="submit">Save</button>
</form>
```

The `asp-for` **tag helpers** generate field names, validation attributes and the antiforgery token. The redirect after a successful post stops a browser refresh from submitting the form twice.

------------------------------------------------------------------------

## 4. MVC

MVC separates the controller, the model and the view. Most older .NET web apps use this structure.

``` csharp
builder.Services.AddControllersWithViews();

var app = builder.Build();

app.MapControllerRoute("default", "{controller=Home}/{action=Index}/{id?}");
```

``` csharp
using Microsoft.AspNetCore.Mvc;

public sealed class ItemsController(AppDbContext db) : Controller
{
    public async Task<IActionResult> Index(CancellationToken ct)
    {
        var items = await db.Items.AsNoTracking().ToListAsync(ct);
        return View(items);
    }
}
```

The action renders `Views/Items/Index.cshtml`. Django calls this same pattern MTV (model, template, view), and a Django view does the job of an MVC controller.

------------------------------------------------------------------------

## 5. Blazor

**Blazor** builds interactive components in C#. A component is a `.razor` file.

``` razor
@page "/counter"
@rendermode InteractiveServer

<h1>Count: @count</h1>
<button @onclick="Increment">Add one</button>

@code {
    private int count;

    private void Increment() => count++;
}
```

The **render mode** decides where the component runs:

| Render mode | Where code runs | Trade-off |
|----|----|----|
| Static server rendering | Server, HTML only | Fast and simple, with no interactivity |
| Interactive Server | Server, updates over SignalR | Small download, but needs a live connection |
| Interactive WebAssembly | Browser | Works offline after a larger download |
| Interactive Auto | Server first, then WebAssembly | Best first load, with the most complexity |

``` csharp
builder.Services.AddRazorComponents()
    .AddInteractiveServerComponents();

var app = builder.Build();

app.UseAntiforgery();
app.MapStaticAssets();
app.MapRazorComponents<App>()
    .AddInteractiveServerRenderMode();
```

Interactive Server keeps state in server memory for each connected user. Plan capacity for that memory, and use sticky sessions with several replicas.

------------------------------------------------------------------------

## 6. Host a React SPA

Build React with Vite and serve the output from the API. The browser and the API then share one origin, so you need no CORS rules.

``` csharp
var app = builder.Build();

app.UseDefaultFiles();
app.MapStaticAssets();

app.MapGroup("/api").MapItems();

app.MapFallbackToFile("index.html");

app.Run();
```

`MapFallbackToFile` returns `index.html` for unknown paths, so client-side routes survive a browser refresh. Keep all API routes under `/api`.

In development, run Vite separately and proxy API calls:

``` typescript
// vite.config.ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  server: {
    proxy: { "/api": "http://localhost:5050" },
  },
});
```

Copy the Vite `dist` output into `wwwroot` during the Docker build.

------------------------------------------------------------------------

## 7. Static Files

| Method | Behaviour |
|----|----|
| `MapStaticAssets` | Serves build-time files with compression and fingerprinted cache headers |
| `UseStaticFiles` | Serves files from `wwwroot` at run time |

Use `MapStaticAssets` for files known at build time. Use `UseStaticFiles` for files added after the build.

------------------------------------------------------------------------

## 8. Antiforgery

Cookie-authenticated forms need protection from CSRF (cross-site request forgery). Razor form tag helpers include the token automatically. Minimal API form endpoints validate it through the middleware:

``` csharp
builder.Services.AddAntiforgery();

var app = builder.Build();
app.UseAntiforgery();
```

------------------------------------------------------------------------

## 9. Common Mistakes

| Mistake | What goes wrong | Do this instead |
|----|----|----|
| `@Html.Raw` on user content | Stored XSS | Keep the default encoding |
| No redirect after POST | A refresh repeats the submission | Redirect on success |
| Missing `MapFallbackToFile` | Refreshing a React route returns `404` | Add the fallback last |
| API routes outside `/api` | The SPA fallback can hide missing endpoints | Group all API routes |
| Blazor Server with no capacity plan | Memory grows with each connected user | Load test the connection count |
| Mixing Razor Pages, MVC and Blazor | The team maintains three UI models | Choose one |

------------------------------------------------------------------------

Back to top
