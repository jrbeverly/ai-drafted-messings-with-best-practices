# Input Formatters

Custom input formatters. Parse request bodies in different formats. CSV, XML, custom formats.

## Principle

Extend model binding to support custom request formats. Parse non-JSON request bodies. Type-safe deserialization.

## Built-in JSON Formatter

```csharp
// JSON is default
builder.Services.AddControllers()
    .AddJsonOptions(options =>
    {
        options.JsonSerializerOptions.PropertyNameCaseInsensitive = true;
        options.JsonSerializerOptions.AllowTrailingCommas = true;
    });

[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    [HttpPost]
    public IActionResult CreateUser([FromBody] CreateUserRequest request)
    {
        // Request body automatically deserialized from JSON
        var user = CreateNewUser(request);
        return CreatedAtAction(nameof(GetUser), new { id = user.Id }, user);
    }
}
```

## Add XML Input Formatter

```csharp
// Support XML request bodies
builder.Services.AddControllers()
    .AddXmlSerializerFormatters();

[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    [HttpPost]
    [Consumes("application/json", "application/xml")]
    public IActionResult CreateUser([FromBody] CreateUserRequest request)
    {
        // Accepts both JSON and XML
        var user = CreateNewUser(request);
        return Ok(user);
    }
}

// Request with Content-Type: application/xml
// <CreateUserRequest>
//   <Email>user@example.com</Email>
//   <Name>John Doe</Name>
// </CreateUserRequest>
```

## Custom CSV Input Formatter

```csharp
// CSV input formatter
public class CsvInputFormatter : TextInputFormatter
{
    public CsvInputFormatter()
    {
        SupportedMediaTypes.Add("text/csv");
        SupportedMediaTypes.Add("application/csv");

        SupportedEncodings.Add(Encoding.UTF8);
        SupportedEncodings.Add(Encoding.Unicode);
    }

    protected override bool CanReadType(Type type)
    {
        // Support IEnumerable<T>
        return typeof(IEnumerable).IsAssignableFrom(type);
    }

    public override async Task<InputFormatterResult> ReadRequestBodyAsync(
        InputFormatterContext context,
        Encoding effectiveEncoding)
    {
        var request = context.HttpContext.Request;

        using var reader = new StreamReader(request.Body, effectiveEncoding);
        var csv = await reader.ReadToEndAsync();

        var lines = csv.Split('\n', StringSplitOptions.RemoveEmptyEntries);

        if (lines.Length < 2)
        {
            return await InputFormatterResult.FailureAsync();
        }

        // Parse header
        var headers = lines[0].Split(',').Select(h => h.Trim()).ToArray();

        // Parse rows
        var items = new List<Dictionary<string, string>>();

        for (int i = 1; i < lines.Length; i++)
        {
            var values = lines[i].Split(',');
            var item = new Dictionary<string, string>();

            for (int j = 0; j < headers.Length && j < values.Length; j++)
            {
                item[headers[j]] = values[j].Trim();
            }

            items.Add(item);
        }

        // Convert to target type
        var modelType = context.ModelType;
        var genericType = modelType.GetGenericArguments().FirstOrDefault();

        if (genericType != null)
        {
            var typedItems = items.Select(item =>
            {
                var instance = Activator.CreateInstance(genericType);

                foreach (var prop in genericType.GetProperties())
                {
                    if (item.TryGetValue(prop.Name, out var value))
                    {
                        var convertedValue = Convert.ChangeType(value, prop.PropertyType);
                        prop.SetValue(instance, convertedValue);
                    }
                }

                return instance;
            }).ToList();

            var listType = typeof(List<>).MakeGenericType(genericType);
            var list = Activator.CreateInstance(listType) as IList;

            foreach (var item in typedItems)
            {
                list?.Add(item);
            }

            return await InputFormatterResult.SuccessAsync(list!);
        }

        return await InputFormatterResult.FailureAsync();
    }
}

// Register
builder.Services.AddControllers(options =>
{
    options.InputFormatters.Add(new CsvInputFormatter());
});

// Usage
[HttpPost("batch")]
[Consumes("text/csv")]
public IActionResult CreateUsers([FromBody] List<CreateUserRequest> users)
{
    foreach (var user in users)
    {
        CreateNewUser(user);
    }

    return Ok(new { Count = users.Count });
}

// Request with Content-Type: text/csv
// Email,Name,Age
// john@example.com,John Doe,30
// jane@example.com,Jane Smith,25
```

