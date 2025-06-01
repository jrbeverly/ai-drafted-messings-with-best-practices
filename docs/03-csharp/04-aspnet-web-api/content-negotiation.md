# Content Negotiation

Clients specify preferred response format via `Accept` header. Server responds in requested format (JSON, XML, CSV, custom).

## Why It Matters

- Single endpoint serves multiple formats without separate routes
- Follows HTTP standards for client-driven format selection
- Enables export endpoints (CSV, Excel) alongside JSON APIs

## Key Patterns

```csharp
// Add XML support
builder.Services.AddControllers()
    .AddXmlSerializerFormatters();

// Return 406 Not Acceptable for unsupported formats
builder.Services.AddControllers(options =>
    options.ReturnHttpNotAcceptable = true);

// Custom output formatter (e.g., CSV)
public class CsvOutputFormatter : TextOutputFormatter
{
    public CsvOutputFormatter()
    {
        SupportedMediaTypes.Add("text/csv");
        SupportedEncodings.Add(Encoding.UTF8);
    }
    // Override CanWriteType + WriteResponseBodyAsync
}
```

**Quality values:** `Accept: application/json;q=1.0, application/xml;q=0.8` -- server picks highest quality match.

**Format via query param:** `/api/users?format=csv` as alternative to Accept header.

**Vendor media types:** `application/vnd.myapp.v1+json` for versioned content negotiation.

## Pitfalls to Avoid

- Not defaulting to JSON when no Accept header is provided
- Silently returning JSON when client asks for unsupported format (enable `ReturnHttpNotAcceptable`)
- Building custom CSV serializers with reflection without caching property info
- Forgetting to register custom formatters in `options.OutputFormatters`
