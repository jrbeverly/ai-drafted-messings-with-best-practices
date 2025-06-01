# Resource-Based Authorization

Authorize actions against specific resource instances (ownership, collaboration, tenant isolation) at runtime.

## Why It Matters

- Roles and policies alone cannot answer "can this user edit THIS document?"
- Resource-level checks prevent horizontal privilege escalation between users
- Tenant isolation is mandatory for multi-tenant SaaS

## Key Recommendations

- **Operations**: Define as static `OperationAuthorizationRequirement` instances

```csharp
public static class Operations
{
    public static OperationAuthorizationRequirement Read = new() { Name = "Read" };
    public static OperationAuthorizationRequirement Update = new() { Name = "Update" };
    public static OperationAuthorizationRequirement Delete = new() { Name = "Delete" };
}
```

- **Authorization handler**: Implement `AuthorizationHandler<OperationAuthorizationRequirement, TResource>`

```csharp
// Check order: Admin -> Owner -> Collaborator -> Public
if (context.User.IsInRole("Admin")) { context.Succeed(requirement); return; }
if (resource.OwnerId == userId) { context.Succeed(requirement); return; }
if (resource.CollaboratorIds.Contains(userId) && requirement.Name is "Read" or "Update")
    context.Succeed(requirement);
```

- **Endpoint pattern**: Fetch resource, then authorize, then act

```csharp
var document = await documentService.GetByIdAsync(id);
if (document is null) return Results.NotFound();
var authResult = await authService.AuthorizeAsync(user, document, Operations.Read);
if (!authResult.Succeeded) return Results.Forbid();
return Results.Ok(document);
```

- **Shared resources**: Use member roles (Owner/Editor/Viewer) with pattern matching
- **Tenant isolation**: Compare `resource.TenantId` against user's `tenant_id` claim first; deny cross-tenant access unconditionally
- **Hierarchical resources**: Check parent folder permissions recursively when child has no direct grant
- **Reusable helper**: Create `AuthorizeAndExecuteAsync<T>` extension to reduce boilerplate

## Pitfalls to Avoid

- Returning 403 on missing resources (reveals existence -- return 404 instead)
- Checking authorization before fetching the resource (you need the resource data to decide)
- Forgetting tenant isolation check (cross-tenant access is a critical vulnerability)
- N+1 queries in handlers -- batch permission lookups where possible
- Not logging authorization failures for audit trails
- Exposing ownership details in error messages
