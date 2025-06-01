# Authorization

Role-based and policy-based authorization. Protect resources. Enforce permissions.

## Principle

Principle of least privilege. Explicit permissions. Resource-level authorization. Deny by default.

## Role-Based Authorization

Define roles:

```csharp
public static class Roles
{
    public const string Admin = "Admin";
    public const string User = "User";
    public const string Guest = "Guest";
}

// Add role claim when generating token
var claims = new[]
{
    new Claim(ClaimTypes.NameIdentifier, user.Id),
    new Claim(ClaimTypes.Email, user.Email),
    new Claim(ClaimTypes.Role, user.Role)  // "Admin", "User", etc.
};
```

Require role on endpoint:

```csharp
// Single role
app.MapDelete("/users/{id}", DeleteUser)
    .RequireAuthorization(policy => policy.RequireRole(Roles.Admin));

// Multiple roles (OR)
app.MapGet("/admin/dashboard", GetAdminDashboard)
    .RequireAuthorization(policy => policy.RequireRole(Roles.Admin, Roles.SuperAdmin));

// Using [Authorize] attribute
[Authorize(Roles = Roles.Admin)]
public static async Task<IResult> DeleteUser(
    string id,
    IUserService userService)
{
    await userService.DeleteUserAsync(id);
    return Results.NoContent();
}
```

## Policy-Based Authorization

Define policies:

```csharp
// Program.cs
builder.Services.AddAuthorization(options =>
{
    // Simple policy
    options.AddPolicy("RequireAdmin", policy =>
        policy.RequireRole(Roles.Admin));

    // Multiple requirements
    options.AddPolicy("RequireEmailVerified", policy =>
        policy.RequireClaim("email_verified", "true"));

    // Custom requirement
    options.AddPolicy("CanManageUsers", policy =>
        policy.Requirements.Add(new ManageUsersRequirement()));

    // Combine requirements
    options.AddPolicy("AdminWithVerifiedEmail", policy =>
    {
        policy.RequireRole(Roles.Admin);
        policy.RequireClaim("email_verified", "true");
    });
});

// Use policy
app.MapDelete("/users/{id}", DeleteUser)
    .RequireAuthorization("RequireAdmin");
```

## Custom Authorization Requirements

Create custom requirement:

```csharp
// Requirement
public class ManageUsersRequirement : IAuthorizationRequirement
{
}

// Handler
public class ManageUsersAuthorizationHandler
    : AuthorizationHandler<ManageUsersRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        ManageUsersRequirement requirement)
    {
        // Check if user has permission
        var hasPermission = context.User.HasClaim(c =>
            c.Type == "permission" &&
            c.Value == "users:manage");

        if (hasPermission || context.User.IsInRole(Roles.Admin))
        {
            context.Succeed(requirement);
        }

        return Task.CompletedTask;
    }
}

// Register
builder.Services.AddSingleton<IAuthorizationHandler, ManageUsersAuthorizationHandler>();
```

## Resource-Based Authorization

Check ownership:

```csharp
// Authorization service
public interface IResourceAuthorizationService
{
    Task<bool> CanAccessAsync(string userId, string resourceId);
    Task<bool> CanModifyAsync(string userId, string resourceId);
    Task<bool> CanDeleteAsync(string userId, string resourceId);
}

public class ResourceAuthorizationService : IResourceAuthorizationService
{
    private readonly IUserRepository _userRepository;
    private readonly IResourceRepository _resourceRepository;

    public async Task<bool> CanAccessAsync(string userId, string resourceId)
    {
        var user = await _userRepository.GetByIdAsync(userId);
        var resource = await _resourceRepository.GetByIdAsync(resourceId);

        if (user == null || resource == null)
            return false;

        // Admin can access everything
        if (user.Role == Roles.Admin)
            return true;

        // Owner can access their own resources
        if (resource.OwnerId == userId)
            return true;

        // Check shared resources
        return resource.SharedWith.Contains(userId);
    }

    public async Task<bool> CanModifyAsync(string userId, string resourceId)
    {
        var user = await _userRepository.GetByIdAsync(userId);
        var resource = await _resourceRepository.GetByIdAsync(resourceId);

        if (user == null || resource == null)
            return false;

        // Only admin and owner can modify
        return user.Role == Roles.Admin || resource.OwnerId == userId;
    }

    public async Task<bool> CanDeleteAsync(string userId, string resourceId)
    {
        var user = await _userRepository.GetByIdAsync(userId);
        var resource = await _resourceRepository.GetByIdAsync(resourceId);

        if (user == null || resource == null)
            return false;

        // Only admin and owner can delete
        return user.Role == Roles.Admin || resource.OwnerId == userId;
    }
}

// Use in endpoint
public static async Task<IResult> UpdateLoan(
    string id,
    HttpContext context,
    UpdateLoanRequest request,
    ILoanService loanService,
    IResourceAuthorizationService authService)
{
    var userId = context.User.FindFirst(ClaimTypes.NameIdentifier)?.Value
        ?? throw new UnauthorizedAccessException();

    // Check authorization
    if (!await authService.CanModifyAsync(userId, id))
    {
        return Results.Problem(
            type: "https://api.example.com/errors/forbidden",
            title: "Forbidden",
            status: 403,
            detail: "You do not have permission to modify this resource");
    }

    var loan = await loanService.UpdateLoanAsync(id, request);
    return Results.Ok(loan);
}
```

