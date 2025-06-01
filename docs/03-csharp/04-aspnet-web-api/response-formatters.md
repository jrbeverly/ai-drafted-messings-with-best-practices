# Response Formatters

Custom output formatters. Serialize responses in different formats. CSV, XML, custom formats.

## Principle

Extend ASP.NET Core's content negotiation with custom formatters. Support multiple output formats from single endpoint.

## Built-in JSON Formatter

```csharp
// JSON is default
builder.Services.AddControllers()
    .AddJsonOptions(options =>
    {
        options.JsonSerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;
        options.JsonSerializerOptions.WriteIndented = true;
        options.JsonSerializerOptions.DefaultIgnoreCondition =
            JsonIgnoreCondition.WhenWritingNull;
    });
```

## Add XML Formatter

```csharp
// Support XML responses
builder.Services.AddControllers()
    .AddXmlSerializerFormatters()
    .AddXmlDataContractSerializerFormatters();

[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    [HttpGet]
    [Produces("application/json", "application/xml")]
    public IActionResult GetUsers()
    {
        var users = GetUserList();
        return Ok(users);
    }
}

// Request with Accept: application/xml
// Response:
// <?xml version="1.0" encoding="utf-8"?>
// <ArrayOfUser>
//   <User>
//     <Id>1</Id>
//     <Name>John</Name>
//   </User>
// </ArrayOfUser>
```

## Custom CSV Formatter

```csharp
// CSV output formatter
public class CsvOutputFormatter : TextOutputFormatter
{
    public CsvOutputFormatter()
    {
        SupportedMediaTypes.Add("text/csv");
        SupportedMediaTypes.Add("application/csv");

        SupportedEncodings.Add(Encoding.UTF8);
        SupportedEncodings.Add(Encoding.Unicode);
    }

    protected override bool CanWriteType(Type? type)
    {
        // Support IEnumerable<T>
        if (type == null)
            return false;

        return typeof(IEnumerable).IsAssignableFrom(type);
    }

    public override async Task WriteResponseBodyAsync(
        OutputFormatterWriteContext context,
        Encoding selectedEncoding)
    {
        var response = context.HttpContext.Response;

        if (context.Object is not IEnumerable items)
        {
            return;
        }

        var csv = new StringBuilder();
        Type? itemType = null;
        PropertyInfo[]? properties = null;

        foreach (var item in items)
        {
            if (itemType == null)
            {
                itemType = item.GetType();
                properties = itemType.GetProperties();

                // Write header
                csv.AppendLine(string.Join(",", properties.Select(p => p.Name)));
            }

            // Write row
            var values = properties!.Select(p =>
            {
                var value = p.GetValue(item);
                var stringValue = value?.ToString() ?? "";

                // Escape commas and quotes
                if (stringValue.Contains(',') || stringValue.Contains('"'))
                {
                    stringValue = $"\"{stringValue.Replace("\"", "\"\"")}\"";
                }

                return stringValue;
            });

            csv.AppendLine(string.Join(",", values));
        }

        await response.WriteAsync(csv.ToString(), selectedEncoding);
    }
}

// Register
builder.Services.AddControllers(options =>
{
    options.OutputFormatters.Add(new CsvOutputFormatter());
});

// Usage
[HttpGet]
[Produces("application/json", "text/csv")]
public IActionResult GetUsers()
{
    var users = GetUserList();
    return Ok(users);
}

// Request with Accept: text/csv
// Response:
// Id,Name,Email
// 1,John Doe,john@example.com
// 2,Jane Smith,jane@example.com
```

## Custom Binary Formatter

```csharp
// MessagePack binary formatter
public class MessagePackOutputFormatter : OutputFormatter
{
    public MessagePackOutputFormatter()
    {
        SupportedMediaTypes.Add("application/x-msgpack");
    }

    public override async Task WriteResponseBodyAsync(
        OutputFormatterWriteContext context)
    {
        var response = context.HttpContext.Response;

        // Serialize to MessagePack
        var bytes = MessagePackSerializer.Serialize(
            context.Object,
            MessagePackSerializerOptions.Standard);

        await response.Body.WriteAsync(bytes);
    }
}

// Register
builder.Services.AddControllers(options =>
{
    options.OutputFormatters.Add(new MessagePackOutputFormatter());
});
```

