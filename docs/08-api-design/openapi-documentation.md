# OpenAPI Documentation

API documentation with OpenAPI (Swagger). Auto-generated from code. Interactive documentation.

## Principle

Documentation from code. Keep it up-to-date. Include examples. Interactive and testable.

## Setup

Install Swashbuckle:

```bash
dotnet add package Swashbuckle.AspNetCore
```

Configure in Program.cs:

```csharp
var builder = WebApplication.CreateBuilder(args);

// Add OpenAPI/Swagger
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "Library API",
        Version = "v1",
        Description = "RESTful API for library management system",
        Contact = new OpenApiContact
        {
            Name = "Support",
            Email = "support@example.com",
            Url = new Uri("https://example.com/support")
        },
        License = new OpenApiLicense
        {
            Name = "MIT",
            Url = new Uri("https://opensource.org/licenses/MIT")
        }
    });

    // Include XML comments
    var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
    var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
    options.IncludeXmlComments(xmlPath);
});

var app = builder.Build();

// Enable Swagger middleware
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI(options =>
    {
        options.SwaggerEndpoint("/swagger/v1/swagger.json", "Library API v1");
        options.RoutePrefix = string.Empty;  // Swagger UI at root
    });
}

app.Run();
```

Enable XML documentation:

```xml
<!-- LibraryService.Api.csproj -->
<PropertyGroup>
  <GenerateDocumentationFile>true</GenerateDocumentationFile>
  <NoWarn>$(NoWarn);1591</NoWarn> <!-- Disable missing XML comment warnings -->
</PropertyGroup>
```

## Endpoint Documentation

Document endpoints with XML comments:

```csharp
/// <summary>
/// Get user by ID
/// </summary>
/// <param name="id">User ID</param>
/// <returns>User details</returns>
/// <response code="200">Returns the user</response>
/// <response code="404">User not found</response>
public static async Task<IResult> GetUser(
    string id,
    IUserService userService)
{
    var user = await userService.GetUserByIdAsync(id);

    if (user == null)
        return Results.NotFound(new { error = "User not found" });

    return Results.Ok(user);
}

// Register with metadata
app.MapGet("/users/{id}", GetUser)
    .WithName("GetUser")
    .WithSummary("Get user by ID")
    .WithDescription("Retrieves a single user by their unique identifier")
    .Produces<UserResponse>(StatusCodes.Status200OK)
    .Produces<ProblemDetails>(StatusCodes.Status404NotFound)
    .WithTags("Users");
```

## Request/Response Examples

Add examples to DTOs:

```csharp
/// <summary>
/// Create user request
/// </summary>
public record CreateUserRequest
{
    /// <summary>
    /// User's email address
    /// </summary>
    /// <example>john.doe@example.com</example>
    [Required]
    [EmailAddress]
    public string Email { get; init; } = string.Empty;

    /// <summary>
    /// User's full name
    /// </summary>
    /// <example>John Doe</example>
    [Required]
    [MinLength(2)]
    public string Name { get; init; } = string.Empty;

    /// <summary>
    /// User's phone number (optional)
    /// </summary>
    /// <example>+1234567890</example>
    public string? PhoneNumber { get; init; }
}

/// <summary>
/// User response
/// </summary>
public record UserResponse
{
    /// <summary>
    /// Unique user identifier
    /// </summary>
    /// <example>user_123abc</example>
    public required string Id { get; init; }

    /// <summary>
    /// User's email address
    /// </summary>
    /// <example>john.doe@example.com</example>
    public required string Email { get; init; }

    /// <summary>
    /// User's full name
    /// </summary>
    /// <example>John Doe</example>
    public required string Name { get; init; }

    /// <summary>
    /// Account creation timestamp
    /// </summary>
    /// <example>2024-01-15T10:30:00Z</example>
    public required DateTime CreatedAt { get; init; }
}
```

## Operation Metadata

Add operation details:

```csharp
app.MapPost("/users", CreateUser)
    .WithName("CreateUser")
    .WithSummary("Create a new user")
    .WithDescription(@"
Creates a new user account with the provided information.
Email must be unique across the system.
Returns the created user with generated ID.")
    .Accepts<CreateUserRequest>("application/json")
    .Produces<UserResponse>(StatusCodes.Status201Created)
    .Produces<ValidationProblemDetails>(StatusCodes.Status400BadRequest)
    .Produces<ProblemDetails>(StatusCodes.Status409Conflict)
    .WithTags("Users")
    .WithOpenApi(operation =>
    {
        // Customize operation
        operation.OperationId = "CreateUser";
        operation.Deprecated = false;

        // Add response headers
        operation.Responses["201"].Headers.Add("Location", new OpenApiHeader
        {
            Description = "URL of the created user",
            Schema = new OpenApiSchema { Type = "string" }
        });

        return operation;
    });
```

