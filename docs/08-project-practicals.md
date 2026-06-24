# 08 — Practical Projects & Exercises

> **Learning by doing.** Build these projects in order. Each one reinforces the concepts from the docs.

---

## Project 1: Console-Based Banking System

**Goal:** Practice C# fundamentals (classes, LINQ, DI)

### Requirements
- Create `BankAccount`, `Transaction` classes
- Support Deposit, Withdraw, Transfer
- Calculate interest using DI (`IInterestCalculator`)
- Use LINQ to: find top 5 accounts by balance, filter transactions by date, group by account type
- Store data in memory (List<T>)

### Key Concepts
- Classes, properties, records
- LINQ to Objects
- Basic DI (manually or with `Microsoft.Extensions.DependencyInjection`)
- `ILogger<T>`

### Sample Structure

```
BankingApp/
├── Program.cs
├── Models/
│   ├── BankAccount.cs
│   └── Transaction.cs
├── Services/
│   ├── IAccountService.cs
│   ├── AccountService.cs
│   ├── IInterestCalculator.cs
│   └── SimpleInterestCalculator.cs
└── Data/
    └── InMemoryDataStore.cs
```

---

## Project 2: Todo API (Minimal API)

**Goal:** First ASP.NET Core project — Minimal API

### Requirements
- `GET /todos` — list all
- `POST /todos` — create
- `PUT /todos/{id}` — toggle complete
- `DELETE /todos/{id}` — delete
- Use in-memory EF Core database
- Add validation (FluentValidation)
- Add request logging middleware

### Key Concepts
- Minimal APIs vs Controllers
- EF Core InMemory
- Middleware
- Configuration (IOptions)

### Stretch Goals
- Add JWT auth
- Add filtering by status
- Add pagination

---

## Project 3: BookStore API (Full Stack)

**Goal:** Full-featured Web API with DI, EF, LINQ

### Requirements

```
Books (Id, Title, AuthorId, Price, PublishedDate, Genre, ISBN)
Authors (Id, Name, Bio, BirthDate)
Orders (Id, CustomerName, OrderDate, TotalAmount)
OrderItems (Id, OrderId, BookId, Quantity, UnitPrice)
```

### Endpoints

```csharp
// Books
GET    /api/books                    // List with search, filter, sort, pagination
GET    /api/books/{id}               // Detail with author info
POST   /api/books                    // Create
PUT    /api/books/{id}               // Update
DELETE /api/books/{id}               // Soft delete
GET    /api/books/top-sellers        // Top 10 by order quantity (LINQ!)
GET    /api/books/by-genre/{genre}   // Group by genre count

// Authors
GET    /api/authors/{id}/books       // Author's books with total revenue

// Orders
POST   /api/orders                   // Place order (transaction!)
GET    /api/orders/{id}              // Order detail with items
```

### Key Concepts
- EF Core relationships (1:N, N:N via OrderItems)
- LINQ aggregation (total revenue per author)
- Transactions (order placement)
- Soft delete with global query filters
- AutoMapper or manual mapping

### Architecture

```
BookStore.Api/
├── Controllers/
├── Services/
├── Data/
│   ├── AppDbContext.cs
│   └── Migrations/
├── Models/
│   ├── Entities/
│   └── Dtos/
├── Middleware/
├── Program.cs
└── appsettings.json
```

### LINQ Challenges in This Project

```csharp
// Top 10 selling books
var topSellers = await _context.OrderItems
    .GroupBy(oi => oi.BookId)
    .Select(g => new {
        BookId = g.Key,
        TotalSold = g.Sum(oi => oi.Quantity),
        Revenue = g.Sum(oi => oi.Quantity * oi.UnitPrice)
    })
    .OrderByDescending(x => x.TotalSold)
    .Take(10)
    .Join(_context.Books.Include(b => b.Author),
          x => x.BookId,
          b => b.Id,
          (x, b) => new TopSellerDto(b.Title, b.Author.Name, x.TotalSold, x.Revenue))
    .ToListAsync();

// Author revenue report
var authorRevenue = await _context.Authors
    .Select(a => new {
        Author = a.Name,
        TotalRevenue = a.Books
            .SelectMany(b => b.OrderItems)
            .Sum(oi => oi.Quantity * oi.UnitPrice),
        BooksPublished = a.Books.Count,
        AvgPrice = a.Books.Average(b => b.Price)
    })
    .OrderByDescending(x => x.TotalRevenue)
    .ToListAsync();
```