## YAML Formatter

```csharp
// YAML output formatter
public class YamlOutputFormatter : TextOutputFormatter
{
    private readonly ISerializer _serializer;

    public YamlOutputFormatter()
    {
        _serializer = new SerializerBuilder()
            .WithNamingConvention(CamelCaseNamingConvention.Instance)
            .Build();

        SupportedMediaTypes.Add("application/x-yaml");
        SupportedMediaTypes.Add("text/yaml");

        SupportedEncodings.Add(Encoding.UTF8);
    }

    public override async Task WriteResponseBodyAsync(
        OutputFormatterWriteContext context,
        Encoding selectedEncoding)
    {
        var yaml = _serializer.Serialize(context.Object);
        await context.HttpContext.Response.WriteAsync(yaml, selectedEncoding);
    }
}

// Request with Accept: application/x-yaml
// Response:
// users:
//   - id: 1
//     name: John Doe
//     email: john@example.com
//   - id: 2
//     name: Jane Smith
//     email: jane@example.com
```

## Excel Formatter (XLSX)

```csharp
// Excel output formatter using EPPlus
public class ExcelOutputFormatter : OutputFormatter
{
    public ExcelOutputFormatter()
    {
        SupportedMediaTypes.Add(
            "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet");
    }

    protected override bool CanWriteType(Type? type)
    {
        return type != null && typeof(IEnumerable).IsAssignableFrom(type);
    }

    public override async Task WriteResponseBodyAsync(
        OutputFormatterWriteContext context)
    {
        if (context.Object is not IEnumerable items)
        {
            return;
        }

        using var package = new ExcelPackage();
        var worksheet = package.Workbook.Worksheets.Add("Data");

        Type? itemType = null;
        PropertyInfo[]? properties = null;
        int row = 1;

        foreach (var item in items)
        {
            if (itemType == null)
            {
                itemType = item.GetType();
                properties = itemType.GetProperties();

                // Write headers
                for (int col = 0; col < properties.Length; col++)
                {
                    worksheet.Cells[row, col + 1].Value = properties[col].Name;
                }

                row++;
            }

            // Write data row
            for (int col = 0; col < properties!.Length; col++)
            {
                worksheet.Cells[row, col + 1].Value = properties[col].GetValue(item);
            }

            row++;
        }

        // Auto-fit columns
        worksheet.Cells.AutoFitColumns();

        // Write to response
        var bytes = package.GetAsByteArray();
        await context.HttpContext.Response.Body.WriteAsync(bytes);
    }
}
```

## PDF Formatter

```csharp
// PDF output formatter using iText7
public class PdfOutputFormatter : OutputFormatter
{
    public PdfOutputFormatter()
    {
        SupportedMediaTypes.Add("application/pdf");
    }

    public override async Task WriteResponseBodyAsync(
        OutputFormatterWriteContext context)
    {
        using var stream = new MemoryStream();
        using var writer = new PdfWriter(stream);
        using var pdf = new PdfDocument(writer);
        var document = new Document(pdf);

        // Add content
        document.Add(new Paragraph($"Generated: {DateTime.Now}"));

        if (context.Object is IEnumerable<User> users)
        {
            var table = new Table(3);
            table.AddHeaderCell("ID");
            table.AddHeaderCell("Name");
            table.AddHeaderCell("Email");

            foreach (var user in users)
            {
                table.AddCell(user.Id.ToString());
                table.AddCell(user.Name);
                table.AddCell(user.Email);
            }

            document.Add(table);
        }

        document.Close();

        var bytes = stream.ToArray();
        await context.HttpContext.Response.Body.WriteAsync(bytes);
    }
}
```

## Conditional Formatting

