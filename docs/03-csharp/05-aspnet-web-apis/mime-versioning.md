# MIME-Type Versioning

API versioning via vendor-specific media types in the Accept header -- the preferred versioning approach.

## Why It Matters

- Clean, stable URLs that never change between versions
- Version is metadata about format, not part of the resource identifier
- Industry standard (GitHub, Stripe)

## Format

```
Accept: application/vnd.{vendor}.{resource}.v{version}+json
```

Example: `application/vnd.library.loan-create.v1+json`

## Version Constants

```csharp
public static class ApiVersionConstants
{
    public const string VendorPrefix = "application/vnd.library";
    public static string GetMediaType(string resourceName, int version)
        => $"{VendorPrefix}.{resourceName}.v{version}+json";
}
```

## Extension Methods

```csharp
.WithApiVersioning(ResourceName, Version)       // adds version filter
.AcceptsVersioned<Request>(ResourceName, Version) // sets Accept content type
.ProducesVersioned<Response>(ResourceName, Version) // sets response content type
```

## Version Filter

Checks the Accept header; returns **406 Not Acceptable** if the client doesn't request a supported version. Sets `Content-Type` on the response.

## Deprecation

```csharp
.AddEndpointFilter(new DeprecationFilter(DateTime.Parse("2026-12-31")));
// Adds Deprecation: true and Sunset: <date> headers
```

## Key Recommendations

- Major versions only (v1, v2, v3) -- breaking changes require a new version
- Maintain old versions for 6+ months with documented migration paths
- Resource names in kebab-case: `loan-create`, `user-update`
- Return 406 for unsupported versions; allow `application/json` and `*/*` as fallbacks

## Avoid

- URL versioning (`/api/v1/`, `/api/v2/`)
- Query string versioning (`?version=2`)
- Custom headers (`X-API-Version: 2`)
- Sharing code between version implementations

## Related

- [endpoint-organization.md](./endpoint-organization.md) -- versioned namespace structure
- [minimal-api-routing.md](./minimal-api-routing.md) -- route registration
