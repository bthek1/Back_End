# C# and .NET Basics

The C# you need before ASP.NET Core: types, nullability, records, generics, LINQ, async and the dotnet CLI, explained from Python.

Author

Benedict Thekkel

C# is statically typed and compiled, so the compiler rejects many mistakes that Python only finds at run time. Most concepts carry over from typed Python. This page covers the language features that every later page uses.

> **NOTE:**
>
> The code on this page has not been compiled or run yet. It targets .NET 11 and C# 15.

------------------------------------------------------------------------

## 1. Project and CLI

Create a console project and run it:

``` bash
dotnet new console -n Basics
cd Basics
dotnet run
```

| Task             | Python           | .NET                       |
|------------------|------------------|----------------------------|
| Add a package    | `uv add httpx`   | `dotnet add package Polly` |
| Restore packages | `uv sync`        | `dotnet restore`           |
| Compile          | No separate step | `dotnet build`             |
| Run tests        | `pytest`         | `dotnet test`              |
| Format           | `ruff format`    | `dotnet format`            |

A file can hold **top-level statements**, which are code without a `Main` method. Only one file in a project can use them, and that file is normally `Program.cs`.

------------------------------------------------------------------------

## 2. Types and Nullability

`var` infers the type at compile time. The variable cannot change type later.

``` csharp
var count = 3;              // int
decimal price = 19.95m;     // decimal, use it for money
string name = "Ada";        // cannot hold null
string? nickname = null;    // can hold null

int length = nickname?.Length ?? 0;
```

| Operator | Meaning | Python equivalent |
|----|----|----|
| `string?` | The value can be null | `Optional[str]` |
| `a?.B` | Read `B` only when `a` is not null | `a.b if a is not None else None` |
| `a ?? b` | Use `b` when `a` is null | `a if a is not None else b` |
| `a ??= b` | Assign `b` when `a` is null | `if a is None: a = b` |
| `a!` | Tell the compiler that `a` is not null | `typing.cast` |

Keep `<Nullable>enable</Nullable>` in every project. Treat each `!` as a claim that you must be able to prove.

------------------------------------------------------------------------

## 3. Classes, Records and Interfaces

Use a **record** for data, a **class** for behaviour and an **interface** for a contract.

``` csharp
public record ItemDto(int Id, string Name, decimal Price);

public interface IClock
{
    DateTimeOffset UtcNow { get; }
}

public sealed class SystemClock : IClock
{
    public DateTimeOffset UtcNow => DateTimeOffset.UtcNow;
}

public sealed class InvoiceService(IClock clock)
{
    public bool IsOverdue(DateTimeOffset due) => clock.UtcNow > due;
}
```

`InvoiceService(IClock clock)` is a **primary constructor**. The `clock` parameter stays available to every method in the class.

Records compare by value, and `with` creates a changed copy:

``` csharp
var a = new ItemDto(1, "Pen", 2.50m);
var b = a with { Price = 3.00m };
Console.WriteLine(a == b); // False
```

Use `DateTimeOffset.UtcNow` for timestamps. It carries the UTC offset, which plays the same role as a timezone-aware Python datetime.

------------------------------------------------------------------------

## 4. Generics and Collections

Generics are checked by the compiler and exist at run time.

``` csharp
List<int> numbers = [1, 2, 3];
Dictionary<string, int> stock = new() { ["pen"] = 4 };
HashSet<string> tags = ["new", "sale"];

public sealed class Page<T>(IReadOnlyList<T> items, int total)
{
    public IReadOnlyList<T> Items { get; } = items;
    public int Total { get; } = total;
}
```

| Python | C# | Note |
|----|----|----|
| `list` | `List<T>` | Growable array |
| `dict` | `Dictionary<TKey, TValue>` | Not thread-safe |
| `set` | `HashSet<T>` | Not thread-safe |
| `collections.deque` | `Queue<T>` | First in, first out |
| Thread-safe dict | `ConcurrentDictionary<TKey, TValue>` | Use it in singletons |

------------------------------------------------------------------------

## 5. Pattern Matching

`switch` expressions return a value, and the compiler warns when a case is missing.

``` csharp
static string Describe(object value) => value switch
{
    null => "nothing",
    int n when n < 0 => "negative",
    int => "a number",
    string { Length: 0 } => "empty text",
    string s => $"text: {s}",
    _ => "unknown",
};
```

------------------------------------------------------------------------

## 6. LINQ

**LINQ** (Language Integrated Query) replaces comprehensions, `map`, `filter` and `sorted`.

``` csharp
var names = items
    .Where(i => i.Price > 10m)
    .OrderBy(i => i.Name)
    .Select(i => i.Name)
    .ToList();

var totals = items
    .GroupBy(i => i.Category)
    .ToDictionary(g => g.Key, g => g.Sum(i => i.Price));
```

| Python                                | LINQ                   |
|---------------------------------------|------------------------|
| `[f(x) for x in xs]`                  | `xs.Select(f)`         |
| `[x for x in xs if p(x)]`             | `xs.Where(p)`          |
| `sorted(xs, key=k)`                   | `xs.OrderBy(k)`        |
| `any(p(x) for x in xs)`               | `xs.Any(p)`            |
| `next(x for x in xs if p(x))`         | `xs.First(p)`          |
| `next((x for x in xs if p(x)), None)` | `xs.FirstOrDefault(p)` |

LINQ is **lazy**, like a Python generator. Nothing runs until you enumerate the result or call `ToList`, `Count` or `First`. Each new enumeration runs the query again.

------------------------------------------------------------------------

## 7. Async and Await

An `async` method returns a `Task` or a `Task<T>`.

``` csharp
public async Task<string> ReadTitleAsync(HttpClient http, string url, CancellationToken ct)
{
    using var response = await http.GetAsync(url, ct);
    response.EnsureSuccessStatusCode();
    return await response.Content.ReadAsStringAsync(ct);
}

var both = await Task.WhenAll(
    ReadTitleAsync(http, urlA, ct),
    ReadTitleAsync(http, urlB, ct));
```

Python coroutines start only when awaited. A .NET method starts as soon as you call it, and the returned `Task` represents work already in progress. `Task.WhenAll` is the equivalent of `asyncio.gather`.

Pass a `CancellationToken` through every async call. ASP.NET Core cancels the token when the client disconnects, so abandoned requests stop using the database.

------------------------------------------------------------------------

## 8. Errors and Disposal

``` csharp
try
{
    await SaveAsync(item, ct);
}
catch (TimeoutException ex) when (attempt < 3)
{
    logger.LogWarning(ex, "Save timed out on attempt {Attempt}", attempt);
}
finally
{
    timer.Stop();
}
```

`using` is the equivalent of a Python `with` block. It calls `Dispose` when the variable leaves scope. Use `await using` for a resource that closes asynchronously.

``` csharp
await using var stream = File.OpenWrite("report.csv");
```

------------------------------------------------------------------------

## 9. Common Mistakes

| Mistake | What goes wrong | Do this instead |
|----|----|----|
| Calling `.Result` or `.Wait()` | Blocks a thread and can stall the server | `await` the task |
| Using `async void` | The caller cannot catch the exception | Return `Task` |
| Enumerating a LINQ query twice | The work runs twice | Call `ToList()` once |
| Using `float` or `double` for money | Rounding errors | Use `decimal` |
| Using `DateTime.Now` | Depends on the server timezone | Use `DateTimeOffset.UtcNow` |
| Comparing strings with a default culture | Results change between machines | Use `StringComparison.Ordinal` |

------------------------------------------------------------------------

Back to top