```csharp
// Different format based on user agent
public class AdaptiveOutputFormatter : TextOutputFormatter
{
    public AdaptiveOutputFormatter()
    {
        SupportedMediaTypes.Add("application/json");
        SupportedEncodings.Add(Encoding.UTF8);
    }

    public override async Task WriteResponseBodyAsync(
        OutputFormatterWriteContext context,
        Encoding selectedEncoding)
    {
        var userAgent = context.HttpContext.Request.Headers.UserAgent.ToString();

        string output;

        if (userAgent.Contains("Mobile"))
        {
            // Minimal format for mobile
            output = JsonSerializer.Serialize(context.Object, new JsonSerializerOptions
            {
                PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
                WriteIndented = false
            });
        }
        else
        {
            // Detailed format for desktop
            output = JsonSerializer.Serialize(context.Object, new JsonSerializerOptions
            {
                PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
                WriteIndented = true
            });
        }

        await context.HttpContext.Response.WriteAsync(output, selectedEncoding);
    }
}
```

## Formatter Selection Order

```csharp
// Control formatter priority
builder.Services.AddControllers(options =>
{
    // Clear default formatters
    options.OutputFormatters.Clear();

    // Add in priority order
    options.OutputFormatters.Add(new MessagePackOutputFormatter()); // Highest priority
    options.OutputFormatters.Add(new SystemTextJsonOutputFormatter(
        new JsonSerializerOptions()));
    options.OutputFormatters.Add(new XmlSerializerOutputFormatter());
    options.OutputFormatters.Add(new CsvOutputFormatter()); // Lowest priority

    // Return 406 Not Acceptable if format not supported
    options.ReturnHttpNotAcceptable = true;
});
```

## Formatter with Compression

```csharp
// Compress responses
public class CompressedJsonFormatter : TextOutputFormatter
{
    public CompressedJsonFormatter()
    {
        SupportedMediaTypes.Add("application/json");
        SupportedEncodings.Add(Encoding.UTF8);
    }

    public override async Task WriteResponseBodyAsync(
        OutputFormatterWriteContext context,
        Encoding selectedEncoding)
    {
        var json = JsonSerializer.Serialize(context.Object);

        var acceptEncoding = context.HttpContext.Request.Headers.AcceptEncoding.ToString();

        if (acceptEncoding.Contains("gzip"))
        {
            context.HttpContext.Response.Headers.ContentEncoding = "gzip";

            using var gzipStream = new GZipStream(
                context.HttpContext.Response.Body,
                CompressionMode.Compress);

            var bytes = selectedEncoding.GetBytes(json);
            await gzipStream.WriteAsync(bytes);
        }
        else
        {
            await context.HttpContext.Response.WriteAsync(json, selectedEncoding);
        }
    }
}
```

## Formatter with Streaming

```csharp
// Stream large responses
public class StreamingJsonFormatter : TextOutputFormatter
{
    public StreamingJsonFormatter()
    {
        SupportedMediaTypes.Add("application/json");
        SupportedEncodings.Add(Encoding.UTF8);
    }

    public override async Task WriteResponseBodyAsync(
        OutputFormatterWriteContext context,
        Encoding selectedEncoding)
    {
        if (context.Object is not IAsyncEnumerable<object> items)
        {
            // Fall back to regular serialization
            var json = JsonSerializer.Serialize(context.Object);
            await context.HttpContext.Response.WriteAsync(json, selectedEncoding);
            return;
        }

        // Stream items one by one
        await context.HttpContext.Response.WriteAsync("[", selectedEncoding);

        bool first = true;
        await foreach (var item in items)
        {
            if (!first)
            {
                await context.HttpContext.Response.WriteAsync(",", selectedEncoding);
            }

            var itemJson = JsonSerializer.Serialize(item);
            await context.HttpContext.Response.WriteAsync(itemJson, selectedEncoding);

            first = false;
        }

        await context.HttpContext.Response.WriteAsync("]", selectedEncoding);
    }
}
```

## Minimal API Custom Formatter

