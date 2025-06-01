# Role-Based Authorization

Restrict endpoint access based on user roles -- the simplest and most common authorization model in ASP.NET Core.

## Why It Matters

- Built into ASP.NET Core with zero extra dependencies
- Covers the majority of authorization scenarios (admin vs. user vs. moderator)
- Easy to reason about, test, and audit

## Key Recommendations

- **Define role constants**: Avoid magic strings

```csharp
public static class Roles
{
    public const string Admin = "Admin";
    public const string User = "User";
    public const string Moderator = "Moderator";
}
```

- **Apply to endpoints**: Single role or OR logic (any of the listed roles)

```csharp
app.MapGet("/admin/users", GetAllUsers)
    .RequireAuthorization(policy => policy.RequireRole(Roles.Admin));

app.MapGet("/mod/reports", GetReports)
    .RequireAuthorization(policy => policy.RequireRole(Roles.Admin, Roles.Moderator));
```

- **AND logic**: Chain multiple `.RequireRole()` calls in a policy (user must have ALL roles)
- **Named policies**: Register once, apply by name across endpoints

```csharp
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("RequireAdmin", policy => policy.RequireRole(Roles.Admin));
    options.AddPolicy("RequireModerator", policy => policy.RequireRole(Roles.Admin, Roles.Moderator));
});
```

- **JWT role claims**: Add `ClaimTypes.Role` for each role when generating tokens
- **Inline checks**: `user.IsInRole(Roles.Admin)` for conditional logic within endpoints
- **Response filtering**: Admins see all fields; regular users see a subset
- **HTTP status codes**: 401 = not authenticated, 403 = authenticated but insufficient role

## Role Hierarchy (Optional)

```csharp
// Map roles to levels: Guest(0) < User(1) < Moderator(2) < Admin(3)
// Check: userLevel >= requiredLevel
```

## Pitfalls to Avoid

- Using magic role strings scattered across endpoints (use constants)
- Creating too many roles (complexity grows; consider claims/policies instead)
- Trusting client-sent role claims without server-side validation
- Returning 403 when the user is not authenticated (should be 401)
- Not auditing role assignment changes
- Confusing OR vs AND: `.RequireRole("A", "B")` = OR; chained `.RequireRole("A").RequireRole("B")` = AND
