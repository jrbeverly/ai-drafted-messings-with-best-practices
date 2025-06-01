# With Expressions

C# 9 with expressions create modified copies of immutable objects. Non-destructive mutation for records and classes.

## Principle

Create new instance with modified properties. Original instance unchanged. Enables immutable data patterns.

## Basic Syntax

Modify properties in copy:

```csharp
public record User
{
    public string Name { get; init; }
    public string Email { get; init; }
    public int Age { get; init; }
}

var user1 = new User
{
    Name = "John Doe",
    Email = "john@example.com",
    Age = 30
};

// Create copy with modified age
var user2 = user1 with { Age = 31 };

// user1 unchanged
Console.WriteLine(user1.Age); // 30
Console.WriteLine(user2.Age); // 31
```

## Multiple Properties

Modify multiple properties:

```csharp
var user1 = new User
{
    Name = "John Doe",
    Email = "john@example.com",
    Age = 30
};

var user2 = user1 with
{
    Name = "Jane Doe",
    Email = "jane@example.com"
};

// Only Age unchanged from user1
Console.WriteLine(user2.Name);  // Jane Doe
Console.WriteLine(user2.Email); // jane@example.com
Console.WriteLine(user2.Age);   // 30
```

## With Records

Natural fit for records:

```csharp
public record Address(string Street, string City, string State, string Zip);

var address1 = new Address("123 Main St", "Seattle", "WA", "98101");

// Update just the street
var address2 = address1 with { Street = "456 Oak Ave" };

// Other properties copied
Console.WriteLine(address2.City);  // Seattle
Console.WriteLine(address2.State); // WA
Console.WriteLine(address2.Zip);   // 98101
```

## With Classes

Works with classes having init properties:

```csharp
public class Product
{
    public string Name { get; init; } = string.Empty;
    public decimal Price { get; init; }
    public int Stock { get; init; }
}

var product1 = new Product
{
    Name = "Widget",
    Price = 10.00m,
    Stock = 100
};

// Reduce stock
var product2 = product1 with { Stock = 99 };
```

## Nested With

Chain multiple with expressions:

```csharp
var user = new User { Name = "John", Email = "john@example.com", Age = 30 };

var updated = user
    with { Age = 31 }
    with { Email = "john.doe@example.com" };

// Equivalent to:
var updated2 = user with
{
    Age = 31,
    Email = "john.doe@example.com"
};
```

## Domain Updates

Update domain entities immutably:

```csharp
public record Loan
{
    public required string Id { get; init; }
    public required string UserId { get; init; }
    public required string BookId { get; init; }
    public required DateTime DueDate { get; init; }
    public required LoanStatus Status { get; init; }
}

public enum LoanStatus { Active, Returned, Overdue }

// Return a loan
public Loan ReturnLoan(Loan loan)
{
    return loan with
    {
        Status = LoanStatus.Returned
    };
}

// Extend due date
public Loan ExtendLoan(Loan loan, int additionalDays)
{
    return loan with
    {
        DueDate = loan.DueDate.AddDays(additionalDays)
    };
}
```

## API Response Updates

Modify responses:

```csharp
public record UserResponse
{
    public required string UserId { get; init; }
    public required string Name { get; init; }
    public required string Email { get; init; }
    public DateTime? LastLogin { get; init; }
}

public UserResponse AddLastLogin(UserResponse response, DateTime lastLogin)
{
    return response with { LastLogin = lastLogin };
}
```

## State Transitions

Immutable state machine:

```csharp
public record Order
{
    public required string OrderId { get; init; }
    public required OrderStatus Status { get; init; }
    public required decimal Total { get; init; }
    public DateTime? ShippedAt { get; init; }
    public DateTime? DeliveredAt { get; init; }
}

public enum OrderStatus { Pending, Confirmed, Shipped, Delivered }

public Order ConfirmOrder(Order order)
{
    if (order.Status != OrderStatus.Pending)
        throw new InvalidOperationException("Can only confirm pending orders");

    return order with { Status = OrderStatus.Confirmed };
}

public Order ShipOrder(Order order)
{
    if (order.Status != OrderStatus.Confirmed)
        throw new InvalidOperationException("Can only ship confirmed orders");

    return order with
    {
        Status = OrderStatus.Shipped,
        ShippedAt = DateTime.UtcNow
    };
}

public Order DeliverOrder(Order order)
{
    if (order.Status != OrderStatus.Shipped)
        throw new InvalidOperationException("Can only deliver shipped orders");

    return order with
    {
        Status = OrderStatus.Delivered,
        DeliveredAt = DateTime.UtcNow
    };
}
```

