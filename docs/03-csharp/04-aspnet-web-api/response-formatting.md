# Response Formatting

Control how API responses are serialized. JSON configuration, content negotiation, custom formatters.

## Principle

Consistent format. Configure once, apply everywhere. Support multiple formats when needed.

## JSON Configuration

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.ConfigureHttpJsonOptions(options =>
{
    // Property naming
    options.SerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;

    // Null handling
    options.SerializerOptions.DefaultIgnoreCondition =
        JsonIgnoreCondition.WhenWritingNull;

    // Indentation (development only)
    options.SerializerOptions.WriteIndented =
        builder.Environment.IsDevelopment();

    // Allow trailing commas
    options.SerializerOptions.AllowTrailingCommas = true;

    // Case-insensitive property matching
    options.SerializerOptions.PropertyNameCaseInsensitive = true;

    // Enum handling
    options.SerializerOptions.Converters.Add(
        new JsonStringEnumConverter(JsonNamingPolicy.CamelCase));

    // Number handling
    options.SerializerOptions.NumberHandling =
        JsonNumberHandling.AllowReadingFromString;
});
```

## Response Types

```csharp
// JSON response (default)
app.MapGet("/users", () =>
{
    var users = new[] { new { Id = 1, Name = "John" } };
    return Results.Ok(users);
});

// Plain text
app.MapGet("/health", () =>
    Results.Text("Healthy", "text/plain", statusCode: 200));

// HTML
app.MapGet("/about", () =>
    Results.Content("<h1>About</h1>", "text/html"));

// File download
app.MapGet("/export", () =>
{
    var bytes = Encoding.UTF8.GetBytes("CSV data");
    return Results.File(bytes, "text/csv", "export.csv");
});

// Stream
app.MapGet("/large-file", async () =>
{
    var stream = File.OpenRead("large-file.zip");
    return Results.Stream(stream, "application/zip", "download.zip");
});
```

## Custom JSON Serializer Options

```csharp
// Response with custom options
app.MapGet("/custom-format", () =>
{
    var data = new { Name = "John", Age = 30 };

    var options = new JsonSerializerOptions
    {
        PropertyNamingPolicy = JsonNamingPolicy.SnakeCaseLower,
        WriteIndented = true
    };

    return Results.Json(data, options, statusCode: 200);
});

// Response:
// {
//   "name": "John",
//   "age": 30
// }
```

## Source-Generated JSON

```csharp
// Define serialization context
[JsonSerializable(typeof(User))]
[JsonSerializable(typeof(User[]))]
[JsonSerializable(typeof(ApiResponse<User>))]
[JsonSourceGenerationOptions(
    PropertyNamingPolicy = JsonKnownNamingPolicy.CamelCase,
    DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull)]
internal partial class AppJsonSerializerContext : JsonSerializerContext
{
}

// Configure
builder.Services.ConfigureHttpJsonOptions(options =>
{
    options.SerializerOptions.TypeInfoResolverChain.Insert(0,
        AppJsonSerializerContext.Default);
});

// Use in endpoint
app.MapGet("/users", () =>
{
    var users = new[] { new User { Id = "1", Name = "John" } };
    return Results.Json(users, AppJsonSerializerContext.Default.UserArray);
});
```

## Content Negotiation

```csharp
app.MapGet("/data", (HttpContext context) =>
{
    var acceptHeader = context.Request.Headers.Accept.ToString();

    var data = new { Name = "John", Age = 30 };

    if (acceptHeader.Contains("application/xml"))
    {
        var xml = SerializeToXml(data);
        return Results.Content(xml, "application/xml");
    }

    if (acceptHeader.Contains("text/csv"))
    {
        var csv = SerializeToCsv(data);
        return Results.Text(csv, "text/csv");
    }

    // Default to JSON
    return Results.Json(data);
});
```

## Response Wrapper

```csharp
public record ApiResponse<T>
{
    public bool Success { get; init; }
    public T? Data { get; init; }
    public string? Error { get; init; }
    public DateTime Timestamp { get; init; } = DateTime.UtcNow;
}