## MessagePack Input Formatter

```csharp
// Binary MessagePack formatter
public class MessagePackInputFormatter : InputFormatter
{
    public MessagePackInputFormatter()
    {
        SupportedMediaTypes.Add("application/x-msgpack");
    }

    public override async Task<InputFormatterResult> ReadRequestBodyAsync(
        InputFormatterContext context)
    {
        var request = context.HttpContext.Request;

        using var ms = new MemoryStream();
        await request.Body.CopyToAsync(ms);
        ms.Position = 0;

        try
        {
            var result = await MessagePackSerializer.DeserializeAsync(
                context.ModelType,
                ms,
                MessagePackSerializerOptions.Standard);

            return await InputFormatterResult.SuccessAsync(result);
        }
        catch
        {
            return await InputFormatterResult.FailureAsync();
        }
    }
}

// Register
builder.Services.AddControllers(options =>
{
    options.InputFormatters.Add(new MessagePackInputFormatter());
});
```

## YAML Input Formatter

```csharp
// YAML input formatter
public class YamlInputFormatter : TextInputFormatter
{
    private readonly IDeserializer _deserializer;

    public YamlInputFormatter()
    {
        _deserializer = new DeserializerBuilder()
            .WithNamingConvention(CamelCaseNamingConvention.Instance)
            .Build();

        SupportedMediaTypes.Add("application/x-yaml");
        SupportedMediaTypes.Add("text/yaml");

        SupportedEncodings.Add(Encoding.UTF8);
    }

    public override async Task<InputFormatterResult> ReadRequestBodyAsync(
        InputFormatterContext context,
        Encoding effectiveEncoding)
    {
        var request = context.HttpContext.Request;

        using var reader = new StreamReader(request.Body, effectiveEncoding);
        var yaml = await reader.ReadToEndAsync();

        try
        {
            var result = _deserializer.Deserialize(yaml, context.ModelType);
            return await InputFormatterResult.SuccessAsync(result);
        }
        catch
        {
            return await InputFormatterResult.FailureAsync();
        }
    }
}

// Request with Content-Type: application/x-yaml
// email: user@example.com
// name: John Doe
// age: 30
```

## Form URL Encoded Formatter

```csharp
// Custom form URL encoded formatter
public class CustomFormUrlEncodedInputFormatter : InputFormatter
{
    public CustomFormUrlEncodedInputFormatter()
    {
        SupportedMediaTypes.Add("application/x-www-form-urlencoded");
    }

    public override async Task<InputFormatterResult> ReadRequestBodyAsync(
        InputFormatterContext context)
    {
        var request = context.HttpContext.Request;

        var form = await request.ReadFormAsync();

        var instance = Activator.CreateInstance(context.ModelType);

        foreach (var prop in context.ModelType.GetProperties())
        {
            if (form.TryGetValue(prop.Name, out var value))
            {
                var convertedValue = Convert.ChangeType(value.ToString(), prop.PropertyType);
                prop.SetValue(instance, convertedValue);
            }
        }

        return await InputFormatterResult.SuccessAsync(instance!);
    }
}
```

## Multipart Form Data Formatter

```csharp
// Custom multipart form data formatter
public class MultipartFormDataFormatter : InputFormatter
{
    public MultipartFormDataFormatter()
    {
        SupportedMediaTypes.Add("multipart/form-data");
    }

    public override async Task<InputFormatterResult> ReadRequestBodyAsync(
        InputFormatterContext context)
    {
        var request = context.HttpContext.Request;

        if (!request.HasFormContentType)
        {
            return await InputFormatterResult.FailureAsync();
        }

        var form = await request.ReadFormAsync();

        // Custom multipart parsing logic
        var instance = Activator.CreateInstance(context.ModelType);

        foreach (var prop in context.ModelType.GetProperties())
        {
            // Handle files
            if (prop.PropertyType == typeof(IFormFile))
            {
                var file = form.Files.GetFile(prop.Name);
                prop.SetValue(instance, file);
            }
            // Handle regular form fields
            else if (form.TryGetValue(prop.Name, out var value))
            {
                var convertedValue = Convert.ChangeType(value.ToString(), prop.PropertyType);
                prop.SetValue(instance, convertedValue);
            }
        }

        return await InputFormatterResult.SuccessAsync(instance!);
    }
}
```