## Test Data

Build test objects incrementally:

```csharp
[Fact]
public void ProcessOrder_ValidOrder_Succeeds()
{
    // Base test object
    var baseOrder = new Order
    {
        OrderId = "order_123",
        Status = OrderStatus.Pending,
        Total = 100.00m
    };

    // Create variations
    var confirmedOrder = baseOrder with { Status = OrderStatus.Confirmed };
    var shippedOrder = confirmedOrder with
    {
        Status = OrderStatus.Shipped,
        ShippedAt = DateTime.UtcNow
    };

    // Test with different states
    Assert.Equal(OrderStatus.Pending, baseOrder.Status);
    Assert.Equal(OrderStatus.Confirmed, confirmedOrder.Status);
    Assert.Equal(OrderStatus.Shipped, shippedOrder.Status);
}
```

## Configuration Updates

Override configuration:

```csharp
public record DatabaseConfig
{
    public required string TableName { get; init; }
    public required string Region { get; init; }
    public int TimeoutSeconds { get; init; } = 30;
}

// Default config
var defaultConfig = new DatabaseConfig
{
    TableName = "Users",
    Region = "us-west-2"
};

// Override for testing
var testConfig = defaultConfig with
{
    TableName = "Users_Test",
    TimeoutSeconds = 5
};

// Override for production
var prodConfig = defaultConfig with
{
    Region = "us-east-1",
    TimeoutSeconds = 60
};
```

## Versioned Updates

Track changes with timestamps:

```csharp
public record UserProfile
{
    public required string UserId { get; init; }
    public required string DisplayName { get; init; }
    public required string Bio { get; init; }
    public DateTime UpdatedAt { get; init; }
}

public UserProfile UpdateProfile(UserProfile profile, string newDisplayName, string newBio)
{
    return profile with
    {
        DisplayName = newDisplayName,
        Bio = newBio,
        UpdatedAt = DateTime.UtcNow
    };
}
```

## Nested Objects

Modify nested properties:

```csharp
public record Address(string Street, string City, string State);
public record Person(string Name, Address Address);

var person = new Person(
    "John Doe",
    new Address("123 Main St", "Seattle", "WA")
);

// Update nested address
var updated = person with
{
    Address = person.Address with { Street = "456 Oak Ave" }
};

Console.WriteLine(updated.Name);           // John Doe
Console.WriteLine(updated.Address.Street); // 456 Oak Ave
Console.WriteLine(updated.Address.City);   // Seattle
```

## Copy and Replace Pattern

Common service pattern:

```csharp
public class UserService : IUserService
{
    private readonly IUserRepository _repository;

    public async Task<User> UpdateEmailAsync(string userId, string newEmail)
    {
        var user = await _repository.GetByIdAsync(userId)
            ?? throw new NotFoundException("User", userId);

        // Create updated copy
        var updatedUser = user with { Email = newEmail };

        // Save and return
        await _repository.SaveAsync(updatedUser);
        return updatedUser;
    }
}
```

## Guidelines

**Use With Expressions:**
- Immutable data updates
- State transitions
- Domain entity modifications
- Test data variations
- Configuration overrides

**Benefits:**
- Original unchanged (immutability)
- Clear intent (copy-and-modify pattern)
- Thread-safe (no shared mutable state)
- Readable (declarative updates)

**Combine With:**
- Records (primary use case)
- Init-only properties
- Required members
- Pattern matching

**Performance:**
- Creates new instance (allocation)
- Acceptable for most cases
- Avoid in tight loops with large objects

## Comparison

Traditional vs with expressions:

```csharp
// Traditional - mutable
public class User
{
    public string Name { get; set; }
    public int Age { get; set; }
}

var user = new User { Name = "John", Age = 30 };
user.Age = 31; // Mutates in place

// With expressions - immutable
public record User
{
    public string Name { get; init; }
    public int Age { get; init; }
}

var user1 = new User { Name = "John", Age = 30 };
var user2 = user1 with { Age = 31 }; // Creates new instance
```

## Benefits

Immutability. Original objects never change.

Clear intent. Copy-and-modify pattern is explicit.

Thread safety. Immutable objects are inherently thread-safe.

Functional programming. Enables functional-style updates.

## Related

- [record-types.md](./record-types.md) - Records with with expressions
- [init-only-properties.md](./init-only-properties.md) - Init properties
- [pattern-matching.md](./pattern-matching.md) - Destructuring with patterns
