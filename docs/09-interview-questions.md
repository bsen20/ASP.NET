# 09 — .NET / ASP.NET Core Interview Questions

> Organized by topic. Questions increase in difficulty.  
> Use these to assess your understanding and prepare for interviews.

---

## 1. C# Fundamentals

### Q1: What's the difference between `ref`, `out`, and `in` parameters?
**A:**
- `ref`: Pass by reference. Must be initialized before calling.
- `out`: Pass by reference. Must be assigned inside the method.
- `in`: Pass by reference but **readonly**. Cannot be modified.

```csharp
void Modify(ref int x) { x = 10; }       // x must be initialized
void Create(out int x) { x = 10; }        // x must be assigned inside
void Read(in int x) { /* x is readonly */ }

int a = 5;
Modify(ref a);  // a = 10

Create(out int b);  // b = 10
```

### Q2: What are `record` types and when would you use them?
**A:** Records are immutable reference types with **value-based equality**. Perfect for DTOs, API responses, and any type where equality is based on data, not identity.

```csharp
public record ProductDto(int Id, string Name, decimal Price);

// Value equality
var a = new ProductDto(1, "A", 10);
var b = new ProductDto(1, "A", 10);
Console.WriteLine(a == b);  // true (unlike class)

// Non-destructive mutation
var c = a with { Price = 20 };
```

### Q3: Explain `async` / `await`. What happens under the hood?
**A:** `async` marks a method as asynchronous. `await` yields control to the caller until the awaited task completes. The compiler transforms the method into a **state machine**.

```csharp
public async Task<Product> GetProductAsync(int id) {
    // Method runs synchronously until await
    var product = await _db.Products.FindAsync(id);  // Yield here
    return product;  // Resume when task completes
}
```

**Key:** `await` does NOT block the thread. It returns a Task and resumes on the captured SynchronizationContext.

### Q4: What's the difference between `String` and `string`? `int` vs `Int32`?
**A:** No difference. `string` is an alias for `System.String`, `int` for `System.Int32`. The lowercase versions are C# keywords.

### Q5: What are extension methods?
**A:** Methods that appear to extend a type without modifying it. Used heavily by LINQ.

```csharp
public static class StringExtensions {
    public static bool IsPalindrome(this string str) {
        return str.SequenceEqual(str.Reverse());
    }
}

"racecar".IsPalindrome();  // true
```

---

## 2. .NET / CLR Concepts

### Q6: What is the CLR? How is it different from the JVM?
**A:** Common Language Runtime — .NET's virtual machine. Like JVM:
- Compiles IL (CIL/MSIL) to native code via JIT
- Manages memory (GC), threads, exceptions
- Enforces type safety and security

