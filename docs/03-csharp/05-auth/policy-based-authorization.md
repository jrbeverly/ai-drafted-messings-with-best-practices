# Policy-Based Authorization

Encapsulate complex authorization rules (beyond simple role checks) into reusable, testable policies.

## Why It Matters

- Policies are reusable across endpoints and composable (AND multiple requirements)
- Custom handlers support async logic (DB lookups, feature flags, subscription checks)
- Separates authorization logic from endpoint code, making both testable independently

## Key Recommendations

- **Basic policies**: Combine built-in requirements

```csharp
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("VerifiedPremiumUser", policy =>
    {
        policy.RequireAuthenticatedUser();
        policy.RequireClaim("email_verified", "true");
        policy.RequireClaim("subscription_tier", "Premium", "Enterprise");
    });
});
```

- **Custom requirement + handler**: Implement `IAuthorizationRequirement` and `AuthorizationHandler<T>`

```csharp
public class MinimumAgeRequirement(int minimumAge) : IAuthorizationRequirement
{
    public int MinimumAge { get; } = minimumAge;
}

// Handler: context.Succeed(requirement) only when check passes
// Register: builder.Services.AddSingleton<IAuthorizationHandler, MinimumAgeHandler>();
```

- **Async handlers**: Inject services via DI for DB-backed checks (e.g., `ISubscriptionService`)
- **OR logic**: Use `RequireAssertion` for inline expressions

```csharp
options.AddPolicy("AdminOrOwner", policy =>
    policy.RequireAssertion(ctx => ctx.User.IsInRole("Admin") || ctx.User.HasClaim("role", "Owner")));
```

- **Default/fallback policies**: Set `options.DefaultPolicy` and `options.FallbackPolicy`
- **Failure reasons**: Use `context.Fail(new AuthorizationFailureReason(...))` for debugging
- **Resource-based**: Use `AuthorizationHandler<TRequirement, TResource>` with `IAuthorizationService.AuthorizeAsync`

## Testing Pattern

```csharp
var handler = new MinimumAgeHandler();
var user = new ClaimsPrincipal(new ClaimsIdentity(new[] { new Claim("date_of_birth", "2000-01-01") }));
var context = new AuthorizationHandlerContext(new[] { requirement }, user, null);
await handler.HandleRequirementAsync(context, requirement);
Assert.True(context.HasSucceeded);
```

## Pitfalls to Avoid

- Calling `context.Succeed()` without verifying the condition (fail by default)
- Expensive DB queries in handlers without caching
- Putting authorization logic directly in endpoints instead of reusable handlers
- Forgetting to register handlers in DI (`AddSingleton` for stateless, `AddScoped` for async/DB)
- Confusing multiple `RequireRole` calls (each call is AND; values within one call are OR)