// Success response
app.MapGet("/users/{id}", async (
    string id,
    IUserService userService) =>
{
    var user = await userService.GetUserByIdAsync(id);

    if (user is null)
    {
        return Results.NotFound(new ApiResponse<User>
        {
            Success = false,
            Error = "User not found"
        });
    }

    return Results.Ok(new ApiResponse<User>
    {
        Success = true,
        Data = user
    });
});

// Extension method
public static class ResultsExtensions
{
    public static IResult SuccessResponse<T>(this IResultExtensions _, T data)
    {
        return Results.Ok(new ApiResponse<T>
        {
            Success = true,
            Data = data
        });
    }

    public static IResult ErrorResponse(this IResultExtensions _, string error, int statusCode = 400)
    {
        return Results.Json(
            new ApiResponse<object>
            {
                Success = false,
                Error = error
            },
            statusCode: statusCode);
    }
}

// Usage
app.MapGet("/users/{id}", async (string id, IUserService userService) =>
{
    var user = await userService.GetUserByIdAsync(id);

    return user is not null
        ? Results.Extensions.SuccessResponse(user)
        : Results.Extensions.ErrorResponse("User not found", 404);
});
```

## Pagination Response

```csharp
public record PagedResponse<T>
{
    public required T[] Items { get; init; }
    public required int Page { get; init; }
    public required int Limit { get; init; }
    public required int TotalCount { get; init; }
    public required int TotalPages { get; init; }
    public required bool HasMore { get; init; }
}

app.MapGet("/users", async (
    int page = 1,
    int limit = 20,
    IUserService userService) =>
{
    var (users, totalCount) = await userService.GetUsersPagedAsync(page, limit);

    var totalPages = (int)Math.Ceiling(totalCount / (double)limit);

    var response = new PagedResponse<User>
    {
        Items = users,
        Page = page,
        Limit = limit,
        TotalCount = totalCount,
        TotalPages = totalPages,
        HasMore = page < totalPages
    };

    return Results.Ok(response);
});
```

## Hypermedia (HATEOAS)

```csharp
public record UserResponse
{
    public required string Id { get; init; }
    public required string Name { get; init; }
    public required string Email { get; init; }
    public Dictionary<string, Link> Links { get; init; } = new();
}

public record Link(string Href, string Method, string Rel);

