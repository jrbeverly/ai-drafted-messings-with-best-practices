# Required Members

C# 11 required keyword enforces property initialization at compile time.

## Principle

Required members must be initialized when creating object. Compiler enforces completeness.

## Basic Required Properties

Enforce initialization:

```csharp
public class User
{
    public required string Name { get; init; }
    public required string Email { get; init; }
    public string? PhoneNumber { get; init; }
}

// Must initialize required properties
var user = new User
{
    Name = "John Doe",
    Email = "john@example.com"
    // PhoneNumber is optional
};

// Compiler error - missing required properties
var invalid = new User(); // Error: Required member 'Name' must be set
```

## With Records

Required members in records:

```csharp
public record User
{
    public required string Name { get; init; }
    public required string Email { get; init; }
    public DateOnly? DateOfBirth { get; init; }
}

var user = new User
{
    Name = "John Doe",
    Email = "john@example.com"
};
```

## Required vs Constructor

Two approaches for enforcing initialization:

```csharp
// Constructor approach
public class User
{
    public string Name { get; init; }
    public string Email { get; init; }

    public User(string name, string email)
    {
        Name = name;
        Email = email;
    }
}

var user1 = new User("John", "john@example.com");

// Required approach
public class User
{
    public required string Name { get; init; }
    public required string Email { get; init; }
}

var user2 = new User
{
    Name = "John",
    Email = "john@example.com"
};
```

**Constructor Approach:**
- Positional, ordered arguments
- Best for small, stable sets of properties
- Good for domain entities

**Required Approach:**
- Named properties, any order
- Best for DTOs with many properties
- Easier to extend without breaking changes

## Combining Required and Constructor

Use both together:

```csharp
public class User
{
    public required string Name { get; init; }
    public required string Email { get; init; }
    public string? PhoneNumber { get; init; }
    public DateTime CreatedAt { get; init; }

    public User()
    {
        CreatedAt = DateTime.UtcNow; // Set default in constructor
    }
}

var user = new User
{
    Name = "John",
    Email = "john@example.com"
    // CreatedAt set automatically
};
```

## SetsRequiredMembers Attribute

Constructor satisfies required members:

```csharp
public class User
{
    public required string Name { get; init; }
    public required string Email { get; init; }

    [SetsRequiredMembers]
    public User(string name, string email)
    {
        Name = name;
        Email = email;
    }
}

// Both valid
var user1 = new User("John", "john@example.com");

var user2 = new User
{
    Name = "John",
    Email = "john@example.com"
};
```

## API Request DTOs

Required members for API contracts:

```csharp
namespace LibraryService.Routes.Users.v1;

public static class UserCreateRoute
{
    public record Request
    {
        [Required]
        [EmailAddress]
        public required string Email { get; init; }

        [Required]
        [MinLength(2)]
        public required string Name { get; init; }

        [Phone]
        public string? PhoneNumber { get; init; }
    }

    public record Response
    {
        public required string UserId { get; init; }
        public required string Email { get; init; }
        public required string Name { get; init; }
    }
}

// Client code must provide all required fields
var request = new UserCreateRoute.Request
{
    Email = "john@example.com",
    Name = "John Doe"
    // PhoneNumber optional
};
```

## Validation Order

Required members checked before DataAnnotations:

```csharp
public record Request
{
    [Required] // Redundant, but explicit for validation message
    [EmailAddress]
    public required string Email { get; init; }
}

// Compiler error first (required not set)
var invalid1 = new Request(); // Compile error

// DataAnnotations validation second (invalid email)
var invalid2 = new Request { Email = "not-an-email" }; // Runtime validation error
```

## Inheritance

Required members propagate to derived classes:

```csharp
public class Person
{
    public required string Name { get; init; }
}

public class Employee : Person
{
    public required string EmployeeId { get; init; }
    public string? Department { get; init; }
}

// Must set both Name and EmployeeId
var employee = new Employee
{
    Name = "John Doe",
    EmployeeId = "EMP001"
};
```

## Required vs Nullable

Required non-nullable enforces both initialization and non-null:

```csharp
public class User
{
    // Required and non-nullable
    public required string Email { get; init; }

    // Required but nullable (must set, can be null)
    public required string? OptionalEmail { get; init; }

    // Not required, nullable
    public string? PhoneNumber { get; init; }
}

var user = new User
{
    Email = "john@example.com",
    OptionalEmail = null // Must set, but can be null
};
```

## JSON Deserialization

Required members work with System.Text.Json:

```csharp
public record User
{
    public required string Name { get; init; }
    public required string Email { get; init; }
}

// JSON must include required properties
var json = """
{
  "Name": "John Doe",
  "Email": "john@example.com"
}
""";

var user = JsonSerializer.Deserialize<User>(json); // Success

var invalidJson = """{ "Name": "John" }"""; // Missing Email

var invalid = JsonSerializer.Deserialize<User>(invalidJson); // JsonException
```

## Guidelines

**When to Use Required:**
- API request/response DTOs
- Configuration objects
- Domain entities with mandatory properties
- Any object where missing properties would be invalid

**When to Use Constructor:**
- Small, stable property sets
- Positional data (coordinates, tuples)
- Domain value objects
- When order matters

**Combining:**
- Use required for mandatory business properties
- Use constructor for defaults and computed values
- Use optional properties for truly optional data

**With Nullable Reference Types:**
- Required + non-nullable: Must set, cannot be null
- Required + nullable: Must set, can be null
- Optional + nullable: May omit, can be null

## Benefits

Compile-time safety. Missing required properties caught by compiler.

Clear intent. Required keyword documents mandatory properties.

Better defaults. No more null or empty strings as defaults.

## Related

- [nullable-reference-types.md](./nullable-reference-types.md) - Nullability annotations
- [record-types.md](./record-types.md) - Records with required members
- [../05-aspnet-web-apis/request-response-pattern.md](../05-aspnet-web-apis/request-response-pattern.md) - DTOs