---

## Project 4: Real-Time Chat API (SignalR)

**Goal:** Learn real-time communication with SignalR

### Requirements
- Users can join chat rooms
- Send/receive messages in real-time
- Typing indicators
- Online user list
- Message history (stored in DB)

### Key Concepts
- SignalR Hub
- WebSocket fallback
- Group management
- EF Core for message persistence

### SignalR Hub Example

```csharp
public class ChatHub : Hub {
    private readonly AppDbContext _db;
    private readonly ILogger<ChatHub> _logger;

    public ChatHub(AppDbContext db, ILogger<ChatHub> logger) {
        _db = db;
        _logger = logger;
    }

    public async Task JoinRoom(string room, string username) {
        await Groups.AddToGroupAsync(Context.ConnectionId, room);

        var msg = new Message {
            Room = room,
            Username = "System",
            Content = $"{username} joined",
            Timestamp = DateTime.UtcNow
        };
        _db.Messages.Add(msg);
        await _db.SaveChangesAsync();

        await Clients.Group(room).SendAsync("ReceiveMessage", msg.Username, msg.Content, msg.Timestamp);
    }

    public async Task SendMessage(string room, string username, string content) {
        var msg = new Message {
            Room = room,
            Username = username,
            Content = content,
            Timestamp = DateTime.UtcNow
        };
        _db.Messages.Add(msg);
        await _db.SaveChangesAsync();

        await Clients.Group(room).SendAsync("ReceiveMessage", username, content, msg.Timestamp);
    }

    public override async Task OnDisconnectedAsync(Exception? exception) {
        _logger.LogInformation("Client disconnected: {Id}", Context.ConnectionId);
        await base.OnDisconnectedAsync(exception);
    }
}
```

---

## Project 5: E-Commerce Microservices (Advanced)

**Goal:** Practice microservices architecture with .NET

### Services

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ Product API  │     │ Order API   │     │ Payment API  │
│ (EF + SQL)  │────▶│ (EF + SQL)  │────▶│ (External)   │
└─────────────┘     └─────────────┘     └─────────────┘
       │                    │
       │              ┌─────┴─────┐
       │              │ Message   │
       └──────────────│ Broker    │
                      │ (RabbitMQ)│
                      └───────────┘
```

### Key Concepts
- Multiple projects in one solution
- `dotnet new sln` + `dotnet sln add`
- Inter-service communication (HTTP / Message Queue)
- API Gateway (YARP / Ocelot)
- Containerization with Docker Compose
- Health checks

---

## Mini Exercises (15-30 mins each)

### Exercise 1: DI
```csharp
// Create this interface:
public interface INotificationService {
    Task SendAsync(string to, string message);
}

// Implement: EmailNotificationService, SmsNotificationService
// Register both with IEnumerable<INotificationService>
// Create a NotificationSender that sends via ALL registered services
```

### Exercise 2: EF Query
```csharp
// Given these entities: Student (Id, Name), Course (Id, Title), Enrollment (StudentId, CourseId, Grade)
// Write a LINQ query that:
// 1. Gets all students with their average grade
// 2. Gets the top 3 courses by enrollment count
// 3. Gets students who passed all their courses (grade >= 60)
```

### Exercise 3: LINQ Puzzle
```csharp
// Given: ["apple", "banana", "apricot", "cherry", "avocado", "blueberry"]
// 1. Group by first letter
// 2. For each group, order by length then alphabetically
// 3. Select: { Letter, Words, AverageLength }
```

### Exercise 4: Middleware
```csharp
// Create a middleware that:
// 1. Logs the request path and HTTP method
// 2. Adds a custom header "X-Processed-By" with the machine name
// 3. Times the request and logs duration
// Register it in Program.cs
```

---

## Project Checklist

Before considering yourself "production-ready":

- [ ] Can build a CRUD API from scratch without looking at docs
- [ ] Understand DI lifetimes and can spot captive dependencies
- [ ] Can write complex LINQ queries (GroupBy, Join, aggregation)
- [ ] Can configure EF Core relationships (1:1, 1:N, N:N)
- [ ] Can create and apply migrations
- [ ] Can avoid N+1 queries
- [ ] Can implement JWT auth from scratch
- [ ] Can write unit tests with Moq
- [ ] Can write integration tests with WebApplicationFactory
- [ ] Can deploy with Docker
- [ ] Understand async/await patterns and don't block with `.Result`