## Authentication

Document authentication:

```csharp
builder.Services.AddSwaggerGen(options =>
{
    // JWT Bearer authentication
    options.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Description = "JWT Authorization header using the Bearer scheme. Enter 'Bearer' [space] and then your token.",
        Name = "Authorization",
        In = ParameterLocation.Header,
        Type = SecuritySchemeType.ApiKey,
        Scheme = "Bearer"
    });

    options.AddSecurityRequirement(new OpenApiSecurityRequirement
    {
        {
            new OpenApiSecurityScheme
            {
                Reference = new OpenApiReference
                {
                    Type = ReferenceType.SecurityScheme,
                    Id = "Bearer"
                }
            },
            Array.Empty<string>()
        }
    });
});

// Mark endpoint as requiring authentication
app.MapGet("/users/me", GetCurrentUser)
    .RequireAuthorization()
    .WithSummary("Get current user")
    .Produces<UserResponse>(StatusCodes.Status200OK)
    .Produces(StatusCodes.Status401Unauthorized);
```

## Grouping Endpoints

Organize with tags:

```csharp
// Group by tag
var users = app.MapGroup("/users").WithTags("Users");
var loans = app.MapGroup("/loans").WithTags("Loans");
var books = app.MapGroup("/books").WithTags("Books");

users.MapGet("/", GetUsers);
users.MapGet("/{id}", GetUser);
users.MapPost("/", CreateUser);

loans.MapGet("/", GetLoans);
loans.MapGet("/{id}", GetLoan);
loans.MapPost("/", CreateLoan);

// Configure tag descriptions
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo { /* ... */ });

    options.TagActionsBy(api =>
    {
        if (api.GroupName != null)
            return new[] { api.GroupName };

        if (api.ActionDescriptor is RouteEndpointActionDescriptor descriptor)
        {
            var tags = descriptor.Metadata.OfType<TagsAttribute>().FirstOrDefault();
            if (tags != null)
                return tags.Tags;
        }

        return new[] { "Default" };
    });

    options.DocInclusionPredicate((name, api) => true);
});
```

## Multiple Versions

Document multiple API versions:

```csharp
builder.Services.AddSwaggerGen(options =>
{
    // V1
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "Library API v1",
        Version = "v1",
        Description = "Version 1 of the Library API (Deprecated)",
        Deprecated = true
    });

    // V2
    options.SwaggerDoc("v2", new OpenApiInfo
    {
        Title = "Library API v2",
        Version = "v2",
        Description = "Version 2 of the Library API (Current)"
    });

    // Filter endpoints by version
    options.DocInclusionPredicate((docName, apiDesc) =>
    {
        if (apiDesc.RelativePath == null)
            return false;

        return apiDesc.RelativePath.Contains($"/{docName}/");
    });
});

app.UseSwaggerUI(options =>
{
    options.SwaggerEndpoint("/swagger/v1/swagger.json", "Library API v1");
    options.SwaggerEndpoint("/swagger/v2/swagger.json", "Library API v2");
});
```

## Custom Schemas

Define custom schema mappings:

```csharp
builder.Services.AddSwaggerGen(options =>
{
    // Map custom types
    options.MapType<DateTime>(() => new OpenApiSchema
    {
        Type = "string",
        Format = "date-time",
        Example = new OpenApiString("2024-01-15T10:30:00Z")
    });

    options.MapType<DateOnly>(() => new OpenApiSchema
    {
        Type = "string",
        Format = "date",
        Example = new OpenApiString("2024-01-15")
    });

    // Add schema filter
    options.SchemaFilter<EnumSchemaFilter>();
});

// Enum schema filter
public class EnumSchemaFilter : ISchemaFilter
{
    public void Apply(OpenApiSchema schema, SchemaFilterContext context)
    {
        if (context.Type.IsEnum)
        {
            schema.Enum.Clear();
            foreach (var name in Enum.GetNames(context.Type))
            {
                schema.Enum.Add(new OpenApiString(name));
            }
        }
    }
}
```

## Operation Filters

Customize operations:

```csharp
// Add request ID header to all operations
public class RequestIdOperationFilter : IOperationFilter
{
    public void Apply(OpenApiOperation operation, OperationFilterContext context)
    {
        operation.Parameters.Add(new OpenApiParameter
        {
            Name = "X-Request-ID",
            In = ParameterLocation.Header,
            Required = false,
            Description = "Optional request ID for tracking",
            Schema = new OpenApiSchema { Type = "string" }
        });

        // Add request ID to all responses
        foreach (var response in operation.Responses.Values)
        {
            response.Headers.Add("X-Request-ID", new OpenApiHeader
            {
                Description = "Request tracking ID",
                Schema = new OpenApiSchema { Type = "string" }
            });
        }
    }
}

// Register
builder.Services.AddSwaggerGen(options =>
{
    options.OperationFilter<RequestIdOperationFilter>();
});
```

## Code Generation

Generate client code from OpenAPI:

```bash
# Install NSwag CLI
dotnet tool install -g NSwag.ConsoleCore

# Generate TypeScript client
nswag openapi2tsclient \
  /input:https://api.example.com/swagger/v1/swagger.json \
  /output:src/api/generated/api-client.ts \
  /generateClientClasses:true \
  /generateDtoTypes:true \
  /className:ApiClient

# Generate C# client
nswag openapi2csclient \
  /input:https://api.example.com/swagger/v1/swagger.json \
  /output:src/ApiClient/Generated/ApiClient.cs \
  /namespace:LibraryService.ApiClient \
  /generateClientClasses:true \
  /generateDtoTypes:true
```

Generated TypeScript client usage:

```typescript
import { ApiClient, UserResponse, CreateUserRequest } from './api/generated/api-client'

const client = new ApiClient('https://api.example.com')

// Create user
const request: CreateUserRequest = {
  email: 'john@example.com',
  name: 'John Doe'
}

const user: UserResponse = await client.createUser(request)

// Get user
const user2: UserResponse = await client.getUser('user_123')
```

## Export OpenAPI Spec

Export specification file:

```bash
# Run API
dotnet run

# Download spec
curl https://localhost:5001/swagger/v1/swagger.json > openapi.json

# Or use Swagger CLI
swagger-cli validate openapi.json
```

CI/CD integration:

```yaml
# .github/workflows/openapi.yml
name: OpenAPI

on:
  push:
    branches: [main]

jobs:
  export-spec:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: 8.0.x

      - name: Build
        run: dotnet build src/LibraryService/LibraryService.Api

      - name: Export OpenAPI spec
        run: |
          dotnet run --project src/LibraryService/LibraryService.Api &
          sleep 10
          curl https://localhost:5001/swagger/v1/swagger.json > openapi.json
          kill %1

      - name: Validate spec
        run: npx @apidevtools/swagger-cli validate openapi.json

      - name: Upload artifact
        uses: actions/upload-artifact@v3
        with:
          name: openapi-spec
          path: openapi.json
```

## Custom UI

Customize Swagger UI:

```csharp
app.UseSwaggerUI(options =>
{
    options.SwaggerEndpoint("/swagger/v1/swagger.json", "Library API v1");

    // Custom CSS
    options.InjectStylesheet("/swagger-ui/custom.css");

    // Custom JavaScript
    options.InjectJavascript("/swagger-ui/custom.js");

    // UI settings
    options.DocExpansion(DocExpansion.List);
    options.DefaultModelsExpandDepth(-1);  // Hide models section
    options.DisplayRequestDuration();
    options.EnableDeepLinking();
    options.EnableFilter();
});

// Serve custom files
app.UseStaticFiles(new StaticFileOptions
{
    FileProvider = new PhysicalFileProvider(
        Path.Combine(Directory.GetCurrentDirectory(), "wwwroot", "swagger-ui")),
    RequestPath = "/swagger-ui"
});
```

## Guidelines

**Documentation:**
- Document all public endpoints
- Include request/response examples
- Describe error responses
- Keep documentation up-to-date

**Schemas:**
- Use XML comments for descriptions
- Add examples to DTOs
- Document required fields
- Explain validation rules

**Organization:**
- Group endpoints with tags
- Use meaningful operation IDs
- Consistent naming conventions
- Logical endpoint ordering

**Authentication:**
- Document auth requirements
- Include auth examples
- Explain token format
- Document scopes/permissions

**Versioning:**
- Separate docs per version
- Mark deprecated versions
- Link to migration guides
- Explain breaking changes

## Benefits

Auto-generated. Documentation from code.

Up-to-date. Changes reflected immediately.

Interactive. Test endpoints in browser.

Client generation. Auto-generate API clients.

## Related

- [rest-api-principles.md](./rest-api-principles.md) - API design patterns
- [api-versioning.md](./api-versioning.md) - Version management
- [api-error-handling.md](./api-error-handling.md) - Error handling
- [minimal-api.md](../03-csharp/01-core/minimal-api.md) - Implementation patterns