## Permission-Based Authorization

Fine-grained permissions:

```csharp
public static class Permissions
{
    // Users
    public const string UsersRead = "users:read";
    public const string UsersWrite = "users:write";
    public const string UsersDelete = "users:delete";

    // Loans
    public const string LoansRead = "loans:read";
    public const string LoansWrite = "loans:write";
    public const string LoansDelete = "loans:delete";

    // Admin
    public const string AdminAccess = "admin:access";
}

// Add permissions to token
public string GenerateAccessToken(User user)
{
    var permissions = GetUserPermissions(user.Role);

    var claims = new List<Claim>
    {
        new Claim(ClaimTypes.NameIdentifier, user.Id),
        new Claim(ClaimTypes.Email, user.Email),
        new Claim(ClaimTypes.Role, user.Role)
    };

    // Add permission claims
    foreach (var permission in permissions)
    {
        claims.Add(new Claim("permission", permission));
    }

    // Generate token...
}

private string[] GetUserPermissions(string role)
{
    return role switch
    {
        Roles.Admin => new[]
        {
            Permissions.UsersRead,
            Permissions.UsersWrite,
            Permissions.UsersDelete,
            Permissions.LoansRead,
            Permissions.LoansWrite,
            Permissions.LoansDelete,
            Permissions.AdminAccess
        },
        Roles.User => new[]
        {
            Permissions.UsersRead,
            Permissions.LoansRead,
            Permissions.LoansWrite
        },
        _ => Array.Empty<string>()
    };
}

// Permission requirement
public class HasPermissionRequirement : IAuthorizationRequirement
{
    public HasPermissionRequirement(string permission)
    {
        Permission = permission;
    }

    public string Permission { get; }
}

public class HasPermissionHandler : AuthorizationHandler<HasPermissionRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        HasPermissionRequirement requirement)
    {
        var hasPermission = context.User.HasClaim(c =>
            c.Type == "permission" &&
            c.Value == requirement.Permission);

        if (hasPermission)
        {
            context.Succeed(requirement);
        }

        return Task.CompletedTask;
    }
}

// Register
builder.Services.AddSingleton<IAuthorizationHandler, HasPermissionHandler>();

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("CanDeleteUsers", policy =>
        policy.Requirements.Add(new HasPermissionRequirement(Permissions.UsersDelete)));
});

// Use
app.MapDelete("/users/{id}", DeleteUser)
    .RequireAuthorization("CanDeleteUsers");
```

## Authorization Filter

Reusable authorization filter:

```csharp
public class ResourceAuthorizationFilter : IEndpointFilter
{
    private readonly IResourceAuthorizationService _authService;

    public ResourceAuthorizationFilter(IResourceAuthorizationService authService)
    {
        _authService = authService;
    }

    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context,
        EndpointFilterDelegate next)
    {
        var httpContext = context.HttpContext;
        var userId = httpContext.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;

        if (userId == null)
            return Results.Unauthorized();

        // Get resource ID from route
        var resourceId = httpContext.GetRouteValue("id")?.ToString();

        if (resourceId == null)
            return Results.BadRequest(new { error = "Resource ID required" });

        // Check authorization based on HTTP method
        var authorized = httpContext.Request.Method switch
        {
            "GET" => await _authService.CanAccessAsync(userId, resourceId),
            "PUT" or "PATCH" => await _authService.CanModifyAsync(userId, resourceId),
            "DELETE" => await _authService.CanDeleteAsync(userId, resourceId),
            _ => false
        };

        if (!authorized)
        {
            return Results.Problem(
                type: "https://api.example.com/errors/forbidden",
                title: "Forbidden",
                status: 403,
                detail: "You do not have permission to access this resource");
        }

        return await next(context);
    }
}

// Extension method
public static RouteHandlerBuilder WithResourceAuthorization(
    this RouteHandlerBuilder builder)
{
    return builder.AddEndpointFilter<ResourceAuthorizationFilter>();
}

// Usage
app.MapPut("/loans/{id}", UpdateLoan)
    .RequireAuthorization()
    .WithResourceAuthorization();
```

## Scope-Based Authorization

OAuth scopes:

```csharp
// Define scopes
public static class Scopes
{
    public const string ReadUsers = "read:users";
    public const string WriteUsers = "write:users";
    public const string ReadLoans = "read:loans";
    public const string WriteLoans = "write:loans";
}

// Add to token
var claims = new List<Claim>
{
    new Claim(ClaimTypes.NameIdentifier, user.Id),
    new Claim("scope", $"{Scopes.ReadUsers} {Scopes.WriteUsers}")
};

// Require scope
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("RequireReadUsers", policy =>
        policy.RequireClaim("scope", scope =>
            scope.Contains(Scopes.ReadUsers)));
});

app.MapGet("/users", GetUsers)
    .RequireAuthorization("RequireReadUsers");
```