## Protobuf Input Formatter

```csharp
// Protocol Buffers formatter
public class ProtobufInputFormatter : InputFormatter
{
    public ProtobufInputFormatter()
    {
        SupportedMediaTypes.Add("application/x-protobuf");
    }

    public override async Task<InputFormatterResult> ReadRequestBodyAsync(
        InputFormatterContext context)
    {
        var request = context.HttpContext.Request;

        try
        {
            using var ms = new MemoryStream();
            await request.Body.CopyToAsync(ms);
            ms.Position = 0;

            var result = Serializer.Deserialize(context.ModelType, ms);

            return await InputFormatterResult.SuccessAsync(result);
        }
        catch
        {
            return await InputFormatterResult.FailureAsync();
        }
    }
}
```

## Compressed Input Formatter

```csharp
// Decompress gzipped requests
public class GzipInputFormatter : InputFormatter
{
    private readonly InputFormatter _innerFormatter;

    public GzipInputFormatter(InputFormatter innerFormatter)
    {
        _innerFormatter = innerFormatter;

        foreach (var mediaType in innerFormatter.SupportedMediaTypes)
        {
            SupportedMediaTypes.Add(mediaType);
        }
    }

    public override async Task<InputFormatterResult> ReadRequestBodyAsync(
        InputFormatterContext context)
    {
        var request = context.HttpContext.Request;

        if (request.Headers.ContentEncoding == "gzip")
        {
            using var decompressed = new MemoryStream();
            using var gzip = new GZipStream(request.Body, CompressionMode.Decompress);

            await gzip.CopyToAsync(decompressed);
            decompressed.Position = 0;

            // Replace request body with decompressed stream
            var originalBody = request.Body;
            request.Body = decompressed;

            var result = await _innerFormatter.ReadRequestBodyAsync(context);

            request.Body = originalBody;

            return result;
        }

        return await _innerFormatter.ReadRequestBodyAsync(context);
    }
}

// Wrap JSON formatter
builder.Services.AddControllers(options =>
{
    var jsonFormatter = options.InputFormatters
        .OfType<SystemTextJsonInputFormatter>()
        .First();

    options.InputFormatters.Insert(0, new GzipInputFormatter(jsonFormatter));
});
```

## Encrypted Input Formatter

```csharp
// Decrypt encrypted request bodies
public class EncryptedInputFormatter : InputFormatter
{
    private readonly IEncryptionService _encryptionService;
    private readonly InputFormatter _innerFormatter;

    public EncryptedInputFormatter(
        IEncryptionService encryptionService,
        InputFormatter innerFormatter)
    {
        _encryptionService = encryptionService;
        _innerFormatter = innerFormatter;

        SupportedMediaTypes.Add("application/encrypted+json");
    }

    public override async Task<InputFormatterResult> ReadRequestBodyAsync(
        InputFormatterContext context)
    {
        var request = context.HttpContext.Request;

        using var reader = new StreamReader(request.Body);
        var encrypted = await reader.ReadToEndAsync();

        // Decrypt
        var decrypted = _encryptionService.Decrypt(encrypted);

        // Create stream from decrypted data
        var decryptedStream = new MemoryStream(Encoding.UTF8.GetBytes(decrypted));

        // Replace request body
        var originalBody = request.Body;
        request.Body = decryptedStream;

        var result = await _innerFormatter.ReadRequestBodyAsync(context);

        request.Body = originalBody;

        return result;
    }
}
```

## Formatter Selection Order

```csharp
// Control formatter priority
builder.Services.AddControllers(options =>
{
    // Clear defaults
    options.InputFormatters.Clear();

    // Add in priority order
    options.InputFormatters.Add(new MessagePackInputFormatter()); // Highest priority
    options.InputFormatters.Add(new SystemTextJsonInputFormatter(
        new JsonSerializerOptions(),
        new Logger<SystemTextJsonInputFormatter>(new LoggerFactory())));
    options.InputFormatters.Add(new XmlSerializerInputFormatter(options));
    options.InputFormatters.Add(new CsvInputFormatter()); // Lowest priority
});
```