```csharp
// Custom result type for Minimal APIs
public class CsvResult : IResult
{
    private readonly IEnumerable<object> _data;

    public CsvResult(IEnumerable<object> data)
    {
        _data = data;
    }

    public async Task ExecuteAsync(HttpContext httpContext)
    {
        httpContext.Response.ContentType = "text/csv";

        var csv = new StringBuilder();
        Type? itemType = null;
        PropertyInfo[]? properties = null;

        foreach (var item in _data)
        {
            if (itemType == null)
            {
                itemType = item.GetType();
                properties = itemType.GetProperties();
                csv.AppendLine(string.Join(",", properties.Select(p => p.Name)));
            }

            var values = properties!.Select(p => p.GetValue(item)?.ToString() ?? "");
            csv.AppendLine(string.Join(",", values));
        }

        await httpContext.Response.WriteAsync(csv.ToString());
    }
}

// Extension method
public static class ResultsExtensions
{
    public static IResult Csv(this IResultExtensions _, IEnumerable<object> data)
    {
        return new CsvResult(data);
    }
}

// Usage
app.MapGet("/users/export", (IUserService userService) =>
{
    var users = userService.GetUsers();
    return Results.Extensions.Csv(users);
});
```

## Testing Formatters

```csharp
public class FormatterTests
{
    [Theory]
    [InlineData("application/json", "application/json")]
    [InlineData("application/xml", "application/xml")]
    [InlineData("text/csv", "text/csv")]
    public async Task GetUsers_AcceptHeader_ReturnsCorrectFormat(
        string accept,
        string expectedContentType)
    {
        // Arrange
        var factory = new WebApplicationFactory<Program>();
        var client = factory.CreateClient();
        client.DefaultRequestHeaders.Accept.Add(
            new MediaTypeWithQualityHeaderValue(accept));

        // Act
        var response = await client.GetAsync("/api/users");

        // Assert
        response.EnsureSuccessStatusCode();
        Assert.Contains(
            expectedContentType,
            response.Content.Headers.ContentType?.ToString());
    }

    [Fact]
    public async Task GetUsers_UnsupportedFormat_Returns406()
    {
        // Arrange
        var factory = new WebApplicationFactory<Program>()
            .WithWebHostBuilder(builder =>
            {
                builder.ConfigureServices(services =>
                {
                    services.AddControllers(options =>
                    {
                        options.ReturnHttpNotAcceptable = true;
                    });
                });
            });

        var client = factory.CreateClient();
        client.DefaultRequestHeaders.Accept.Add(
            new MediaTypeWithQualityHeaderValue("application/pdf"));

        // Act
        var response = await client.GetAsync("/api/users");

        // Assert
        Assert.Equal(HttpStatusCode.NotAcceptable, response.StatusCode);
    }
}
```

## Guidelines

**When to Create Custom Formatters:**
- Support legacy formats (SOAP, XML)
- Data export (CSV, Excel, PDF)
- Binary protocols (MessagePack, Protobuf)
- Custom domain formats

**Performance:**
- Stream large responses
- Compress when appropriate
- Cache serialization results
- Avoid reflection in hot paths

**Content Negotiation:**
- Set SupportedMediaTypes correctly
- Handle quality values (q=)
- Return 406 Not Acceptable for unsupported formats
- Document supported formats

**Best Practices:**
- Inherit from appropriate base (TextOutputFormatter, OutputFormatter)
- Implement CanWriteType correctly
- Use dependency injection for services
- Test with various Accept headers
- Support UTF-8 encoding minimum

## Benefits

Flexible. Support multiple output formats.

Client-driven. Clients choose format via Accept header.

Extensible. Add custom formats easily.

Standard. Uses HTTP content negotiation.

## Related

- [content-negotiation.md](./content-negotiation.md) - Format selection
- [input-formatters.md](./input-formatters.md) - Request formatters
- [mime-type-versioning.md](./mime-type-versioning.md) - Media type versioning
- [streaming-responses.md](./streaming-responses.md) - Streaming large data
