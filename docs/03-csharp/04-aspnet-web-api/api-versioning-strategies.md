# API Versioning Strategies

Version APIs to evolve without breaking existing clients. Package: `Asp.Versioning.Http`.

## Why It Matters

- Breaking changes (renamed fields, changed types) require new versions
- Clients need time to migrate; parallel version support is essential
- Choosing strategy early avoids painful migration later

## Key Patterns

**URL path versioning (recommended for simplicity):**

```csharp
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true; // api-supported-versions header
});

app.MapGet("/api/v1/users", GetUsersV1).WithApiVersionSet().MapToApiVersion(1.0);
app.MapGet("/api/v2/users", GetUsersV2).WithApiVersionSet().MapToApiVersion(2.0);
```

**Combine multiple readers:**

```csharp
options.ApiVersionReader = ApiVersionReader.Combine(
    new UrlSegmentApiVersionReader(),
    new HeaderApiVersionReader("api-version"),
    new QueryStringApiVersionReader("api-version"));
```

**Deprecation:** Add `Sunset` header + `.HasDeprecatedApiVersion(1.0)`, announce 6+ months ahead.

**Breaking vs non-breaking:** Changed field types, removed fields, changed auth = breaking. Added optional fields, new endpoints = non-breaking.

## Pitfalls to Avoid

- Not versioning from day one (retrofitting is painful)
- Removing old versions without sunset period
- Treating added required fields as non-breaking (they are breaking)
- Mixing versioning strategies inconsistently across endpoints