## Minimal API Custom Parsing

```csharp
// Custom request parsing for Minimal APIs
app.MapPost("/users/csv", async (HttpContext context) =>
{
    if (context.Request.ContentType != "text/csv")
    {
        return Results.BadRequest("Expected CSV content");
    }

    using var reader = new StreamReader(context.Request.Body);
    var csv = await reader.ReadToEndAsync();

    var users = ParseCsv(csv);

    foreach (var user in users)
    {
        await CreateUser(user);
    }

    return Results.Ok(new { Count = users.Count });
});

static List<CreateUserRequest> ParseCsv(string csv)
{
    var lines = csv.Split('\n', StringSplitOptions.RemoveEmptyEntries);
    var users = new List<CreateUserRequest>();

    // Skip header
    for (int i = 1; i < lines.Length; i++)
    {
        var parts = lines[i].Split(',');

        if (parts.Length >= 2)
        {
            users.Add(new CreateUserRequest
            {
                Email = parts[0].Trim(),
                Name = parts[1].Trim()
            });
        }
    }

    return users;
}
```

## Testing Input Formatters

```csharp
public class InputFormatterTests
{
    [Fact]
    public async Task CreateUser_CsvContent_Parses()
    {
        // Arrange
        var factory = new WebApplicationFactory<Program>();
        var client = factory.CreateClient();

        var csv = "Email,Name\njohn@example.com,John Doe\njane@example.com,Jane Smith";
        var content = new StringContent(csv, Encoding.UTF8, "text/csv");

        // Act
        var response = await client.PostAsync("/api/users/batch", content);

        // Assert
        response.EnsureSuccessStatusCode();

        var result = await response.Content.ReadFromJsonAsync<BatchResult>();
        Assert.Equal(2, result!.Count);
    }

    [Fact]
    public async Task CreateUser_XmlContent_Parses()
    {
        // Arrange
        var factory = new WebApplicationFactory<Program>();
        var client = factory.CreateClient();

        var xml = "<CreateUserRequest><Email>user@example.com</Email><Name>John</Name></CreateUserRequest>";
        var content = new StringContent(xml, Encoding.UTF8, "application/xml");

        // Act
        var response = await client.PostAsync("/api/users", content);

        // Assert
        response.EnsureSuccessStatusCode();
    }

    [Fact]
    public async Task CreateUser_InvalidCsv_Returns400()
    {
        // Arrange
        var factory = new WebApplicationFactory<Program>();
        var client = factory.CreateClient();

        var invalidCsv = "Not,Valid,CSV\n"; // Only header
        var content = new StringContent(invalidCsv, Encoding.UTF8, "text/csv");

        // Act
        var response = await client.PostAsync("/api/users/batch", content);

        // Assert
        Assert.Equal(HttpStatusCode.BadRequest, response.StatusCode);
    }
}
```

## Guidelines

**When to Create Custom Input Formatters:**
- Support legacy formats (SOAP, EDI)
- Binary protocols (MessagePack, Protobuf)
- Custom file formats (CSV, Excel)
- Encrypted/compressed payloads

**Security:**
- Validate content before parsing
- Set maximum request size
- Sanitize input data
- Handle malformed input gracefully

**Performance:**
- Stream large payloads
- Avoid loading entire body in memory
- Use async operations
- Cache deserializers

**Error Handling:**
- Return InputFormatterResult.Failure for invalid input
- Log parsing errors
- Provide helpful error messages
- Don't expose sensitive details

**Best Practices:**
- Set SupportedMediaTypes correctly
- Implement CanReadType for type checking
- Use dependency injection for services
- Test with various Content-Type headers
- Support UTF-8 encoding minimum

## Benefits

Flexible. Support multiple input formats.

Type-safe. Automatic deserialization.

Extensible. Add custom formats easily.

Standard. Uses HTTP Content-Type header.

## Related

- [response-formatters.md](./response-formatters.md) - Output formatters
- [model-binding.md](./model-binding.md) - Parameter binding
- [content-negotiation.md](./content-negotiation.md) - Format selection
- [file-upload-aspnet.md](./file-upload-aspnet.md) - File uploads