**Key difference:** .NET has **multiple languages** (C#, F#, VB.NET) compiling to the same IL.

### Q7: Explain the garbage collector. Generations?
**A:** .NET uses a generational GC:
- **Gen 0:** Short-lived objects (local vars). Collected most frequently.
- **Gen 1:** Buffer between Gen 0 and Gen 2.
- **Gen 2:** Long-lived objects (static data, singletons, large objects).
- **LOH (Large Object Heap):** Objects > 85KB. Collected with Gen 2.

Objects are promoted: survive Gen 0 → Gen 1 → Gen 2.

### Q8: Value types vs Reference types?
**A:**
| | Value Types (`struct`) | Reference Types (`class`) |
|--|----------------------|--------------------------|
| Memory | Stack (or inline in object) | Heap |
| Assignment | Copies value | Copies reference |
| Nullable | No (unless `Nullable<T>`) | Yes |
| Default | `0`, `false`, etc. | `null` |
| Examples | `int`, `bool`, `DateTime`, `struct` | `string`, `class`, `interface`, `record` |

### Q9: What is `IDisposable`? What is the `using` statement?
**A:** `IDisposable` is for unmanaged resource cleanup (file handles, DB connections, sockets). `using` ensures `Dispose()` is called even if an exception occurs.

```csharp
using var file = File.OpenRead("data.txt");
// file.Dispose() called automatically at end of scope
```

### Q10: Boxing and unboxing?
**A:** Converting a value type to `object` (boxing) allocates on the heap. Unboxing extracts it back. Avoid in perf-critical paths.

```csharp
int x = 42;
object o = x;       // Boxing — 42 copied to heap
int y = (int)o;     // Unboxing
```

---

## 3. Dependency Injection

### Q11: What are the three DI lifetimes in .NET? Give real-world examples.
**A:**
| Lifetime | Behavior | Example |
|----------|----------|---------|
| **Singleton** | One instance for the whole app | Cache service, Configuration, Logger |
| **Scoped** | One per HTTP request | DbContext, Unit of Work |
| **Transient** | New every injection | Stateless calculators, lightweight utilities |

### Q12: What is a captive dependency? How do you detect it?
**A:** When a **Singleton** depends on a **Scoped** or **Transient** service. The scoped service is captured forever by the singleton, leading to stale data.

```csharp
// ❌ WRONG
builder.Services.AddSingleton<IService, MyService>();
builder.Services.AddScoped<IRepo, MyRepo>();

// MyService will capture the first MyRepo forever
```

**Detection:** .NET 8+ validates at startup. Also check with analyzers.

### Q13: How do you register multiple implementations of the same interface?
**A:** Register each implementation, then inject `IEnumerable<T>`:

```csharp
builder.Services.AddScoped<IPaymentProcessor, CreditCardProcessor>();
builder.Services.AddScoped<IPaymentProcessor, PayPalProcessor>();

public class PaymentService(IEnumerable<IPaymentProcessor> processors) { }
```

### Q14: What is the Options pattern and why use it?
**A:** Strongly-typed configuration access with `IOptions<T>`, `IOptionsSnapshot<T>` (reloads per request), or `IOptionsMonitor<T>` (reloads on change).

```csharp
builder.Services.Configure<JwtSettings>(config.GetSection("Jwt"));

public class AuthService {
    public AuthService(IOptions<JwtSettings> options) {
        var settings = options.Value;  // Strongly-typed access
    }
}
```

---

## 4. Entity Framework Core

### Q15: What's the difference between `IEnumerable<T>` and `IQueryable<T>` in EF?
**A:**
- `IQueryable<T>` builds an expression tree that EF translates to SQL — **filtering happens in the database**.
- `IEnumerable<T>` executes the query immediately and filters **in memory**.

```csharp
// IQueryable — SQL: SELECT * FROM Products WHERE Price > 100
var query = db.Products.Where(p => p.Price > 100);

// IEnumerable — loads ALL products then filters
var enumerable = db.Products.AsEnumerable().Where(p => p.Price > 100);
```

### Q16: What is the N+1 query problem? How do you prevent it?
**A:** When you load N parent entities and each triggers an additional query for its children (total: 1 + N queries).

```csharp
// ❌ N+1: 1 query for blogs + N queries for posts
foreach (var blog in blogs) {
    Console.WriteLine(blog.Posts.Count);  // Extra query each time!
}

// ✅ Fix: Eager loading with Include (1 JOIN query)
var blogs = await db.Blogs.Include(b => b.Posts).ToListAsync();
```

### Q17: Explain the difference between `FindAsync` and `FirstOrDefaultAsync`.
**A:**
- `FindAsync`: Looks in the **tracked entities cache first** (no DB call if already loaded), then queries by PK. **Only works with key**.
- `FirstOrDefaultAsync`: Always goes to the database. Works with any predicate.

```csharp
// FindAsync — checks local cache first
var p1 = await db.Products.FindAsync(1);  // May hit cache

// FirstOrDefaultAsync — always hits DB
var p2 = await db.Products.FirstOrDefaultAsync(p => p.Name == "X");
```

### Q18: What are EF Core Migrations? Walk through the workflow.
**A:** Migrations are C# files that describe schema changes. Workflow:

```bash
# 1. Create/change entity classes
# 2. Add migration
dotnet ef migrations add AddProductDescription
# 3. Review generated code in Migrations/ folder
# 4. Apply
dotnet ef database update
# 5. Or generate SQL script for DBA review
dotnet ef migrations script
```

### Q19: Explain `AsNoTracking()`. When would you use it?
**A:** Disables change tracking. Faster and uses less memory. Use for **read-only queries** where you won't update the entities.

```csharp
var products = await db.Products.AsNoTracking().ToListAsync();  // ~30% faster
```

### Q20: How do you handle soft deletes in EF Core?
**A:** Add an `IsDeleted` flag and a global query filter:

```csharp
public class Product {
    public bool IsDeleted { get; set; }
}

// In OnModelCreating:
entity.HasQueryFilter(p => !p.IsDeleted);

// Bypass filter when needed:
var deleted = await db.Products.IgnoreQueryFilters()
    .Where(p => p.IsDeleted).ToListAsync();
```

---

## 5. LINQ

### Q21: What is deferred execution? Give an example of a bug it could cause.
**A:** LINQ queries don't execute until you materialize the results. Bug example:

```csharp
var query = products.Where(p => p.Price > 100);
products.Add(new Product { Name = "Late", Price = 200 });
var result = query.ToList();  // "Late" is included! // Unexpected!
```

### Q22: Write a LINQ query to group products by category and show average price.
**A:**
```csharp
var result = products
    .GroupBy(p => p.CategoryId)
    .Select(g => new {
        CategoryId = g.Key,
        Count = g.Count(),
        AvgPrice = g.Average(p => p.Price),
        MaxPrice = g.Max(p => p.Price)
    })
    .ToList();
```

### Q23: How do you do pagination with LINQ?
**A:**
```csharp
var page = products
    .OrderBy(p => p.Id)              // Must order before Skip/Take
    .Skip((pageNumber - 1) * pageSize)
    .Take(pageSize)
    .ToList();
```

### Q24: Difference between `Select` and `SelectMany`?
**A:**
- `Select`: 1-to-1 mapping. Each input → one output.
- `SelectMany`: 1-to-many mapping. Each input → many outputs, then flattened.

```csharp
// Select — list of lists
var listOfLists = blogs.Select(b => b.Posts);        // IEnumerable<List<Post>>

// SelectMany — flat list
var allPosts = blogs.SelectMany(b => b.Posts);        // IEnumerable<Post>
```

### Q25: Explain `GroupJoin` with an example.
**A:** `GroupJoin` creates a left outer join — every element of the outer sequence appears once, paired with elements from the inner sequence.

```csharp
var result = categories.GroupJoin(
    products,                    // inner
    c => c.Id,                   // outer key
    p => p.CategoryId,           // inner key
    (category, categoryProducts) => new {
        Category = category.Name,
        Products = categoryProducts.DefaultIfEmpty()
    }
);
```

---

## 6. ASP.NET Core

### Q26: What is middleware? Walk through the pipeline.
**A:** Middleware are components that handle requests and responses in a pipeline. Each middleware can:
1. Process the request
2. Call the next middleware
3. Process the response on the way back

```
Request → Logger → Auth → CORS → Controller → Response
```

### Q27: What's the difference between `AddTransient`, `AddScoped`, and `AddSingleton`?
**A:** See Q11. Know this cold — it's the most-asked DI question.

### Q28: How do controllers work in ASP.NET Core?
**A:** Controllers handle HTTP requests. They:
- Are registered via `AddControllers()`
- Use `[ApiController]` and `[Route]` attributes
- Receive dependencies via constructor injection
- Can return `ActionResult<T>`, `IActionResult`, or specific types
- Automatically validate `[FromBody]` parameters

### Q29: `ActionResult<T>` vs `IActionResult` — when to use which?
**A:**
- `ActionResult<T>`: Type-safe. Know what's returned. Produces documentation. **Prefer this.**
- `IActionResult`: When returning different types (e.g., File, Redirect).

### Q30: How do you read configuration in ASP.NET Core?
**A:** Multiple ways:

```csharp
// 1. Direct access
var val = builder.Configuration["Section:Key"];

// 2. Strongly-typed (Options pattern) — BEST
builder.Services.Configure<MyOptions>(config.GetSection("MyOptions"));

// 3. In a service
public class MyService(IOptions<MyOptions> options) {
    var opts = options.Value;
}
```

### Q31: What is Kestrel?
**A:** Kestrel is the **cross-platform web server** built into ASP.NET Core. It's what actually listens on the socket. In production, it typically sits behind IIS (Windows), Nginx (Linux), or a reverse proxy.

### Q32: Explain the startup process.
**A:**
1. `WebApplication.CreateBuilder(args)` — create host builder
2. Configure services (DI registration)
3. `builder.Build()` — build the app (container is frozen)
4. Configure middleware pipeline
5. `app.Run()` — start listening

---

## 7. Advanced

### Q33: How does the `async` state machine work? What is `ConfigureAwait(false)`?
**A:** The compiler generates a state machine struct. When `await` is hit:
1. If the task is already complete → continue synchronously
2. If not → capture context, return task, resume when done

`ConfigureAwait(false)` tells the awaiter not to marshal back to the original context. Use in libraries to avoid deadlocks.

### Q34: Explain `Span<T>` vs `Memory<T>`.
**A:**
- `Span<T>`: Stack-only value type. Fast access to contiguous memory. Cannot be used in async methods.
- `Memory<T>`: Wrapper that can be stored on heap. Used in async scenarios.

```csharp
Span<int> span = stackalloc int[] { 1, 2, 3 };  // Stack allocated
Memory<int> memory = new int[] { 1, 2, 3 };      // Heap — can be used in async
```

### Q35: What's your approach to error handling in ASP.NET Core?
**A:** Multi-layered:
1. **FluentValidation** for request validation
2. **Global exception middleware** for unhandled exceptions
3. **ProblemDetails** standard for error responses (RFC 7807)
4. **ILogger** to log all errors
5. **Return appropriate status codes** (400/404/409/500)

---

## 8. Scenario-Based Questions

### Q36: Your API returns products slowly. How do you debug and fix it?
**A:** 
1. Check if N+1 queries exist (look for Lazy Loading)
2. Add `AsNoTracking()` for read-only queries
3. Check if filtering happens in memory (move `Where` before `ToList`)
4. Check generated SQL (enable logging)
5. Add pagination if returning large datasets
6. Add caching for frequently accessed data
7. Add database indexes

### Q37: A background service needs to use DbContext. What's the correct pattern?
**A:** Don't inject DbContext directly (it's Scoped). Use `IServiceScopeFactory`:

```csharp
public class CleanupService : BackgroundService {
    private readonly IServiceScopeFactory _scopeFactory;

    protected override async Task ExecuteAsync(CancellationToken ct) {
        using var scope = _scopeFactory.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        // Use db...
    }
}
```

### Q38: You're getting `ObjectDisposedException` from DbContext. Why?
**A:** The DbContext was disposed (end of HTTP request) but something is still trying to use it. Common causes:
- Lazy loading after the request ends
- Returning `IQueryable` from a controller (query executes after dispose)
- Background thread using a request-scoped DbContext

**Fix:** Use `AsNoTracking()` or eager load before the request ends.

---

## 9. Quick Fire Questions

| # | Question | Answer |
|---|----------|--------|
| 39 | What's the default DI lifetime? | Transient |
| 40 | `List<T>` vs `IEnumerable<T>`? | `List<T>` is materialized, `IEnumerable<T>` can be deferred |
| 41 | `StringBuilder` vs string concatenation? | StringBuilder for **many** concatenations (reduces allocations) |
| 42 | What's `Task` vs `ValueTask`? | `ValueTask` is a struct — reduces allocations in hot paths |
| 43 | What's `[FromBody]` vs `[FromQuery]`? | Body = JSON body, Query = URL parameters |
| 44 | What port does `dotnet run` use by default? | 5000 (HTTP) and 5001 (HTTPS) |
| 45 | `virtual` keyword? | Allows method to be overridden in derived class |
| 46 | `sealed` keyword? | Prevents further inheritance/override |
| 47 | `abstract` vs `interface`? | Abstract can have implementation; interface is pure contract |
| 48 | What's the `=>` operator? | Lambda (expression body) or expression-bodied member |
| 49 | `??` vs `?.` vs `??=`? | `??` = null-coalesce, `?.` = null-conditional, `??=` = null-coalesce assignment |
| 50 | `new()` constraint? | Requires a parameterless constructor for generic types |