app.MapGet("/users/{id}", async (
    string id,
    IUserService userService,
    HttpContext context) =>
{
    var user = await userService.GetUserByIdAsync(id);

    if (user is null)
    {
        return Results.NotFound();
    }

    var baseUrl = $"{context.Request.Scheme}://{context.Request.Host}";

    var response = new UserResponse
    {
        Id = user.Id,
        Name = user.Name,
        Email = user.Email,
        Links = new()
        {
            ["self"] = new Link($"{baseUrl}/users/{id}", "GET", "self"),
            ["update"] = new Link($"{baseUrl}/users/{id}", "PUT", "update"),
            ["delete"] = new Link($"{baseUrl}/users/{id}", "DELETE", "delete"),
            ["loans"] = new Link($"{baseUrl}/users/{id}/loans", "GET", "loans")
        }
    };

    return Results.Ok(response);
});
```

## ETag Support

```csharp
app.MapGet("/users/{id}", async (
    string id,
    HttpContext context,
    IUserService userService) =>
{
    var user = await userService.GetUserByIdAsync(id);

    if (user is null)
    {
        return Results.NotFound();
    }

    // Generate ETag from content hash
    var json = JsonSerializer.Serialize(user);
    var hash = Convert.ToBase64String(
        SHA256.HashData(Encoding.UTF8.GetBytes(json)));
    var etag = $"\"{hash}\"";

    // Check If-None-Match header
    var ifNoneMatch = context.Request.Headers.IfNoneMatch.ToString();

    if (ifNoneMatch == etag)
    {
        return Results.StatusCode(304); // Not Modified
    }

    // Add ETag to response
    context.Response.Headers.ETag = etag;

    return Results.Ok(user);
});
```

## Response Headers

```csharp
app.MapGet("/users", async (
    HttpContext context,
    IUserService userService) =>
{
    var users = await userService.GetUsersAsync();

    // Add custom headers
    context.Response.Headers.Add("X-Total-Count", users.Length.ToString());
    context.Response.Headers.Add("X-Page-Size", "20");
    context.Response.Headers.Add("X-Response-Time", "45ms");

    // Cache control
    context.Response.Headers.CacheControl = "public, max-age=300";

    // CORS
    context.Response.Headers.AccessControlExposeHeaders = "X-Total-Count, X-Page-Size";

    return Results.Ok(users);
});
```

## Conditional Responses

```csharp
app.MapGet("/users/{id}", async (
    string id,
    HttpContext context,
    IUserService userService) =>
{
    var user = await userService.GetUserByIdAsync(id);

    if (user is null)
    {
        return Results.NotFound();
    }

    // Check If-Modified-Since
    var ifModifiedSince = context.Request.Headers.IfModifiedSince.ToString();

    if (!string.IsNullOrEmpty(ifModifiedSince) &&
        DateTime.TryParse(ifModifiedSince, out var sinceDate))
    {
        if (user.UpdatedAt <= sinceDate)
        {
            return Results.StatusCode(304); // Not Modified
        }
    }

    // Add Last-Modified header
    context.Response.Headers.LastModified =
        user.UpdatedAt.ToString("R"); // RFC 1123 format

    return Results.Ok(user);
});
```

## Custom JSON Converters

```csharp
// DateTime converter (ISO 8601)
public class Iso8601DateTimeConverter : JsonConverter<DateTime>
{
    public override DateTime Read(
        ref Utf8JsonReader reader,
        Type typeToConvert,
        JsonSerializerOptions options)
    {
        return DateTime.Parse(reader.GetString()!);
    }

    public override void Write(
        Utf8JsonWriter writer,
        DateTime value,
        JsonSerializerOptions options)
    {
        writer.WriteStringValue(value.ToString("O")); // ISO 8601
    }
}

// Register converter
builder.Services.ConfigureHttpJsonOptions(options =>
{
    options.SerializerOptions.Converters.Add(new Iso8601DateTimeConverter());
});
```

## Response Compression

```csharp
builder.Services.AddResponseCompression(options =>
{
    options.EnableForHttps = true;
    options.Providers.Add<BrotliCompressionProvider>();
    options.Providers.Add<GzipCompressionProvider>();

    options.MimeTypes = ResponseCompressionDefaults.MimeTypes.Concat(
        new[] { "application/json", "text/json" });
});

builder.Services.Configure<BrotliCompressionProviderOptions>(options =>
{
    options.Level = CompressionLevel.Fastest;
});

app.UseResponseCompression();

// Responses automatically compressed when:
// - Response size > 1KB
// - Client supports compression (Accept-Encoding header)
// - Content-Type matches configured types
```

## Guidelines

**JSON Configuration:**
- Consistent naming (camelCase for JavaScript clients)
- Ignore null values to reduce payload size
- Indent only in development
- Use source generators for AOT

**Content Types:**
- Default to JSON for APIs
- Support content negotiation if needed
- Use appropriate MIME types
- Document supported formats

**Response Structure:**
- Consistent format across endpoints
- Include metadata (timestamps, request IDs)
- Wrap responses if beneficial
- Follow REST conventions

**Performance:**
- Enable response compression
- Use ETags for caching
- Support conditional requests
- Stream large responses

## Benefits

Consistent. Same format everywhere.

Flexible. Multiple content types.

Performant. Compression, caching, streaming.

Standard. Follow HTTP/REST conventions.

## Related

- [minimal-api-basics.md](./minimal-api-basics.md) - Endpoint definition
- [api-versioning.md](../../08-api-design/api-versioning.md) - Version-specific formats
- [rest-api-principles.md](../../08-api-design/rest-api-principles.md) - REST conventions
- [performance-optimization.md](../../10-performance/performance-optimization.md) - Serialization performance