## Hierarchical Roles

Role hierarchy:

```csharp
public class RoleHierarchy
{
    private static readonly Dictionary<string, string[]> Hierarchy = new()
    {
        [Roles.SuperAdmin] = new[] { Roles.Admin, Roles.User, Roles.Guest },
        [Roles.Admin] = new[] { Roles.User, Roles.Guest },
        [Roles.User] = new[] { Roles.Guest },
        [Roles.Guest] = Array.Empty<string>()
    };

    public static bool HasRole(string userRole, string requiredRole)
    {
        if (userRole == requiredRole)
            return true;

        if (Hierarchy.TryGetValue(userRole, out var inheritedRoles))
        {
            return inheritedRoles.Contains(requiredRole);
        }

        return false;
    }
}

// Custom requirement
public class HierarchicalRoleRequirement : IAuthorizationRequirement
{
    public HierarchicalRoleRequirement(string role)
    {
        Role = role;
    }

    public string Role { get; }
}

public class HierarchicalRoleHandler
    : AuthorizationHandler<HierarchicalRoleRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        HierarchicalRoleRequirement requirement)
    {
        var userRole = context.User.FindFirst(ClaimTypes.Role)?.Value;

        if (userRole != null && RoleHierarchy.HasRole(userRole, requirement.Role))
        {
            context.Succeed(requirement);
        }

        return Task.CompletedTask;
    }
}
```

## Audit Logging

Log authorization decisions:

```csharp
public class AuditAuthorizationHandler : IAuthorizationHandler
{
    private readonly ILogger<AuditAuthorizationHandler> _logger;

    public AuditAuthorizationHandler(ILogger<AuditAuthorizationHandler> logger)
    {
        _logger = logger;
    }

    public Task HandleAsync(AuthorizationHandlerContext context)
    {
        var userId = context.User.FindFirst(ClaimTypes.NameIdentifier)?.Value ?? "anonymous";
        var resource = context.Resource?.ToString() ?? "unknown";

        foreach (var requirement in context.Requirements)
        {
            var requirementType = requirement.GetType().Name;
            var succeeded = context.HasSucceeded;

            if (succeeded)
            {
                _logger.LogInformation(
                    "Authorization succeeded for user {UserId} on resource {Resource} with requirement {Requirement}",
                    userId,
                    resource,
                    requirementType);
            }
            else
            {
                _logger.LogWarning(
                    "Authorization failed for user {UserId} on resource {Resource} with requirement {Requirement}",
                    userId,
                    resource,
                    requirementType);
            }
        }

        return Task.CompletedTask;
    }
}

// Register
builder.Services.AddSingleton<IAuthorizationHandler, AuditAuthorizationHandler>();
```

## Testing Authorization

Unit test authorization:

```csharp
public class AuthorizationTests
{
    [Fact]
    public async Task ManageUsersHandler_AdminUser_Succeeds()
    {
        // Arrange
        var requirement = new ManageUsersRequirement();
        var handler = new ManageUsersAuthorizationHandler();

        var user = new ClaimsPrincipal(new ClaimsIdentity(new[]
        {
            new Claim(ClaimTypes.NameIdentifier, "user_123"),
            new Claim(ClaimTypes.Role, Roles.Admin)
        }));

        var context = new AuthorizationHandlerContext(
            new[] { requirement },
            user,
            null);

        // Act
        await handler.HandleAsync(context);

        // Assert
        Assert.True(context.HasSucceeded);
    }

    [Fact]
    public async Task ManageUsersHandler_RegularUser_Fails()
    {
        // Arrange
        var requirement = new ManageUsersRequirement();
        var handler = new ManageUsersAuthorizationHandler();

        var user = new ClaimsPrincipal(new ClaimsIdentity(new[]
        {
            new Claim(ClaimTypes.NameIdentifier, "user_123"),
            new Claim(ClaimTypes.Role, Roles.User)
        }));

        var context = new AuthorizationHandlerContext(
            new[] { requirement },
            user,
            null);

        // Act
        await handler.HandleAsync(context);

        // Assert
        Assert.False(context.HasSucceeded);
    }
}
```

## Guidelines

**Role-Based:**
- Keep roles simple and meaningful
- Use role hierarchy for complex permissions
- Assign roles at user creation
- Avoid role proliferation

**Policy-Based:**
- Create reusable policies
- Combine multiple requirements
- Clear policy names
- Document policy intent

**Resource-Based:**
- Check ownership at resource level
- Validate on every access
- Log authorization failures
- Return 403 Forbidden when unauthorized

**Permissions:**
- Fine-grained permission model
- Use permission claims in tokens
- Group related permissions
- Document all permissions

**Security:**
- Deny by default
- Validate on server, not client
- Log authorization events
- Regular permission audits

## Benefits

Secure. Explicit authorization checks.

Flexible. Multiple authorization strategies.

Granular. Resource-level permissions.

Auditable. Log all authorization decisions.

## Related

- [authentication.md](./authentication.md) - User authentication
- [secrets-management.md](./secrets-management.md) - Secure configuration
- [owasp-security.md](./owasp-security.md) - Security best practices
