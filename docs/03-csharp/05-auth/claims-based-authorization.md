# Claims-Based Authorization

Fine-grained authorization using user claims (facts about the user) rather than coarse role checks.

## Why It Matters

- More granular than roles -- authorize on any user attribute (email verified, subscription tier, permissions)
- Claims travel inside the JWT, enabling stateless authorization decisions
- Namespace-based permissions (`resource:action`) scale cleanly across large systems

## Key Recommendations

- **Basic claim policies**: Use `RequireClaim` for simple checks

```csharp
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("EmailVerified", policy => policy.RequireClaim("email_verified", "true"));
    options.AddPolicy("PremiumUser", policy => policy.RequireClaim("subscription_tier", "Premium", "Enterprise"));
});
```

- **Permission claims**: Namespace as `resource:action` (e.g., `users:read`, `documents:write`)

```csharp
// Add permission claims to JWT
foreach (var permission in user.Permissions)
    claims.Add(new Claim("permission", permission));

// Check in code
var hasPermission = user.HasClaim("permission", "users:read");
```

- **Custom requirement handlers**: For complex logic (ANY-of, wildcard `users:*`, hierarchical)
- **Claims transformation**: Use `IClaimsTransformation` to enrich tokens with DB-loaded permissions at request time
- **Tenant isolation**: Include `tenant_id` claim, extract via middleware, scope all queries
- **Feature flags as claims**: Gate endpoints with `RequireClaim("feature", "beta-features")`

## Inline Claim Checks

```csharp
var permissions = user.FindAll("permission").Select(c => c.Value);
if (!permissions.Contains("documents:write")) return Results.Forbid();
```

## Pitfalls to Avoid

- Putting sensitive data in claims (tokens are Base64, not encrypted -- visible to client)
- Too many claims bloating token size (keep tokens small, use claims transformation for large permission sets)
- Trusting client-sent claims without server-side validation
- Forgetting to cache `IClaimsTransformation` results (runs on every request)
- Using custom claim names when standard `ClaimTypes.*` exist