---

## 10. Coding Challenge — Whiteboard

```csharp
// Implement a FizzBuzz service with DI:

public interface IFizzBuzzService {
    string Evaluate(int number);
}

public class FizzBuzzService : IFizzBuzzService {
    private readonly IEnumerable<IFizzBuzzRule> _rules;

    public FizzBuzzService(IEnumerable<IFizzBuzzRule> rules) {
        _rules = rules;
    }

    public string Evaluate(int number) {
        var result = string.Concat(_rules
            .Where(r => r.Matches(number))
            .Select(r => r.Output));

        return string.IsNullOrEmpty(result) ? number.ToString() : result;
    }
}

public interface IFizzBuzzRule {
    bool Matches(int number);
    string Output { get; }
}

public class FizzRule : IFizzBuzzRule {
    public bool Matches(int n) => n % 3 == 0;
    public string Output => "Fizz";
}

public class BuzzRule : IFizzBuzzRule {
    public bool Matches(int n) => n % 5 == 0;
    public string Output => "Buzz";
}

// Test this in an interview — demonstrates DI, IEnumerable<T>, SRP, strategy pattern
```

---

> **Preparation Tip:** For each topic, be ready to say "I've used this in [project X] to solve [problem Y]". Concrete examples are worth more than textbook definitions.
